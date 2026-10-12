---
description: "MongoDB update operators: $set, $inc, $push and $pull, positional $, $[] and arrayFilters, upserts and findOneAndUpdate."
---

# 06 · Update Operators, Upserts & Array Updates

In a relational database you often update a row by reading it, changing it in your program, and
writing it back. In MongoDB you usually describe the **change** instead — "add 1 to `views`", "append
this item", "set the quantity of the `pen` line to 5" — and the server applies it to the document
atomically. Learning the update operators is learning to say exactly that.

All examples run in a database called `upd`.

## Why describe the change instead of the result

A small Python program runs 2,000 "add one to qty" operations from 16 threads, twice: once by reading
the document and writing back `qty + 1`, once with `$inc`.

```python
from concurrent.futures import ThreadPoolExecutor
from pymongo import MongoClient

coll = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0").upd.stock
coll.delete_many({})
coll.insert_many([{"_id": "read-modify-write", "qty": 0}, {"_id": "inc", "qty": 0}])

def rmw(_):
    doc = coll.find_one({"_id": "read-modify-write"})
    coll.update_one({"_id": "read-modify-write"}, {"$set": {"qty": doc["qty"] + 1}})

def inc(_):
    coll.update_one({"_id": "inc"}, {"$inc": {"qty": 1}})

with ThreadPoolExecutor(16) as pool:
    list(pool.map(rmw, range(2000)))
    list(pool.map(inc, range(2000)))

for d in coll.find().sort("_id", 1):
    print(d)
```

```text
{'_id': 'inc', 'qty': 2000}
{'_id': 'read-modify-write', 'qty': 400}
```

Four out of five read-modify-write increments were **lost**: two threads read the same value, both
wrote value + 1, and one increment vanished. The exact number lost will differ on every run; that it
loses some will not. `$inc` was correct every time because the server applies it to the current
document under a lock. Every single-document update in MongoDB is atomic — use that.

## Field operators

The starting document:

```javascript
db.carts.insertOne({ _id: "c1", user: "ana",
  items: [{ sku: "pen", qty: 2, price: NumberDecimal("1.50") },
          { sku: "ink", qty: 1, price: NumberDecimal("4.00") }],
  tags: ["new"], views: 0 });
```

```javascript
db.carts.updateOne({ _id: "c1" },
  { $inc: { views: 1 }, $set: { status: "open" }, $currentDate: { updatedAt: true } })
```

```text
{ acknowledged: true, insertedId: null, matchedCount: 1, modifiedCount: 1, upsertedCount: 0 }
```

Run the `$set` part again and look at the counts:

```javascript
db.carts.updateOne({ _id: "c1" }, { $set: { status: "open" } })
```

```text
{ acknowledged: true, insertedId: null, matchedCount: 1, modifiedCount: 0, upsertedCount: 0 }
```

`matchedCount` is how many documents the filter found; `modifiedCount` is how many actually changed.
Setting a field to the value it already has is a no-op — nothing is written to disk or to the oplog.
Check `matchedCount` when "did my document exist?" matters, for example to return a 404.

| Operator | Does | Example |
|----------|------|---------|
| `$set` | set a field (creating it, and any missing parents) | `{$set: {"address.city": "Pune"}}` |
| `$unset` | remove a field | `{$unset: {views: ""}}` |
| `$inc` | add (negative to subtract) | `{$inc: {stock: -1}}` |
| `$mul` | multiply | `{$mul: {price: NumberDecimal("1.1")}}` |
| `$min` / `$max` | set only if the new value is lower / higher | `{$min: {low: 7}}` |
| `$rename` | rename a field | `{$rename: {status: "state"}}` |
| `$currentDate` | set to the server's current date | `{$currentDate: {updatedAt: true}}` |
| `$setOnInsert` | set only when an upsert inserts | see below |

`$min` and `$max` are handy for "lowest price seen" and "last activity" fields without reading first:

```javascript
db.prices.insertOne({ _id: 1, low: 10, high: 10 });
db.prices.updateOne({ _id: 1 }, { $min: { low: 7 }, $max: { high: 8 } });
db.prices.findOne()
```

```text
{ _id: 1, low: 7, high: 10 }
```

`low` became 7 because 7 < 10; `high` stayed 10 because 8 is not higher.

## Array operators

