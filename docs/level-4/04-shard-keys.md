---
description: "Choose a MongoDB shard key by cardinality, frequency and monotonicity; indivisible chunks, unique indexes, refineCollectionShardKey and resharding."
---

# 04 · Choosing & Changing a Shard Key

The shard key decides three things at once: **where each document lives**, **which shards each query must ask**,
and **where each new write lands**. A good key spreads writes, keeps the common queries on one shard, and lets data
divide into as many pieces as you'll ever need. A bad key creates one overloaded shard, broadcasts every query, or
produces chunks too big to move — and before MongoDB 5.0 it was effectively permanent.

This lesson evaluates keys against the criteria, then runs every failure and remedy on the cluster from Lesson 03
(MongoDB 9.0.2, three shards, `shop.orders` with 200,000 orders from 20,000 customers).

## The three properties

**Cardinality** — how many distinct values the key can have. Each chunk holds a range of key values, and a chunk
containing a single value can't be split. A key with 8 possible values can never be spread across more than 8
chunks, however much data arrives.

**Frequency** — how evenly documents are distributed over those values. If 40% of orders come from one enterprise
customer, `customerId` alone puts 40% of the data in a single unsplittable value.

**Monotonicity** — whether new values always increase (timestamps, ObjectIds, auto-increment numbers). Range
sharding on a monotonic key sends every insert to the chunk ending at `MaxKey`, on one shard. Lesson 03 showed it:
10,000 new events, all 10,000 on `shB`.

And the property people forget: **query isolation**. The key should appear in the equality filters of the
application's most frequent queries, so they target one shard.

## Low cardinality: chunks that can't be divided

`lab.bycity` is the orders data sharded on `{city: 1}` — eight cities, each with about 21,500 orders — with the
lab's 1 MB chunk size:

```text
Bengaluru:21577 Chennai:21417 Delhi:21372 Hyderabad:21421 Jaipur:21331 Kochi:21468 Kolkata:21719 Pune:21550
```

```text
shC  MinKey     -> Chennai
shB  Chennai    -> Delhi
shB  Delhi      -> Hyderabad
shC  Hyderabad  -> Jaipur
shB  Jaipur     -> Kochi
shA  Kochi      -> Pune
shA  Pune       -> MaxKey
```

Each city's orders are about 2 MB — double the chunk size — yet asking MongoDB to split a one-city chunk fails:

```javascript
sh.splitFind("lab.bycity", {city: "Chennai"})
```

```text
split Chennai chunk: CannotSplit - Unable to find median in chunk because chunk is indivisible.
```

Every document in that chunk has `city: "Chennai"`; there is no boundary to cut at. At production scale these
become **jumbo chunks**: too large to migrate, so they stay wherever they are, and a popular city's shard grows
without limit. Adding shards beyond the number of distinct values achieves nothing.

## Comparing candidate keys for orders

The application's main queries are "orders for customer X" and "customer X's orders in a date range". Four
candidates:

| Key | Write distribution | "Orders for customer X" | "Customer X, last 30 days" | Divisibility |
|-----|--------------------|-------------------------|----------------------------|--------------|
| `{_id: 1}` (ObjectId) | all inserts to one shard (monotonic) | broadcast | broadcast | fine |
| `{city: 1}` | 8 hot spots | broadcast | broadcast | **8 chunks max** |
| `{customerId: "hashed"}` | even | one shard | one shard | fine, but no range targeting on customerId |
| `{customerId: 1, placedAt: 1}` | spread across customers | one shard | one shard | fine — a big customer splits by date |

The compound key wins for this workload: `customerId` gives distribution and isolation, and `placedAt` as a
suffix lets one very active customer's orders be split by time, solving the frequency problem.
`{customerId: "hashed"}` is a reasonable runner-up and the safer choice if customer ids were themselves
monotonic (assigned sequentially to new sign-ups who then order heavily).

## Changing your mind: `reshardCollection`

`shop.orders` was sharded on `{customerId: "hashed"}` in Lesson 03. Resharding to the compound key is one
command, run through `mongos` — but the first attempt failed:

```javascript
db.adminCommand({reshardCollection: "shop.orders", key: {customerId: 1, placedAt: 1}})
```

```text
MongoServerError: Recipient shard shB reached an unrecoverable error :: caused by :: can't shard collection
'shop.orders' with unique index on { customerId: 1, orderNo: 1 }, proposed shard key { customerId: 1, placedAt: 1 }
and collation {}. Uniqueness can't be maintained unless shard key is a prefix and has simple collation
```

A unique index created earlier (below) was valid for the old key but not the new one. After dropping it:

```text
reshard ok: 1 in 332184 ms
new key: {"customerId":1,"placedAt":1}
count: 200000
```

Five and a half minutes for 200,000 small documents is not the copy speed — it's a deliberate floor. While it ran,
`$currentOp` showed each shard both donating and cloning:

```text
shB cloning elapsed Long('183') remaining est Long('166') copied Long('69752')
shB donating-initial-data elapsed Long('183') ...
minimum duration ms: 300000
```

`reshardingMinimumOperationDurationMillis` is 300,000 on this version: resharding takes at least five minutes so
that the oplog the recipients must catch up on stays bounded. Afterwards the collection had 30 chunks per shard
and the queries were targeted as designed:

```text
find({customerId: 4242})                                SINGLE_SHARD  returned 10    ["shB:10"]
find({customerId: {$gte: 100, $lt: 150}})               SINGLE_SHARD  returned 493   ["shC:493"]
find({customerId: 4242, placedAt: {$gte: 2026-06-01}})  SINGLE_SHARD  returned 6     ["shB:6"]
find({placedAt: {$gte: 2026-09-01}})                    SHARD_MERGE   returned 20035 ["shA:59747","shB:69752","shC:70501"]
```

