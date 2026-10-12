---
description: "Join collections with $lookup (equality and pipeline forms), why the foreign-field index matters, hierarchies with $graphLookup, and $unionWith."
---

# 03 · Joins & Unions: $lookup, $graphLookup, $unionWith

Good MongoDB schemas need fewer joins than relational ones (Level 2 · 01), but "fewer" isn't "none":
reports combine orders with customers, admin screens show a product with its sales, an org chart needs a
manager chain. The pipeline has three stages for combining data: `$lookup` joins, `$graphLookup` follows
recursive references, and `$unionWith` concatenates collections. This lesson shows each one and measures the
one thing that decides whether a join is fast or a disaster.

Examples use the `shop` database from lesson 01.

## $lookup by equality

An order with its customer:

```javascript
db.orders.aggregate([
  { $match: { _id: 1 } },
  { $lookup: { from: "customers", localField: "customerId", foreignField: "_id", as: "customer" } },
  { $set: { customer: { $first: "$customer" } } },
  { $project: { customerId: 1, "customer.name": 1, "customer.joined": 1 } }
])
```

```text
{"_id":1,"customerId":"C0110","customer":{"name":"Customer 110","joined":"2025-02-12T00:00:00.000Z"}}
```

`$lookup` always produces an **array** (`as`), because in general several foreign documents can match. For a
to-one relationship, unwrap it with `$first` (or `$unwind`, which also drops orders with no customer unless you
pass `preserveNullAndEmptyArrays: true`). An unmatched lookup gives an empty array, not an error — it's a left
outer join.

When `localField` is an array, `$lookup` matches any element, and when `foreignField` is an array field in the
foreign documents, it matches if the array contains the value — the usual array semantics from Level 1 · 05.

## $lookup with a pipeline

The equality form returns *every* matching document in full. To filter, sort, limit or reshape the joined
documents, give `$lookup` a pipeline. A customer's two most recent delivered orders:

```javascript
db.customers.aggregate([
  { $match: { _id: { $in: ["C0001", "C0002"] } } },
  { $lookup: { from: "orders", let: { cid: "$_id" }, pipeline: [
      { $match: { $expr: { $eq: ["$customerId", "$$cid"] }, status: "delivered" } },
      { $sort: { placedAt: -1 } }, { $limit: 2 }, { $project: { _id: 1, placedAt: 1 } } ],
    as: "recent" } },
  { $project: { name: 1, recent: 1 } }
])
```

```text
{"_id":"C0001","name":"Customer 1","recent":[{"_id":9130,"placedAt":"2026-06-14T09:59:00.000Z"},{"_id":19071,"placedAt":"2026-04-30T03:48:00.000Z"}]}
{"_id":"C0002","name":"Customer 2","recent":[{"_id":22622,"placedAt":"2026-06-28T10:44:00.000Z"},{"_id":8738,"placedAt":"2026-05-22T18:15:00.000Z"}]}
```

`let` binds fields of the outer document to variables (`$$cid`), and the inner `$match` must use `$expr` to
compare a field with a variable. Since MongoDB 5.0 you can combine both forms — `localField`/`foreignField` for
the join condition plus a `pipeline` for the rest — which is shorter and lets the server use the equality for an
index lookup directly:

```javascript
db.customers.aggregate([
  { $match: { _id: "C0001" } },
  { $lookup: { from: "orders", localField: "_id", foreignField: "customerId", pipeline: [{ $count: "n" }], as: "stats" } }
])
```

```text
{"_id":"C0001","name":"Customer 1","city":"Pune","joined":"2024-11-23T00:00:00.000Z","stats":[{"n":12}]}
```

## The index on the foreign field is everything

For each input document, `$lookup` has to find the matching foreign documents. Join 10 products to the orders
that contain them (`foreignField: "items.sku"`), first with no index on `items.sku`:

```javascript
const pipe = [{ $match: { category: "books" } }, { $limit: 10 },
  { $lookup: { from: "orders", localField: "_id", foreignField: "items.sku", as: "o" } }];
db.products.explain("executionStats").aggregate(pipe)
```

```text
2 EQ_LOOKUP(NestedLoopJoin)
ms 85 docsExamined 300025 keysExamined 0
```

Then after `db.orders.createIndex({"items.sku": 1})`:

```text
2 EQ_LOOKUP(IndexedLoopJoin)
ms 6 docsExamined 2967 keysExamined 2942
```

Without the index, the join strategy was a **nested loop**: for each of the 10 products, scan all 30,000 orders
— 300,000 documents examined (plus the 25 products read). With the index, an **indexed loop join** looks up each
SKU in the index and fetches only the orders that contain it. The same join, two orders of magnitude apart, and
the gap grows linearly with the foreign collection: with 3 million orders the unindexed version examines 30
million documents for 10 products.

(The `2` is `explainVersion` 2: the server ran this `$lookup` in its slot-based execution engine, where it appears
as an `EQ_LOOKUP` stage with a `strategy`. Lookups that can't be pushed down appear as a `$lookup` stage in a
`stages` array instead, with `indexesUsed` and `collectionScans` fields. Either way, the question is the same:
did it use an index on the foreign field?)

Before writing any `$lookup`, check that `foreignField` (or the field your inner `$match` compares) is indexed.

## Anti-joins and $count's empty output

"Products that have never been ordered" is a lookup followed by a filter on an empty array:

```javascript
db.products.aggregate([
  { $lookup: { from: "orders", localField: "_id", foreignField: "items.sku", as: "o",
               pipeline: [{ $limit: 1 }, { $project: { _id: 1 } }] } },
  { $match: { o: { $size: 0 } } },
  { $count: "neverOrdered" }
])
```

This printed **nothing at all** — not `{neverOrdered: 0}`. Every product had at least one order, so no document
reached `$count`, and `$count` emits nothing for empty input. Application code reading `result[0].neverOrdered`
crashes. Handle the empty case (`result[0]?.neverOrdered ?? 0`). Note the inner `$limit: 1`: to test existence
you need one matching document, not all of them.

## $graphLookup: recursive references

