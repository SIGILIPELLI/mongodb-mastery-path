---
description: "MongoDB aggregation basics on 30,000 orders: $match, $group and accumulators, $unwind, $sort, $limit, $project, $count and why stage order matters."
---

# 01 · Aggregation Pipeline Fundamentals

`find()` returns documents as they're stored. Everything else — totals, averages, counts per group,
reshaped documents, joins, reports — is the job of the **aggregation pipeline**. A pipeline is a list of
**stages**; documents flow through them in order, each stage transforming the stream it receives. If you
know SQL, `$match` is `WHERE`, `$group` is `GROUP BY`, `$sort` is `ORDER BY`, and `$project` is the
`SELECT` list — but the pipeline lets you chain them in any order, as many times as you like.

## The Level 3 dataset

Level 3 uses a seeded online-shop database: 2,000 customers, 200 products and 30,000 orders placed between
July 2025 and June 2026. Each order embeds its line items with the price paid. The generator, `shopgen.py`:

```python
"""Seeded sample data for Level 3: customers, products and orders with line items."""
import random
from datetime import datetime, timedelta, timezone
from decimal import Decimal
from bson import Decimal128
from pymongo import MongoClient

r = random.Random(2026)
client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0")
client.drop_database("shop")
db = client.shop

cities = ["Pune", "Mumbai", "Delhi", "Chennai", "Hyderabad", "Bengaluru"]
customers = [{"_id": f"C{i:04d}", "name": f"Customer {i}", "city": r.choice(cities),
              "joined": datetime(2024, 1, 1, tzinfo=timezone.utc) + timedelta(days=r.randint(0, 600))}
             for i in range(1, 2001)]
db.customers.insert_many(customers)

cats = {"books": (199, 1499), "electronics": (999, 59999), "kitchen": (299, 4999), "toys": (149, 2999)}
products = []
for i in range(1, 201):
    cat = r.choice(list(cats))
    lo, hi = cats[cat]
    products.append({"_id": f"P{i:03d}", "name": f"{cat.title()} item {i}", "category": cat,
                     "price": Decimal128(Decimal(r.randint(lo, hi)))})
db.products.insert_many(products)

orders = []
start = datetime(2025, 7, 1, tzinfo=timezone.utc)
for i in range(1, 30001):
    c = r.choice(customers)
    items = []
    for p in r.sample(products, r.choice([1, 1, 1, 2, 2, 3, 4])):
        items.append({"sku": p["_id"], "category": p["category"], "qty": r.choice([1, 1, 1, 2, 3]), "price": p["price"]})
    orders.append({"_id": i, "customerId": c["_id"], "city": c["city"],
                   "status": r.choices(["delivered", "shipped", "cancelled", "returned"], [80, 10, 7, 3])[0],
                   "placedAt": start + timedelta(minutes=r.randint(0, 60 * 24 * 365)), "items": items})
db.orders.insert_many(orders)
db.orders.create_index([("customerId", 1), ("placedAt", -1)])
db.orders.create_index([("placedAt", 1)])
```

```text
{'customers': 2000, 'products': 200, 'orders': 30000}
```

An order looks like this:

```text
{
  _id: 1, customerId: 'C0110', city: 'Hyderabad', status: 'shipped',
  placedAt: ISODate('2026-02-21T11:48:00.000Z'),
  items: [
    { sku: 'P169', category: 'toys', qty: 1, price: Decimal128('254') },
    { sku: 'P055', category: 'kitchen', qty: 1, price: Decimal128('1183') }
  ]
}
```

Prices are whole rupees stored as Decimal128. Outputs below are printed with `JSON.stringify`, which shows
decimals as `{"$numberDecimal": "..."}`.

## $group: one output document per group

```javascript
db.orders.aggregate([
  { $group: { _id: "$status", orders: { $sum: 1 } } },
  { $sort: { orders: -1 } }
])
```

```text
{"_id":"delivered","orders":24080}
{"_id":"shipped","orders":2931}
{"_id":"cancelled","orders":2081}
{"_id":"returned","orders":908}
```

