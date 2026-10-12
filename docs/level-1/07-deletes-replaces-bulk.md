---
description: "Delete and replace documents safely, and batch writes with insertMany and bulkWrite: ordered vs unordered errors, result counts, and deleteMany vs drop."
---

# 07 · Deletes, Replaces & Bulk Writes

This lesson finishes basic CRUD with the operations that remove or overwrite data, then covers
sending many writes at once — which is how imports, sync jobs and batch processes should talk to
MongoDB. The interesting part is not the happy path; it is what happens when **one write in a batch
fails**.

Everything runs in a database called `bulk`.

## Deleting

```javascript
db.t.deleteOne({ _id: 12345 })          // first match only
db.t.deleteMany({ batch: true })        // every match
```

```text
{ acknowledged: true, deletedCount: 0 }
{ acknowledged: true, deletedCount: 3 }
```

Like updates, deletes report a count rather than failing when nothing matched. If deleting nothing is
an error for your application (deleting an order that doesn't exist), check `deletedCount` yourself.

`findOneAndDelete` removes one document **and returns it**, atomically — useful for "take the next job
from the queue":

```javascript
db.t.findOneAndDelete({ batch: true }, { sort: { _id: -1 } })
```

```text
{ _id: 10, s: 'a', batch: true }
```

The `sort` decides *which* matching document is taken when several match.

### Before you run a delete

A delete filter is the most dangerous thing you will type. Two habits prevent most disasters:

1. **Run the filter as a `countDocuments()` or `find()` first** and check the number is what you
   expect.
2. **Never build a delete filter from user input without validating it isn't empty.** In most drivers
   `{}` is a perfectly valid filter: `deleteMany({})` removes everything, and a filter like
   `{ userId: req.query.id }` turns into `{ userId: undefined }` when the parameter is missing — which
   drivers may serialize in surprising ways depending on settings. Validate first.

## deleteMany({}) versus drop()

Both leave you with no documents. They are very different operations. With 200,000 small documents and
one secondary index in each of two collections:

```javascript
let t = Date.now(); d.a.deleteMany({});
print("deleteMany ms", Date.now() - t, "indexes left", d.a.getIndexes().length);
t = Date.now(); d.b.drop();
print("drop ms", Date.now() - t, "collections:", d.getCollectionNames().join(","));
```

```text
deleteMany ms 650 indexes left 2
drop ms 7 collections: customers,t,a,orders
```

`deleteMany({})` deletes documents one at a time, writes an oplog entry for each (on a replica set,
every secondary repeats the work), and leaves the collection, its indexes, options and validation rules
in place. `drop()` removes the collection's files in one metadata operation; indexes and settings go
with it. On a laptop the difference is 650 ms vs 7 ms; on 500 million documents it is hours versus
milliseconds — and hours of replication lag. To empty a large collection and keep its definition, it
is usually faster to drop and recreate indexes (or create a fresh collection and rename it).

## Replacing a document

`replaceOne` swaps the whole document (except `_id`) for a new one:

```javascript
db.t.replaceOne({ _id: 2 }, { s: "replaced" });
db.t.findOne({ _id: 2 })
```

```text
{ _id: 2, s: 'replaced' }
```

Every field not in the replacement is gone. `_id` is immutable — try to change it and the server
refuses:

```text
66 Plan executor error during update :: caused by :: After applying the update, the (immutable) field '_id' was found to have been altered to _id: 22
```

And a replacement containing operators is rejected client-side, the mirror image of the protection
`updateOne` gives you:

```text
MongoInvalidArgumentError Replacement document must not contain atomic operators
```

Use `replaceOne` when your application owns the whole document and writes it back as a unit (a
settings object, a cached API response). Use `updateOne` with operators when several writers may change
different fields — a replace would overwrite their changes with whatever your copy had.

To "change" an `_id`, insert a new document with the new `_id` and delete the old one, inside a
transaction if nothing may see the in-between state (Level 3 · 06).

## Batches: ordered by default

`insertMany` sends many documents in one request. With `_id`s 1 and 2 already present, insert 3, 1, 4:

```javascript
try { db.t.insertMany([{ _id: 3 }, { _id: 1 }, { _id: 4 }]) }
catch (e) { print(e.name, "inserted:", e.result.insertedCount,
                  "errors:", e.writeErrors.map(w => w.index + ":" + w.code).join(",")) }
print(db.t.find().toArray().map(x => x._id).join(","))
```

```text
MongoBulkWriteError inserted: 1 errors: 1:11000
1,2,3
```

The batch is **ordered** by default: the server processes it in sequence and **stops at the first
error**. Document 3 was inserted, document 1 failed with a duplicate key (index 1 in the batch, code
11000), and document 4 was never attempted. Nothing was rolled back — 3 stays.

## Unordered batches keep going

```javascript
try { db.t.insertMany([{ _id: 5 }, { _id: 2 }, { _id: 6 }, { _id: 1 }, { _id: 7 }], { ordered: false }) }
catch (e) { print(e.name, "inserted:", e.result.insertedCount,
                  "errors:", e.writeErrors.map(w => w.index + ":" + w.code).join(",")) }
```

```text
MongoBulkWriteError inserted: 3 errors: 1:11000,3:11000
1,2,3,5,6,7
```

With `ordered: false` the server attempts **every** write and reports every failure by its index in the
batch. That is what you want for imports and idempotent syncs ("insert these, skip the ones that already
exist"). Unordered batches can also be faster, because on a sharded cluster the router can send the
writes to different shards in parallel. Use ordered batches only when a later write depends on an
earlier one.

Either way, a batch is **not a transaction**. Writes that succeeded stay applied when later ones fail.

## bulkWrite: mixed operations in one round trip

`bulkWrite` batches different kinds of writes against one collection:

```javascript
const r = db.t.bulkWrite([
  { insertOne:  { document: { _id: 10, s: "a" } } },
  { updateOne:  { filter: { _id: 1 }, update: { $set: { s: "b" } } } },
  { updateMany: { filter: { _id: { $gte: 5 } }, update: { $set: { batch: true } } } },
  { replaceOne: { filter: { _id: 2 }, replacement: { s: "replaced" } } },
  { deleteOne:  { filter: { _id: 3 } } },
  { updateOne:  { filter: { _id: 99 }, update: { $set: { s: "new" } }, upsert: true } }
]);
```

```text
{
  acknowledged: true,
  insertedCount: 1,
  insertedIds: { '0': 10 },
  matchedCount: 6,
  modifiedCount: 6,
  deletedCount: 1,
  upsertedCount: 1,
  upsertedIds: { '5': 99 }
}
```

`matchedCount` is 6: the first `updateOne` matched 1, the `updateMany` matched 4 (5, 6, 7 **and** 10 —
the document inserted earlier in the same ordered batch), and the `replaceOne` matched 1. The upsert is
reported separately with its batch index (`'5'`).

### Across collections

Since MongoDB 8.0 the server also has a client-level `bulkWrite` command that mixes collections in one
request. `mongosh` exposes it on the connection:

```javascript
db.getMongo().bulkWrite([
  { name: "insertOne", namespace: "bulk.orders", document: { _id: 1, total: 20 } },
  { name: "updateOne", namespace: "bulk.customers", filter: { _id: "ana" },
    update: { $inc: { orders: 1 } }, upsert: true }
])
```

```text
{
  acknowledged: true,
  insertedCount: 1,
  matchedCount: 0,
  modifiedCount: 0,
  deletedCount: 0,
  upsertedCount: 1,
  insertResults: undefined,
  updateResults: undefined,
  deleteResults: undefined
}
```

It saves round trips; it is still not atomic across the two collections. Driver support for the
client-level form arrived over several releases, so check your driver's documentation before relying
on it.

## How It Actually Works

The driver turns `insertMany`, `bulkWrite` and friends into as few server commands as possible. It
groups **consecutive** operations of the same kind (in an ordered batch) or all operations of the same
kind (in an unordered one) into `insert`, `update` and `delete` commands, and splits them further so no
single command message exceeds the server's limits (`maxWriteBatchSize`, 100,000 operations, and the
48 MB message size). That is why a 1-million-document `insertMany` still works: the driver sends it as
several commands.

On the server, each write in a command is its own atomic single-document operation. For an ordered
command, the server stops at the first failure and returns what it did plus the error; for an unordered
one, it records the failure and continues. The driver then merges the per-command replies into one
result, translating the indexes back to positions in *your* list — so `writeErrors[i].index` always
refers to your original array.

A `drop` is a catalog operation: on a replica set it is replicated as one oplog entry, and the data files
are removed after the drop is majority-committed. A `deleteMany` is N separate document deletes, each
with its own oplog entry and index maintenance, which is why its cost scales with the data.

## Common mistakes

- **Running a delete without counting the filter first.**
- **Using `replaceOne` when other code updates the same document** — it silently reverts their fields.
- **Assuming a failed batch rolled back.** Ordered or not, successful writes stay.
- **Using ordered batches for imports**, so one duplicate stops the whole job half-way.
- **Emptying a huge collection with `deleteMany({})`** when a drop-and-recreate would do.
- **Reporting "deleted" when `deletedCount` is 0.**

## Exercise

1. Insert `_id`s 1–10. Then `insertMany` a batch of `_id`s 8–15 ordered, and again unordered. Predict the
   final contents and the error indexes before running each.
2. Write a "sync" script that receives a list of products and, with a single `bulkWrite`, upserts every
   product by `sku`, setting `updatedAt`, and deletes products whose `sku` is not in the list (use
   `deleteMany` with `$nin`). Print the result counts.
3. Time `deleteMany({})` against `drop()` on 500,000 documents on your machine. Then check
   `db.getSiblingDB("local").oplog.rs.countDocuments({ns: "bulk.<yourcollection>"})` after each and
   explain the difference.
4. Use `findOneAndDelete` with a sort to build a tiny job queue: insert five jobs with priorities, and
   take them off in priority order.