An org chart where each person stores who they report to (Level 2 · 02's parent-reference tree):

```javascript
db.staff.insertMany([
  { _id: "asha", title: "CEO", reportsTo: null },
  { _id: "bala", title: "CTO", reportsTo: "asha" }, { _id: "chen", title: "CFO", reportsTo: "asha" },
  { _id: "dev", title: "Eng Manager", reportsTo: "bala" }, { _id: "esha", title: "Engineer", reportsTo: "dev" },
  { _id: "farid", title: "Engineer", reportsTo: "dev" }, { _id: "gita", title: "Accountant", reportsTo: "chen" }]);
db.staff.createIndex({ reportsTo: 1 });
```

**Up the tree** — Esha's management chain:

```javascript
db.staff.aggregate([
  { $match: { _id: "esha" } },
  { $graphLookup: { from: "staff", startWith: "$reportsTo", connectFromField: "reportsTo",
                    connectToField: "_id", as: "chain", depthField: "level" } },
  { $project: { chain: { $sortArray: { input: { $map: { input: "$chain", as: "c",
      in: { who: "$$c._id", level: { $toInt: "$$c.level" } } } }, sortBy: { level: 1 } } } } }
])
```

```text
{"_id":"esha","chain":[{"who":"dev","level":0},{"who":"bala","level":1},{"who":"asha","level":2}]}
```

**Down the tree** — everyone under Bala, at any depth (swap the connect fields):

```javascript
{ $graphLookup: { from: "staff", startWith: "$_id", connectFromField: "_id",
                  connectToField: "reportsTo", as: "team", depthField: "d" } }
```

```text
{"_id":"bala","team":["dev","esha","farid"],"size":3}
```

With `maxDepth: 0` only direct reports are returned (`["dev"]`). `$graphLookup` does a breadth-first search,
issuing one query per level against `connectToField` — so index that field. Its result array is **unordered**
(hence the `$sortArray` above), the depth field is a 64-bit integer, and the whole set of visited documents must
fit in 100 MB of memory. For large graphs (social networks, deep dependency trees), a dedicated graph database
or a precomputed ancestors array (Level 2 · 02) is a better fit.

## $unionWith: concatenating collections

Many systems move old data into archive collections. `$unionWith` reads another collection (optionally through
its own pipeline) and appends its documents to the stream — SQL's `UNION ALL`:

```javascript
db.orders.aggregate([
  { $match: { customerId: "C0001" } }, { $project: { _id: 1, placedAt: 1, src: "live" } },
  { $unionWith: { coll: "orders_2024", pipeline: [
      { $match: { customerId: "C0001" } }, { $project: { _id: 1, placedAt: 1, src: "archive" } } ] } },
  { $sort: { placedAt: 1 } }, { $limit: 4 }
])
```

```text
{"_id":"A2","placedAt":"2024-11-02T00:00:00.000Z","src":"archive"}
{"_id":"A1","placedAt":"2024-12-30T00:00:00.000Z","src":"archive"}
{"_id":9690,"placedAt":"2025-07-01T20:13:00.000Z","src":"live"}
{"_id":20645,"placedAt":"2025-07-02T06:11:00.000Z","src":"live"}
```

Put the `$match` inside the union's pipeline, not after the union, so each collection filters with its own indexes
before the documents are combined.

## How It Actually Works

`$lookup` is executed as a join between the stream of input documents (the outer side) and the foreign collection
(the inner side). For each outer document — or, in the slot-based engine, for each batch — the server needs the
inner documents whose `foreignField` equals the outer `localField`. It picks a strategy when the pipeline is planned:

- **IndexedLoopJoin** when a suitable index on the foreign field exists: one index lookup per outer value.
- **HashJoin** when there is no index but the foreign collection is small enough: read the foreign collection
  once, build a hash table keyed on the foreign field, then probe it per outer document.
- **NestedLoopJoin** otherwise: scan the foreign collection once per outer document.

The pipeline form with `let`/`$expr` and no `localField` is more general and historically ran the inner pipeline
as a separate sub-query per outer document; with `localField`/`foreignField` present, the server can use the
equality to choose an indexed strategy and then apply the rest of the inner pipeline to the matches. That's why the
concise form is usually the faster one.

Join results are embedded in each output document, so each joined document — the input plus its `as` array — must
stay under 16 MB. A product joined to every order it ever appeared in is exactly how pipelines hit that limit; use
an inner `$limit`, `$count` or `$group` to keep the joined part small.

## Common mistakes

- **No index on the foreign field.** The join becomes a scan per input document.
- **Joining before filtering.** `$match` and `$limit` the outer side first, so fewer lookups run.
- **Pulling whole foreign documents** when you need a count or two fields — use a pipeline with `$project`/`$count`.
- **Assuming `$lookup` or `$graphLookup` return ordered arrays.**
- **Reading `$count` output without handling the empty case.**
- **Using `$lookup` on every request** of a hot path where a little denormalisation would remove it.

## Exercise

Using the `shop` database:

1. Produce a report of the 10 customers with the highest delivered revenue, including their name and city from
   `customers` (group first, then look up only 10 customers).
2. For each product category, list the 3 products with the most units sold, joined with their names. Check with
   `explain` that the lookup uses an index.
3. Find customers who joined more than 180 days before their first order (`$lookup` with a `$group`ed inner
   pipeline, or group orders first and join customers).
4. Add 20 more people to the `staff` collection, 5 levels deep. Write a pipeline that returns each manager with the
   **total** number of people under them.
5. Create an `orders_archive` collection with orders from 2025 moved out of `orders` (copy with `$merge` from
   lesson 05, then delete). Write a "customer order history" pipeline with `$unionWith` and confirm both sides use
   their `customerId` indexes.
