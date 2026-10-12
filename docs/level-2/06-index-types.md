---
description: "MongoDB's specialised indexes in practice: multikey arrays, unique (and the null trap), partial, sparse, TTL expiry, wildcard and hidden indexes."
---

# 06 · Multikey, Unique, Partial, TTL, Wildcard & Text Indexes

Lesson 05 covered the ordinary B-tree index and how to order its fields. The same structure, with a few
options switched on, solves a surprising range of problems: enforcing uniqueness, indexing only the
documents that matter, expiring old data automatically, and indexing fields you don't know in advance.
Each option also has a behaviour that catches people out. This lesson runs into each one on purpose.

All examples use a scratch database, `itypes`.

## Multikey indexes: arrays

Index a field that holds arrays and MongoDB creates a **multikey** index automatically — one entry per
element:

```javascript
db.p.insertMany([{ _id: 1, tags: ["a", "b"], sizes: [1, 2] }, { _id: 2, tags: ["b"], sizes: [3] }]);
db.p.createIndex({ tags: 1 });
db.p.find({ tags: "b" }).explain("executionStats")
```

The plan reports `isMultiKey: true` and examined 2 keys for the 2 documents containing `"b"`. You don't
declare it; the index becomes multikey the first time an array is inserted, and stays that way.

The restriction: a compound index can include **at most one array field per document**.

```javascript
db.p.createIndex({ tags: 1, sizes: 1 })
```

```text
Index build failed: <uuid>: Collection itypes.p ( <uuid> ) :: caused by :: ... :: caused by :: cannot index parallel arrays [sizes] [tags]
```

An index on both would need an entry for every *combination* of elements — the cross product. If the
index exists first, the same rule rejects the write instead:

```javascript
db.p2.createIndex({ tags: 1, sizes: 1 });
db.p2.insertOne({ tags: ["a"], sizes: 5 });        // fine: only one array
db.p2.insertOne({ tags: ["a"], sizes: [5, 6] })    // rejected
```

```text
171 cannot index parallel arrays [sizes] [tags]
```

Compound indexes over two fields *inside the same array of documents* (`items.sku` and `items.qty`) are
fine — that's one array.

## Unique indexes and the null trap

```javascript
db.u.createIndex({ email: 1 }, { unique: true });
db.u.insertOne({ name: "no email 1" });
db.u.insertOne({ name: "no email 2" })
```

```text
unique+missing: 11000 E11000 duplicate key error collection: itypes.u index: email_1 dup key: { email: null }
```

A document **without** the field is indexed as `null`, so only one document may lack it. For optional
unique fields (an email that only some accounts have, an external id that arrives later), make the index
**partial**:

```javascript
db.u2.createIndex({ email: 1 },
  { unique: true, partialFilterExpression: { email: { $type: "string" } } });
db.u2.insertMany([{ name: "a" }, { name: "b" }, { email: "x@y.z" }]);   // all three accepted
db.u2.insertOne({ email: "x@y.z" })                                      // 11000
```

Uniqueness now applies only among documents that have a string email.

Two more facts about unique indexes. They're the **only** reliable way to prevent duplicates under
concurrency (Level 1 · 10). And you can't create one on a collection that already contains duplicates — find
them first with an aggregation (`$group` by the key, `$match` count > 1).

## Partial indexes: index only what you query

Most orders are closed; the application constantly queries **open** orders by total. Index only those:

```javascript
db.orders2.createIndex({ total: 1 }, { partialFilterExpression: { status: "open" } });
```

```text
partial, status open: IXSCAN  no status: COLLSCAN
```

The index is used only when the query **guarantees** its results are within the index — here, by including
`status: "open"`. A query on `total` alone can't use it, because the index doesn't contain closed orders
and using it would give wrong answers. In exchange, a partial index on 5% of a collection is about 5% of the
size and write cost of a full one. Partial indexes are the right tool for "active", "unprocessed",
"not deleted" subsets.

## Sparse indexes

The older way to skip documents: a `sparse` index omits documents that lack the field. It has a sharp edge:

```javascript
db.sp.insertMany([{ _id: 1, nick: "zed" }, { _id: 2 }, { _id: 3, nick: "amy" }]);
db.sp.createIndex({ nick: 1 }, { sparse: true });
db.sp.find().sort({ nick: 1 })                     // 2,3,1
db.sp.find().sort({ nick: 1 }).hint({ nick: 1 })   // 3,1  ← document 2 vanished
```

Forced to use the sparse index, the query silently loses the document without a `nick`. The planner
won't choose a sparse index on its own when that would drop results, but a hint will. Prefer partial
indexes with an explicit `$exists: true` or `$type` filter — they say what they mean.

## TTL indexes: data that deletes itself

A TTL index on a **date** field removes documents a fixed time after that date. Sessions that expire 5
seconds after `lastSeen`:

```javascript
db.sessions.createIndex({ lastSeen: 1 }, { expireAfterSeconds: 5 });
db.sessions.insertMany([{ s: 1, lastSeen: new Date(Date.now() - 60000) },
                        { s: 2, lastSeen: new Date() },
                        { s: 3, lastSeen: "not a date" }]);
```

```text
ttl start: 3 2026-10-11T17:43:12.600Z
2026-10-11T17:43:25.008Z 1,2,3
2026-10-11T17:44:23.745Z 3
```

Twelve seconds later, **nothing** had been deleted, even session 1, which expired a minute before it was
inserted. About a minute later, sessions 1 and 2 were gone. Session 3 will never be deleted: its
`lastSeen` is a string, and TTL only acts on BSON dates (or arrays of dates, using the earliest).