```javascript
db.carts.updateOne({ _id: "c1" }, {
  $push: { items: { sku: "pad", qty: 1, price: NumberDecimal("2.25") } },
  $addToSet: { tags: { $each: ["new", "gift"] } }
});
```

`$push` appends. `$addToSet` appends only values not already present, so `"new"` was skipped and
`tags` became `['new', 'gift']`. `$pull` removes every element matching a condition, and `$pop`
removes the first or last element.

### A capped "most recent" list

`$push` takes modifiers that keep an array bounded. Push five events, keeping only the three with the
highest `at`:

```javascript
db.feed.insertOne({ _id: 1, recent: [] });
for (const n of [5, 9, 2, 7, 1])
  db.feed.updateOne({ _id: 1 },
    { $push: { recent: { $each: [{ n, at: n }], $sort: { at: -1 }, $slice: 3 } } });
db.feed.findOne().recent.map(x => x.n)
```

```text
[ 9, 7, 5 ]
```

This pattern ("last N notifications", "top 3 scores") is one of the cleanest ways to embed a list
without letting it grow toward the 16 MB limit.

## Updating elements inside arrays

There are three ways to target array elements.

**Positional `$`** — the first element matched by the *query*:

```javascript
db.carts.updateOne({ _id: "c1", "items.sku": "pen" }, { $inc: { "items.$.qty": 3 } })
```

**All positional `$[]`** — every element:

```javascript
db.carts.updateOne({ _id: "c1" }, { $set: { "items.$[].checked": false } })
```

**Filtered positional `$[name]` with `arrayFilters`** — every element matching a condition you name:

```javascript
db.carts.updateOne({ _id: "c1" },
  { $set: { "items.$[cheap].discount": NumberDecimal("0.10") } },
  { arrayFilters: [{ "cheap.price": { $lt: NumberDecimal("3") } }] })
```

The cart after these three updates:

```text
{
  user: 'ana',
  items: [
    { sku: 'pen', qty: 5, price: Decimal128('1.50'), checked: false, discount: Decimal128('0.10') },
    { sku: 'ink', qty: 1, price: Decimal128('4.00'), checked: false },
    { sku: 'pad', qty: 1, price: Decimal128('2.25'), checked: false, discount: Decimal128('0.10') }
  ],
  tags: [ 'new', 'gift' ],
  views: 1,
  status: 'open',
  updatedAt: ISODate('2026-10-11T17:28:53.998Z')
}
```

The pen went from 2 to 5, every item got `checked: false`, and only the two items under 3.00 got a
discount. Positional `$` fails if the query didn't match an array element:

```javascript
db.carts.updateOne({ _id: "c1" }, { $set: { "items.$.qty": 1 } })
```

```text
2 Plan executor error during update :: caused by :: The positional operator did not find the match needed from the query.
```

Removing, unsetting and renaming in one update:

```javascript
db.carts.updateOne({ _id: "c1" },
  { $pull: { items: { sku: "ink" } }, $unset: { views: "" }, $rename: { status: "state" } })
```

The `ink` line is gone, `views` is gone and `status` is now `state`.

## Things an update can't do

Two operators can't touch the same path, or a path and its parent, in one update:

```javascript
db.k.insertOne({ _id: 1, a: { b: 1 } });
db.k.updateOne({ _id: 1 }, { $set: { a: 1, "a.b": 2 } })
```

```text
40 Updating the path 'a.b' would create a conflict at 'a'
```

And an update document without operators is rejected by the driver, which protects you from a classic
accident:

```javascript
db.carts.updateOne({ _id: "c1" }, { user: "bob" })
```

```text
MongoInvalidArgumentError Update document requires atomic operators
```

Older APIs (`update()`) treated a document without operators as a **replacement** and quietly wiped
every other field. If you really mean to replace, use `replaceOne` (lesson 07).

## Upserts

An upsert updates the matching document or inserts one if none matches. `$setOnInsert` sets fields
only on the insert:

```javascript
const hit = () => db.counters.updateOne({ _id: "page:/home" },
  { $inc: { hits: 1 }, $setOnInsert: { createdAt: new Date("2026-10-01") } }, { upsert: true });
hit(); hit();
db.counters.findOne()
```

The first call reported `upsertedCount: 1, insertedId: 'page:/home'`; the second
`matchedCount: 1, modifiedCount: 1`. Result:

```text
{ _id: 'page:/home', createdAt: ISODate('2026-10-01T00:00:00.000Z'), hits: 2 }
```