`_id` is the **group key** — here the value of `status` (the `$` prefix means "the field", not the literal
string). Every other field is an **accumulator**: `$sum: 1` adds one per document. The common accumulators:

| Accumulator | Result |
|-------------|--------|
| `$sum`, `$avg`, `$min`, `$max` | arithmetic over the group |
| `$count` | number of documents (`{ $count: {} }`) |
| `$first`, `$last` | first/last value in the order documents arrive — sort first! |
| `$push`, `$addToSet` | array of all values / distinct values |
| `$top`, `$bottom`, `$topN`, `$bottomN` | best N by an explicit `sortBy`, no prior `$sort` needed |
| `$stdDevPop`, `$median`, `$percentile` | statistics |

Grouping and counting by one field is so common it has a shortcut, `$sortByCount`:

```javascript
db.orders.aggregate([{ $sortByCount: "$city" }])
```

```text
{"_id":"Pune","count":5242}
{"_id":"Delhi","count":5183}
...
```

`_id: null` groups the whole collection into one document:

```javascript
db.orders.aggregate([{ $group: { _id: null, orders: { $sum: 1 },
  first: { $min: "$placedAt" }, last: { $max: "$placedAt" } } }])
```

```text
{"_id":null,"orders":30000,"first":"2025-07-01T00:04:00.000Z","last":"2026-06-30T23:43:00.000Z"}
```

## $unwind: one document per array element