The delay is the **TTL monitor**, a background thread that wakes up every 60 seconds by default:

```javascript
db.adminCommand({ getParameter: 1, ttlMonitorSleepSecs: 1 }).ttlMonitorSleepSecs   // 60
```

So TTL is for housekeeping, not for exact expiry. If a session must be invalid the instant it expires, check
`lastSeen` in your query as well and let TTL clean up afterwards. Two useful variations:

- **Expire at a specific time:** set `expireAfterSeconds: 0` and store the expiry moment itself in the field
  (`expiresAt`). Each document then chooses its own lifetime.
- **Change the lifetime later** with `collMod` instead of rebuilding the index.

TTL deletes are ordinary deletes: they're replicated, they cost I/O, and a TTL index suddenly applied to
a hundred million old documents will generate a hundred million deletes. Roll out TTL on big collections
gradually, or delete the backlog in batches first.

## Wildcard indexes: fields you don't know in advance

For documents with user-defined or highly variable fields, a wildcard index indexes every path under a
prefix:

```javascript
db.w.insertMany([{ attrs: { color: "red", size: "M" } }, { attrs: { color: "blue", weight: 3 } },
                 { attrs: { voltage: 230 } }]);
db.w.createIndex({ "attrs.$**": 1 });
db.w.find({ "attrs.weight": 3 }).explain().queryPlanner.winningPlan.inputStage.keyPattern
```

```text
{"$_path":1,"attrs.weight":1}
```

The index stores the path as part of each key (`$_path`), so a query on any one field under `attrs` can use
it. This is an alternative to the attribute pattern (lesson 03) when you'd rather keep natural field names.
Limits: a wildcard index helps queries on **one** field at a time (it doesn't replace a compound index for
multi-field filters, though compound wildcard indexes with fixed prefix fields exist in recent versions), and
indexing *everything* in a document is rarely a good idea for write-heavy collections.

## Hidden indexes: test a removal safely

Before dropping an index you think is unused, **hide** it. The planner ignores it, but it's still maintained,
so un-hiding is instant:

```javascript
db.orders2.createIndex({ i: 1 });
db.orders2.hideIndex("i_1");    // plan for {i: 5}: COLLSCAN
db.orders2.unhideIndex("i_1");  // plan for {i: 5}: FETCH (via IXSCAN)
```

Hide it, watch latency and slow-query logs for a few days, then drop it. Re-creating a dropped index on a large
collection can take hours; un-hiding takes milliseconds.

## Text, geospatial, hashed and others

A few index types belong to later lessons:

- **Text indexes** (`{ description: "text" }`) for word search with stemming and stop words — Level 3 · 09.
- **2dsphere** for geographic queries — Level 3 · 08.
- **Hashed** indexes (`{ userId: "hashed" }`) store a hash of the value; they support equality only and
  exist mainly for **hashed sharding** — Level 4 · 04.
- **Collation** on an index (`{ collation: { locale: "en", strength: 2 } }`) for case-insensitive equality
  and sorting — Level 3 · 09.

## How It Actually Works

All of these are the same WiredTiger B-tree with different rules about **which keys are generated** for a
document:

- **Multikey**: the key generator walks the indexed path, and when it finds an array it emits one key per
  element (deduplicated). The index's catalog entry records which paths are multikey, which tells the planner
  where bounds can't be combined.
- **Unique**: before inserting a key, the index checks whether that key already exists pointing to a different
  document, and fails the write if so. Missing fields generate a `null` key, hence the trap.
- **Partial**: the key generator first evaluates `partialFilterExpression`; documents that fail it get no keys at
  all. The planner may use the index only if it can prove the query's filter implies that expression.
- **Sparse**: documents with none of the indexed fields generate no keys.
- **Wildcard**: the generator walks every path under the prefix and emits `(path, value)` pairs.
- **TTL**: an ordinary index plus a catalog flag. The TTL monitor periodically runs, for each TTL index,
  essentially `deleteMany({field: {$lt: now - expireAfterSeconds}})` using that index — which is why it's
  efficient, why non-date values are never matched, and why deletion is batched rather than instant.
- **Hidden**: a catalog flag the planner checks; the index is otherwise fully maintained.

## Common mistakes

- **A unique index on an optional field** without a partial filter — the second document without it fails.
- **Expecting TTL to delete exactly on time.** Filter on the expiry in queries too.
- **Dates stored as strings in a TTL field.** Those documents live forever.
- **Querying a partial index's field without the filter condition**, then wondering why it isn't used.
- **Hinting a sparse index** and silently losing documents.
- **Indexing two arrays in one compound index**, which fails at build or at write time.
- **Dropping an index without hiding it first.**

## Exercise

1. Create a `users` collection where `username` is required and unique, and `phone` is optional but unique when
   present. Prove both rules with inserts.
2. Insert 100,000 tasks where 2% have `status: "pending"`. Create a partial index for pending tasks ordered by
   `dueAt`. Compare its size with a full index on `{status: 1, dueAt: 1}` using `collStats`.
3. Build an `otpCodes` collection where each code expires at its own `expiresAt` (use `expireAfterSeconds: 0`).
   Insert codes expiring in 30, 90 and 150 seconds, and record when each one actually disappears.
4. Add a wildcard index on `attrs` to a product collection with 20 different attribute names. Find a query that
   can't use it efficiently, and explain why.
5. Hide one of the `idx.orders` indexes from lesson 05 and rerun the ESR query. What does the planner choose
   instead?