The customer **range** query that was a broadcast under the hashed key is now single-shard. The date-only query
is still a broadcast — and examined every document on every shard because no index starts with `placedAt`. No key
serves every query; choose for the frequent ones and index for the rest.

Resharding rewrites the entire collection: each recipient needs room for its new share, there's extra write load
throughout, and writes are blocked during a short critical section at the end — if it can't finish within `reshardingCriticalSectionTimeoutMillis` (5,000 ms on this version) the whole operation is aborted. Plan it for a
quiet period on a cluster with headroom.

## A cheaper change: refining the key

If the key is right but not divisible enough, `refineCollectionShardKey` **appends** fields, without moving data.
For `logs.events` on `{ts: 1}`:

```text
refine without index: InvalidOptions - Please create an index that starts with the proposed shard key before refining the collection's shard key.
refine ok: 1 | key now {"ts":1,"level":1}
refine changing prefix: InvalidOptions - refineCollectionShardKey shard key { level: 1, ts: 1 } does not extend the current shard key { ts: 1, level: 1 }
```

It needs a supporting index, and it can only extend the existing key as a suffix. Refining helps a key with
**frequency** problems (many documents per value) become splittable. It does nothing for monotonicity: `{ts, level}`
still sends every new event to the last chunk.

## Rules that come with a shard key

**Unique indexes must start with the shard key.** Each shard enforces uniqueness only on its own data, so a
cluster-wide unique constraint is only possible when equal values are guaranteed to live on the same shard:

```text
CannotCreateIndex: ... cannot create index with 'unique' or 'prepareUnique' option over { orderNo: 1 } with shard
key pattern { customerId: "hashed" } and collation {}
unique {customerId: 1, orderNo: 1}: "customerId_1_orderNo_1"
```

An order number unique *per customer* was allowed; globally unique wasn't. For globally unique values on a sharded
collection, use `_id` (unique per shard plus generated values like ObjectIds that don't collide), or a separate
small collection sharded on the unique field as a "claim" table.

**A document's shard key value can change**, and the document moves to the owning shard:

```text
before: 4242 on [ 'shC' ]
set customerId to 4247 (hashed 3529713213878061024) modified: 1 | now on [ 'shB' ]
```

The update had to be a retryable write or run in a transaction. Through PyMongo with `retryWrites=false`:

```text
20 None - Plan executor error during update :: caused by :: Must run update to shard key field in a
multi-statement transaction or with retryWrites: true.
inside a transaction: ok, now 4242
```

Under the hood it's a delete on one shard and an insert on another, which must be atomic. It works, but if your
design changes key values often, the key is probably wrong.

**Single-document operations without the shard key** (`updateOne({_id: ...})`) are accepted on recent versions —
`updateOne by _id only, modified: 1` — but `mongos` has to ask every shard to find the document. Include the shard
key in filters on hot paths.

## How It Actually Works

A shard key value maps to exactly one chunk because chunks are contiguous, non-overlapping ranges over the key's
sort order (or over the 64-bit hash for hashed keys) covering `MinKey` to `MaxKey`. A split can only place a
boundary *between* two distinct key values, which is why a range holding one value is indivisible, and why
appending a field (refining) creates new possible boundaries: `{city: "Chennai", customerId: 17}` and
`{city: "Chennai", customerId: 18}` are distinct values of `{city, customerId}`.

Resharding builds the new layout alongside the old. The coordinator (on the config server primary) chooses new
chunk boundaries from a sample of the data; each recipient shard creates a temporary collection, clones its share
from every donor at a fixed timestamp, then tails every donor's oplog to apply changes made since. When the
recipients are caught up and the minimum duration has passed, donors block writes briefly, recipients apply the
last changes, and the coordinator atomically swaps the temporary collections in and installs the new routing table.
The old collection's data is then dropped. That's why it needs roughly double the space while it runs.

## Common mistakes

- **Sharding on a timestamp or ObjectId** with range sharding for an insert-heavy collection.
- **Low-cardinality keys** (`status`, `country`, `type`) that cap the number of chunks and create jumbo chunks.
- **A key absent from the main queries**, turning the hottest path into a broadcast.
- **Hashing a key you need ranges on**, losing range targeting.
- **Forgetting unique indexes** when planning a reshard — or designing on globally unique secondary fields.
- **Resharding a cluster without headroom** in disk and write capacity.
- **Picking the key from the schema, not the workload.** Pull the real query mix (profiler, slow logs, `$queryStats`
  where available) before deciding.

## Exercise

1. For a ride-hailing `trips` collection (fields: `tripId`, `riderId`, `driverId`, `city`, `startedAt`, `status`),
   list the three most frequent queries you'd expect, then propose and justify a shard key. What goes wrong with
   `{city: 1, startedAt: 1}`?
2. Shard a copy of the orders on `{_id: 1}` (ObjectIds) and insert 50,000 new orders. Show where they land.
   Then reshard to `{_id: "hashed"}` and repeat.
3. Create a data set where one `customerId` has 30% of all orders. Shard on `{customerId: 1}` with a small chunk
   size and find the indivisible chunk. Fix it with `refineCollectionShardKey` and show it splitting.
4. Implement globally unique `orderNo` on a collection sharded by `customerId`, using a second collection
   sharded on `orderNo` and a transaction that inserts into both.
5. Run `reshardCollection` while a writer inserts continuously. Measure the writer's longest pause and look in
   the logs for the critical section.