Revenue lives in the line items, which are an array. `$unwind` turns one order with three items into three
documents, each carrying one item (and all the order's other fields):

```javascript
db.orders.aggregate([
  { $match: { status: "delivered" } },
  { $unwind: "$items" },
  { $group: { _id: "$city",
              revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } },
              lines: { $sum: 1 } } },
  { $sort: { revenue: -1 } },
  { $limit: 3 }
])
```

```text
{"_id":"Chennai","revenue":{"$numberDecimal":"135647659"},"lines":8350}
{"_id":"Pune","revenue":{"$numberDecimal":"133519311"},"lines":8198}
{"_id":"Delhi","revenue":{"$numberDecimal":"129433588"},"lines":8330}
```

Read it top to bottom: keep delivered orders, split into line items, sum quantity × price per city, sort, keep
three. Note that `lines` counts **line items**, not orders — after `$unwind` every count is per element. Mixing
up the two is the most common aggregation bug. When you need both, count distinct order ids:

```javascript
db.orders.aggregate([
  { $match: { status: "delivered", placedAt: { $gte: new Date("2026-01-01"), $lt: new Date("2026-04-01") } } },
  { $unwind: "$items" },
  { $group: { _id: { $dateTrunc: { date: "$placedAt", unit: "month" } },
              revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } },
              orders: { $addToSet: "$_id" } } },
  { $project: { _id: 0, month: { $dateToString: { date: "$_id", format: "%Y-%m" } },
                revenue: 1, orders: { $size: "$orders" } } },
  { $sort: { month: 1 } }
])
```

```text
{"revenue":{"$numberDecimal":"66107008"},"month":"2026-01","orders":2018}
{"revenue":{"$numberDecimal":"55116092"},"month":"2026-02","orders":1832}
{"revenue":{"$numberDecimal":"67508922"},"month":"2026-03","orders":2065}
```

`$dateTrunc` rounds each date down to the start of its month (in UTC unless you pass a `timezone`), giving a
group key per month; `$project` renames and formats the output. The `$addToSet` of order ids works here because a
month has a couple of thousand orders; for very large groups it would build a huge array — lesson 02 shows the
leaner way (computing each order's total before unwinding anything).

## Best-per-group with $top

"The latest order in each city" — without `$top` you'd `$sort` by date and take `$first`. With it:

```javascript
db.orders.aggregate([
  { $group: { _id: "$city", latest: { $top: { sortBy: { placedAt: -1 }, output: { id: "$_id", at: "$placedAt" } } } } },
  { $sort: { _id: 1 } }, { $limit: 2 }
])
```

```text
{"_id":"Bengaluru","latest":{"id":8581,"at":"2026-06-30T22:08:00.000Z"}}
{"_id":"Chennai","latest":{"id":9799,"at":"2026-06-30T19:23:00.000Z"}}
```

## Stage order matters

Stages run in the order you write them. Each one sees only what the previous stage produced:

```javascript
db.orders.aggregate([
  { $group: { _id: "$city", n: { $sum: 1 } } },
  { $match: { status: "delivered" } }
])
```

This returns **nothing**. After `$group`, the documents are `{_id: "Pune", n: 5242}` — there is no `status`
field left to match. Filter before you group. As a rule, put `$match` (and `$limit`, when it's safe) as **early**
as possible: it reduces the work of every later stage, and a `$match` at the start of a pipeline can use indexes
exactly like `find()` (lesson 05).

Ties also need care: "top customers by order count" sorts on a count, and many customers share a count. Add a
unique tie-breaker so results are deterministic:

```javascript
db.orders.aggregate([
  { $match: { status: "delivered" } },
  { $group: { _id: "$customerId", orders: { $sum: 1 }, last: { $max: "$placedAt" } } },
  { $sort: { orders: -1, _id: 1 } }, { $limit: 3 }
])
```

```text
{"_id":"C0227","orders":25,"last":"2026-06-25T18:31:00.000Z"}
{"_id":"C1544","orders":25,"last":"2026-06-18T21:47:00.000Z"}
{"_id":"C0117","orders":24,"last":"2026-06-21T08:24:00.000Z"}
```

`$count` is a stage that outputs a single document with the number of documents it received:

```javascript
db.orders.aggregate([{ $match: { status: "returned" } }, { $count: "returned" }])   // {"returned":908}
```

## How It Actually Works

When `aggregate` arrives, the server parses the stages and runs an **optimisation pass** over them before
executing. It moves `$match` stages earlier when that can't change the result, merges adjacent `$match`es,
coalesces `$sort` + `$limit` into a top-k sort, and pushes as much of the beginning of the pipeline as possible
into the **query layer** — the same planner that serves `find()`. A leading `$match` (and `$sort`, and sometimes
`$project`/`$group`) therefore becomes an ordinary indexed query whose results stream into the rest of the
pipeline.

Execution is a chain of pull-based stages, like the plan stages in Level 2 · 07. Streaming stages (`$match`,
`$unwind`, `$project`, `$set`) handle one document at a time with little memory. **Blocking** stages (`$group`,
`$sort`, `$bucket`) must see all of their input before emitting anything; `$group` keeps one accumulator state per
group in a hash table, so its memory grows with the number of **distinct groups**, not the number of documents
— grouping 30,000 orders by six cities needs six entries. Blocking stages that exceed their memory limit (100 MB
per stage by default) spill to disk, which `allowDiskUseByDefault` permits since 6.0.

Results come back through a cursor, in batches, exactly like `find()`; the 16 MB limit applies to each **output
document**, not to the whole result set.

## Common mistakes

- **Counting after `$unwind`** and calling the result "orders".
- **`$match` after `$group`** on fields that no longer exist.
- **Using `$first`/`$last` without sorting first** — the order is undefined. Prefer `$top`/`$bottom`.
- **Sorting on non-unique values without a tie-breaker**, making pagination and tests flaky.
- **`$push`-ing every document of a large group** into one array and hitting the 16 MB output limit.
- **Forgetting the `$` prefix** — `{ _id: "status" }` groups everything under the literal string "status".

## Exercise

Using the `shop` database:

1. Count orders per status per city (a compound `_id`: `{city: "$city", status: "$status"}`), sorted by city
   then count.
2. Find the 5 best-selling SKUs by quantity among delivered orders in 2026.
3. Compute average order value (sum of line totals per order, then average across orders) for each city. You will
   need two `$group` stages — the first per order, the second per city.
4. For each customer, find the date of their first and last order and the number of distinct cities they ordered
   to (should be 1 with this data — why?).
5. Write the same "orders per status" report with `$group` + `$sort` and with `$sortByCount`, and compare their
   `explain()` output.
