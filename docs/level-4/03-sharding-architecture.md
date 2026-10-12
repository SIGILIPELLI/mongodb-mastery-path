---
description: "Build a local MongoDB sharded cluster (mongos, config servers, three shards), read its routing table, compare targeted and broadcast queries."
---

# 03 · Sharding Architecture

A replica set copies all the data to every member, so it scales reads (with caveats) and survives failures, but
every member still has to hold — and write — everything. When one machine can no longer keep up with the data
size or the write rate, MongoDB **shards**: it splits a collection by a *shard key* into ranges and spreads the
ranges over several replica sets. Applications keep talking to what looks like one database.

This lesson builds a complete sharded cluster on one laptop, then looks at how it routes every query, what the
"split pipeline" of an aggregation is, and how the balancer moves data around. The next lesson is about the
decision that determines whether any of this works well: choosing the shard key.

Versions: MongoDB 9.0.2. Sharding behaviour changed substantially across 6.0, 7.0 and 8.0 (how chunks are split,
how the balancer measures imbalance, how many chunks a new collection starts with); what follows is what this
version did, and older material you find online may describe the previous rules.

## The parts

| Component | What it does | In this lab |
|-----------|--------------|-------------|
| **Shards** | Each shard is a replica set holding a subset of the data. | `shA`, `shB`, `shC` (single-node replica sets) |
| **Config servers** | A replica set storing the cluster's metadata: which shards exist, which collections are sharded, which ranges live where. | `cfg` (one node) |
| **`mongos`** | Stateless router. Applications connect to it; it caches the routing table and forwards each operation to the right shard(s). | port 27400 |

In production every shard and the config server set have three members each, and you run several `mongos`
(often one per application host). For learning, single-node replica sets behave the same way for everything in
this lesson.

## Building it

```bash
O="--bind_ip 127.0.0.1 --wiredTigerCacheSizeGB 0.25 --oplogSize 64"
mongod --configsvr --replSet cfg --port 27401 --dbpath cfg $O --logpath cfg.log &
mongod --shardsvr  --replSet shA --port 27411 --dbpath a   $O --logpath a.log &
mongod --shardsvr  --replSet shB --port 27412 --dbpath b   $O --logpath b.log &
mongod --shardsvr  --replSet shC --port 27413 --dbpath c   $O --logpath c.log &
```

(In zsh, write `${=O}` instead of `$O` — zsh doesn't split variables into words, and `mongod` will refuse to start
with one giant argument. That cost a failed first attempt here.)

Initiate each replica set — the config server's with `configsvr: true` — then start the router pointing at the
config server set and register the shards through it:

```javascript
// on 27401
rs.initiate({_id: "cfg", configsvr: true, members: [{_id: 0, host: "127.0.0.1:27401"}]})
// on 27411, 27412, 27413
rs.initiate({_id: "shA", members: [{_id: 0, host: "127.0.0.1:27411"}]})   // and shB, shC
```

```bash
mongos --configdb cfg/127.0.0.1:27401 --port 27400 --bind_ip 127.0.0.1 --logpath mongos.log &
```

```javascript
// on 27400 (mongos)
sh.addShard("shA/127.0.0.1:27411"); sh.addShard("shB/127.0.0.1:27412"); sh.addShard("shC/127.0.0.1:27413")
db.adminCommand({listShards: 1}).shards.forEach(s => print(s._id, s.host))
```

```text
shA shA/127.0.0.1:27411
shB shB/127.0.0.1:27412
shC shC/127.0.0.1:27413
```

From here on, everything goes through `mongos`. Connecting directly to a shard to read or write application data
bypasses routing and is how people end up seeing duplicate or missing documents.

## Sharding a collection

Collections are not sharded by default — each unsharded collection lives entirely on its database's **primary
shard**. Sharding one is a single command naming the key:

```javascript
sh.shardCollection("shop.orders", {customerId: "hashed"})
```

The load script then inserted 200,000 orders (`customerId` 1–20,000, a city, a date, a total) in batches of
5,000:

```text
inserted 200000 in 2942 ms
```

```javascript
db.getSiblingDB("shop").orders.getShardDistribution()
```

```text
Shard shC   docs: 68352, chunks: 1
Shard shA   docs: 67150, chunks: 1
Shard shB   docs: 64498, chunks: 1
Totals      docs: 200000, chunks: 3
```

A **hashed** key spreads documents by a hash of the value, so the collection was created with one chunk per
shard and the inserts landed evenly from the first document.