The inserted document is built from the **equality conditions in the filter** plus the update — so
`_id: "page:/home"` came from the filter. Upserts with a filter that is not on a unique field can
create duplicates when two clients upsert at the same moment; put a unique index on the filter field
(Level 2 · 06) and retry on a duplicate-key error.

## Getting the document back: findOneAndUpdate

`updateOne` tells you counts. When you need the document — say, the next number from a sequence —
use `findOneAndUpdate`:

```javascript
db.counters.findOneAndUpdate({ _id: "seq" }, { $inc: { v: 1 } }, { upsert: true, returnDocument: "after" })
db.counters.findOneAndUpdate({ _id: "seq" }, { $inc: { v: 1 } }, { returnDocument: "before" })
```

```text
{ _id: 'seq', v: 1 }
{ _id: 'seq', v: 1 }
```

The first call created the counter and returned it *after* the increment (1); the second returned it
*before* its increment (still 1; it is now 2). The read and the write happen atomically — no other
client can increment in between.

## Updates that use the document's own fields

An update can be an **aggregation pipeline** (in square brackets), which lets new values be computed
from existing fields:

```javascript
db.people.insertMany([{ _id: 1, first: "Ada", last: "Lovelace" }, { _id: 2, first: "Alan", last: "Turing" }]);
db.people.updateMany({}, [{ $set: { full: { $concat: ["$first", " ", "$last"] } } }]);
db.people.find()
```

```text
[
  { _id: 1, first: 'Ada', last: 'Lovelace', full: 'Ada Lovelace' },
  { _id: 2, first: 'Alan', last: 'Turing', full: 'Alan Turing' }
]
```

Inside a pipeline, `"$first"` means "the value of field `first`"; in an ordinary `$set` it would be the
literal string `"$first"`. Level 3 covers pipeline expressions.

## How It Actually Works

For each matching document the server takes a write intent on that document, applies every operator
to an in-memory copy, validates the result (size limit, `_id` unchanged, schema validation if
configured) and writes the new version through WiredTiger, which keeps the old version available to
readers that started earlier. The read of the current value and the write of the new one happen
inside one storage transaction, so no other writer can slip in between: two concurrent `$inc`s are
serialized, and if two writers race on the same document WiredTiger detects the **write conflict** and
the server transparently retries the losing operation. That is why `$inc` never lost an update in the
experiment while client-side read-modify-write did.

Atomicity is **per document**. `updateMany` matching 1,000 documents applies 1,000 separate atomic
updates; another client may see some updated and others not, and if the server crashes halfway, some
are done and some are not. When several documents must change together, you either design them into
one document (Level 2) or use a transaction (Level 3 · 06).

On a replica set the update is recorded in the oplog not as your operators but as its **effect** — for
`$inc` the resulting value, for `$push` with `$slice` the resulting array or a delta. That makes
replaying an oplog entry *idempotent*: applying it twice gives the same result, which replication
depends on.

## Common mistakes

- **Read-modify-write in application code** for counters and stock levels. Use `$inc` and conditional
  filters (`{_id, stock: {$gte: 1}}` with `$inc: {stock: -1}`).
- **Unbounded `$push`.** Add `$slice`, or move the list to its own collection.
- **Positional `$` without the array in the query.** It errors; use `arrayFilters`.
- **Upserting on a non-unique field** under concurrency, creating duplicates.
- **Ignoring `matchedCount`** and reporting success for an update that touched nothing.
- **Assuming `updateMany` is all-or-nothing.** It isn't.

## Exercise

Using a new `upd.inventory` collection:

1. Insert `{_id: "widget", stock: 3}`. Write a "reserve one" update that decrements stock **only if**
   stock is at least 1. Run it four times and inspect `matchedCount` each time.
2. Store a product with a `priceHistory` array. Write one update that pushes a new `{price, at}` entry
   and keeps only the 5 most recent.
3. Given an order with line items, use `arrayFilters` to set `backordered: true` on every line whose
   `qty` exceeds 10.
4. Implement an "upsert a daily stats document" pattern: one document per `(page, day)` with `hits`
   incremented and `firstSeen` set only on insert.
5. Re-run the lost-update experiment with 4 threads and then with 32 threads. Record how many updates
   were lost each time, then make the read-modify-write version safe without using `$inc` (hint: put
   the value you read into the filter and retry when `modifiedCount` is 0).