## The routing table

The config database records each **chunk** — a contiguous range of shard-key values — and its owner:

```javascript
cfg.chunks.find({uuid: coll.uuid}, {_id: 0, shard: 1, min: 1, max: 1, lastmod: 1}).sort({min: 1})
```

```text
{ min: { customerId: MinKey() },                  max: { customerId: Long('-3074457345618258602') }, shard: 'shC', lastmod: Timestamp({ t: 1, i: 0 }) }
{ min: { customerId: Long('-3074457345618258602') }, max: { customerId: Long('3074457345618258602') }, shard: 'shA', lastmod: Timestamp({ t: 1, i: 1 }) }
{ min: { customerId: Long('3074457345618258602') },  max: { customerId: MaxKey() },                 shard: 'shB', lastmod: Timestamp({ t: 1, i: 2 }) }
```

For a hashed key the ranges are over the 64-bit hash, cut into thirds. `MinKey` and `MaxKey` are special BSON
values smaller and larger than everything, so the ranges cover every possible value. To route customer 4242,
`mongos` hashes it:

```javascript
convertShardKeyToHashed(4242)
```

```text
Long('-6212294558869892362')
```

That is below −3,074,457,345,618,258,602, so customer 4242 lives on `shC`. `lastmod` is the chunk's version;
whenever ownership changes it increases, which is how routers and shards detect that a cached routing table is
out of date.

## Targeted and broadcast operations

Every query falls into one of two classes, and explain on `mongos` shows which:

```javascript
function ex(label, f) {
  const e = f(), qp = e.queryPlanner.winningPlan;
  const shards = (e.executionStats.executionStages.shards || []).map(s => s.shardName + ":" + s.totalDocsExamined);
  print(label, qp.stage, "returned", e.executionStats.nReturned, "examined per shard", JSON.stringify(shards));
}
```

```text
find({customerId: 4242})                            SINGLE_SHARD      returned 11   examined per shard ["shC:11"]
find({customerId: {$in: [1, 2, 3]}})                SHARD_MERGE       returned 28   examined per shard ["shB:10","shA:18"]
find({customerId: {$lt: 50}})                       SHARD_MERGE       returned 512  examined per shard ["shC:109","shA:173","shB:230"]
find({city: 'Pune'}).sort({placedAt: -1}).limit(5)  SHARD_MERGE_SORT  returned 5    examined per shard ["shB:5","shA:5","shC:5"]
```

- An **equality on the shard key** goes to exactly one shard (`SINGLE_SHARD`) — the only shard that can hold
  matching documents. This is the query shape a sharded cluster scales best.
- An **`$in`** goes only to the shards owning those values — here two of the three.
- A **range on a hashed key** can't be targeted: consecutive customer ids hash to scattered places, so every shard
  is asked. A range on a *ranged* key would be targeted (next lesson).
- A query **without the shard key** is broadcast — "scatter-gather". With `sort` + `limit`, each shard returns its
  own top 5 and `mongos` merge-sorts them (`SHARD_MERGE_SORT`). The `{city: 1, placedAt: -1}` index existed on
  every shard, so each one examined only 5 documents.

Broadcast queries aren't wrong. They're fine for occasional reports; the problem is when the busiest query in the
application is one, because then every shard does work for every request and adding shards adds no capacity for it.

Aggregations are split in two:

```javascript
const agg = shop.orders.explain().aggregate([{$match: {city: "Pune"}}, {$group: {_id: "$items", revenue: {$sum: "$total"}}}])
```

```text
aggregate split -> shardsPart: ["$match","$group"] mergerPart: ["$mergeCursors","$group"] merge on: router
```

Each shard runs `$match` and a **partial** `$group`; the router merges the partial groups with a second `$group`.
Stages that need all the data at once (a `$sort` of everything, a `$group` on high-cardinality keys, some
`$lookup`s) move work to the merging side, which may be `mongos` or a chosen shard.

## The balancer, observed

To see data move, the next collection is sharded by **range** on a timestamp — a monotonically increasing key —
with the chunk size lowered to 1 MB (the default is 128 MB; never do this in production):

```javascript
cfg.settings.updateOne({_id: "chunksize"}, {$set: {value: 1}}, {upsert: true})
sh.shardCollection("logs.events", {ts: 1})
// insert 200,000 events with increasing ts
```

```text
primary shard of logs: shB
after inserts      chunks ["shB:1"]                 | docs ["shB=200000"]
after 30s          chunks ["shA:7","shB:1","shC:7"] | docs ["shA=61159","shB=200000","shC=61159"]
after 60s          chunks ["shA:7","shB:1","shC:7"] | docs ["shA=61159","shB=200000","shC=61159"]
```

A range-sharded empty collection starts as one chunk on the primary shard, so all 200,000 inserts went to `shB`.
Within 30 seconds the balancer had moved fourteen ranges to `shA` and `shC` — and then **stopped**, with the
cluster still unbalanced by these counts. Two separate things are going on:

**1. The donor still holds the moved documents.** Those per-shard counts come from `$collStats`, which counts what
is physically stored. After a migration commits, the documents left behind on the donor are **orphans**,
removed later by a background range deleter:

```text
countDocuments via mongos: 200000 | estimatedDocumentCount via mongos: 322318
shB local count: 200000
pending range deletions on shB: 14
orphanCleanupDelaySecs: 3600
```

`countDocuments` through `mongos` is correct, because each shard filters out documents it doesn't own.
`estimatedDocumentCount` reads collection metadata and **counts orphans**, overstating by 122,318. On this version
the range deleter waits an hour (`orphanCleanupDelaySecs`) before deleting, so queries still running against the
old owner can finish.

**2. The balancer balances data size, not chunk counts.** In recent versions it compares each shard's amount of
data for a collection and only migrates when the difference exceeds a threshold derived from the chunk size;
chunks are split as part of migrating. With `shA` and `shC` each holding about 61,000 documents and `shB` owning
the rest, the remaining difference was under the threshold.

New writes show the real problem with this key:

```text
shA returned 0
shC returned 0
shB returned 10000
```

Every new event has a larger `ts` than anything before it, so all 10,000 fell into the last range — `ts` up to
`MaxKey` — owned by `shB`. With a monotonically increasing key, **one shard takes all inserts**, and the balancer
spends its life moving yesterday's data away from it. That is the central lesson of shard-key choice.

## How It Actually Works

`mongos` holds a cached copy of each sharded collection's routing table, loaded from the config servers. Every
operation it sends carries the **shard version** it believes is current. The receiving shard compares that with
the version it knows; if the router is behind — because a migration just changed ownership — the shard rejects
the operation with a stale-config error, the router refreshes its table and retries. That's why routers can be
stateless and why there's no global lock on routing: correctness is checked at the shard on every request.

A migration copies a range from donor to recipient while the donor keeps serving reads and writes, streams the
changes made meanwhile, then enters a brief **critical section** where writes to that range block, commits the
new ownership to the config servers (bumping the version), and leaves the critical section. Queries arriving with
the old version are bounced and retried. The donor's copy then becomes orphaned and is scheduled for deletion.
The balancer itself runs on the config server primary, in rounds, choosing a donor/recipient pair per collection
and issuing these moves.

## Common mistakes

- **Sharding too early.** A sharded cluster is several replica sets plus config servers plus routers to operate.
  Vertical scaling, indexes and schema fixes come first.
- **Connecting applications to shards directly** instead of to `mongos`.
- **Trusting `estimatedDocumentCount` or per-shard stats** on a cluster with recent migrations.
- **Making the busiest query a broadcast**, then adding shards that don't help it.
- **Monotonically increasing range keys** (timestamps, ObjectIds, sequence numbers) for insert-heavy collections.
- **Tiny chunk sizes** copied from tutorials into production, causing endless migrations.
- **Ignoring the primary shard**: unsharded collections in a database all live there and can make it the hottest
  node.

## Exercise

1. Build the cluster and shard `shop.orders` on `{customerId: "hashed"}`. Use explain to find a query shape that is
   targeted to *two* shards.
2. Run `find({customerId: 4242, city: "Pune"})` and `find({city: "Pune"})` from 10 parallel clients for 30
   seconds each. Compare throughput. Then stop one shard and run each again. What does each query do?
3. With the 1 MB chunk size, watch `logs.events` for longer and record when the balancer next moves data. Lower
   `orphanCleanupDelaySecs` (lab only) and confirm `shB`'s local count drops.
4. Use `sh.moveChunk` (or `moveRange`) to move a range of `logs.events` by hand. Read the donor's log for the
   critical section duration.
5. Explain an aggregation with a `$sort` on `total` followed by `$limit: 10`, and one with `$group` on
   `customerId`. Where does each pipeline split, and which does more work on the merging side?
