---
description: "Monitor MongoDB: serverStatus metrics that matter, mongostat and mongotop, the slow-query log, the profiler, and killing runaway operations."
---

# 07 · Monitoring & the Profiler

When a MongoDB application gets slow, the database usually already knows why. It counts every operation, times
them, logs the slow ones with their plans, tracks its cache and its connections, and will list what's running
right now. Monitoring is mostly knowing where those numbers live and which ones to watch.

This lesson reads them from the three-node set (MongoDB 9.0.2, Database Tools 100.19.1) while the bank-transfer
writer from Lesson 06 generates load. Hosted monitoring (Atlas charts, Ops Manager, commercial APM tools) reads
the same server metrics and isn't covered here.

## `serverStatus`: the counters behind every dashboard

```javascript
const s = db.serverStatus()
```

A few fields, as read from the primary after the previous lessons' experiments:

```text
uptime s: 1500 | connections: {"current":24,"available":104833,"totalCreated":121}
opcounters: insert 554003, query 52, update 4905, delete 9, getmore 16484, command 34992, aggregate 4518
opLatencies (micros total / ops): reads 187us avg over 16513, writes 2347us avg over 7874, commands 684us avg over 34844
WT cache: 131.0 MB used of 256 MB | dirty 6.8 MB | pages read into cache 3797
```

What each tells you, and what to alert on:

- **`connections`** — `current` against what you expect from your application fleet. A steady climb usually means
  a client creating a new `MongoClient` per request (Level 1 · 08). `totalCreated` racing upward means connection
  churn, which is expensive (each new connection does a TLS handshake and authentication).
- **`opcounters`** — cumulative counts since startup. Dashboards plot their *rate*. Note the replicated
  operations a secondary applies are counted separately (`opcountersRepl`).
- **`opLatencies`** — cumulative microseconds and operation counts for reads, writes and commands; the change in
  `latency` divided by the change in `ops` over an interval is average latency. Writes averaging 2.3 ms here
  reflects `majority` write concern (Lesson 02). Averages hide outliers; histograms are available with
  `serverStatus({opLatencies: {histograms: true}})`.
- **WiredTiger cache** — `bytes currently in the cache` against `maximum bytes configured`, and dirty bytes. When
  dirty data stays high (WiredTiger aims to keep it well below the cache size) application threads get drafted
  into evicting pages, and latency jumps. `pages read into cache` rising fast means the working set doesn't fit
  in memory. Lesson 08 explains the mechanics.
- **`queues.execution`** — admission tickets for reads and writes. This server reported 8 read and 8 write tickets
  available with throughput probing on, meaning the server adjusts concurrency itself; operations queueing for a
  ticket show up as `queueLength` above zero.

Everything in `serverStatus` is also recorded continuously, about once per second, into the `diagnostic.data`
directory in the dbpath (FTDC — "full-time diagnostic data capture"):

```text
metrics.2026-10-12T18-22-04Z-00000
metrics.2026-10-12T18-23-06Z-00000
```

Those compressed files are what MongoDB support asks for after an incident; keep them when you collect logs.

## `mongostat` and `mongotop`: the live view

`mongostat` prints one line per second of rates from `serverStatus`:

```bash
mongostat --host 127.0.0.1:27303 -n 4 1
```

```text
insert query update delete getmore command dirty  used flushes vsize  res qrw arw net_in net_out conn set repl                time
    *0    *0    190     *0     381   675|0  3.6% 67.9%       0  467G 134M 0|0 0|0   853k    792k   27 rs4  SLV Oct 13 00:21:44.348
    *0    *0    214     *0     428   750|0 12.5% 68.5%       0  467G 110M 0|0 0|0   953k    860k   27 rs4  SLV Oct 13 00:21:45.347
    *0    *0    208     *0     416   732|0 11.2% 68.7%       0  467G 132M 0|0 0|0   926k    845k   27 rs4  SLV Oct 13 00:21:46.347
    *0    *0    209     *0     419   730|0 10.2% 68.7%       0  467G 133M 0|0 0|0   929k    846k   27 rs4  SLV Oct 13 00:21:47.349
```

About 200 updates per second — 100 transfers, two updates each. `dirty` and `used` are WiredTiger cache
percentages; `qrw`/`arw` are queued and active readers|writers; `getmore` here is mostly secondaries pulling the
oplog. `vsize` is virtual address space and means little on 64-bit systems; `res` (resident memory) matters more.

One honest oddity: `rs.status()` reported 27303 as **PRIMARY** at this moment, yet this `mongostat` labelled it
`SLV`, and `mongostat --discover` against the set listed only the two secondaries. The tools and server are
versioned separately, and this tools release appears to misread the 9.0 server's role. The rate columns were
right; for roles, trust `rs.status()` or `db.hello()`.

`mongotop` shows where time is spent per collection:

```bash
mongotop --host 127.0.0.1:27303 -n 1 2
```

```text
                     ns    total    read    write
         local.oplog.rs     21ms    21ms      0ms
          bank.accounts     12ms     0ms     12ms
config.image_collection     12ms    12ms      0ms
    config.transactions      6ms     6ms      0ms
```

Reading the oplog (secondaries fetching) was the busiest namespace, followed by the transfers' writes.
`config.transactions` and `config.image_collection` are internal: the server records retryable-write and
transaction state there.

## The slow-query log

Any operation slower than `slowms` (100 ms by default) is written to the server log with its full details. One
unindexed query:

```javascript
db.getSiblingDB("bank").accounts.find({owner: "nobody"}).toArray()
```

```text
"msg":"Slow query", ... {"type": "command", "ns": "bank.accounts", "command": {"find": "accounts",
"filter": {"owner": "nobody"}, ...}, "planSummary": "COLLSCAN", "planningTimeMicros": 495, "keysExamined": 0,
"docsExamined": 500000, "nreturned": 0, "numYields": 10,
"queryShapeHash": "5F869006B0C736375D7D96EDFAF043B6745867B9A6E0375A18A3AA18E38D3B27", "durationMillis": 179}
```

Everything needed to diagnose it is on one line: a collection scan examined 500,000 documents to return none.
`docsExamined` far above `nreturned` is the classic signature of a missing index (Level 2 · 07). The
`queryShapeHash` groups all queries of the same *shape* regardless of the literal values, so you can count how
often this pattern runs rather than chasing individual instances.

Because the log is structured JSON, it's easy to aggregate: ship it to whatever log system you use and chart
slow queries by `queryShapeHash`, `ns` and `planSummary`.

## The profiler

The profiler records operations into the capped `system.profile` collection of a database, where you can *query*
them. Level 1 records operations slower than `slowms`:

```javascript
const bank = db.getSiblingDB("bank")
bank.setProfilingLevel(1, {slowms: 20})
```

```text
{ was: 0, slowms: 100, slowinprogms: Long('5000'), sampleRate: 1, ok: 1 }
```

Three operations followed: the unindexed `find`, a lookup by `_id`, and an unindexed `updateMany`:

```javascript
bank.system.profile.find({}, {op: 1, ns: 1, command: 1, planSummary: 1, keysExamined: 1, docsExamined: 1,
                              nreturned: 1, nModified: 1, millis: 1, numYield: 1}).sort({ts: -1})
```

```text
update bank.accounts COLLSCAN keys 0 docs 500000 returned 3 millis 130 yields 8 | filter {"balance":{"$gt":1010}}
query  bank.accounts COLLSCAN keys 0 docs 500000 returned 0 millis 179 yields 10 | filter {"owner":"nobody"}
profile entries: 2 | capped: true
```

The `_id` lookup was too fast to record. Both slow operations scanned the whole collection; the update modified 3
documents after reading 500,000.

Profiler guidance:

- **Level 1 with a sensible `slowms`** is safe in production for short investigations; `sampleRate` (0–1) records
  only a fraction of slow operations on very busy systems.
- **Level 2 records everything** and adds a write for every operation. Use it only briefly on a test system.
- It's **per database and per `mongod`**: on a replica set, enable it on the member whose traffic you're
  investigating; on a sharded cluster, on the shards.
- `system.profile` is a small capped collection (1 MB by default), so old entries vanish. Copy out what you need.
- Setting it at level 0 still changes the `slowms` threshold used for the log.

## What's running right now: `$currentOp` and `killOp`

A report someone started with a pathological pipeline — a `$lookup` that scans the whole collection 200 times:

```javascript
db.getSiblingDB("bank").accounts.aggregate([
  {$group: {_id: {$mod: ["$_id", 200]}}},
  {$lookup: {from: "accounts", let: {k: "$_id"},
             pipeline: [{$match: {$expr: {$eq: [{$mod: ["$_id", 200]}, "$$k"]}}}, {$count: "c"}], as: "same"}}
], {comment: "monthly-report"})
```

From another shell, find operations running longer than 2 seconds:

```javascript
db.getSiblingDB("admin").aggregate([{$currentOp: {}}, {$match: {ns: /^bank\./, secs_running: {$gte: 2}}}])
```

```text
opid 307201 | op command | ns bank.accounts | secs_running Long('5') | planSummary COLLSCAN | comment monthly-report | app mongosh 2.13.0
```

```javascript
db.killOp(307201)
```

```text
killOp ok: 1
long query ended: Interrupted - Executor error during aggregate command on namespace: bank.accounts :: caused by :: operation was interrupted
```

Two habits make this much easier in a real incident: **set `appName`** in every application's connection string
so `$currentOp` shows which service an operation came from, and **pass a `comment`** on expensive queries so you
can recognise them. Better still, prevent runaway queries with `maxTimeMS` on anything user-triggered.

(Two earlier versions of this "long query" finished too fast or failed outright — one built an array over the
100 MB per-array limit and died with `ExceededMemoryLimit`. Aggregations have guard rails; the `$count`
sub-pipeline keeps arrays tiny.)

## What to put on a dashboard

| Signal | Source | Why |
|--------|--------|-----|
| Operation rates and latencies by type | `opcounters`, `opLatencies` | load and the first sign of trouble |
| Slow queries per shape | slow log / profiler (`queryShapeHash`) | which query to fix |
| Replication lag per secondary | `rs.status()` optimes | stale reads, failover risk, oplog window |
| Oplog window (hours) | `rs.printReplicationInfo()` | how long a member can be down |
| Cache used / dirty, pages read | `wiredTiger.cache` | working set vs memory |
| Connections current / created | `connections` | pool misconfiguration, churn |
| Queued operations, tickets | `queues.execution` | saturation |
| Disk space, IOPS, CPU | OS | everything else depends on them |
| Elections | log, `rs.status().term` | instability (Lesson 01's laptop had 104 overnight) |

## How It Actually Works

Every operation passes through a `CurOp` object that accumulates its statistics as it runs — keys and documents
examined, yields, locks, time spent waiting. `$currentOp` reads the live `CurOp`s; when an operation finishes,
the same object is used to decide whether to log it (duration above `slowms`, subject to `sampleRate`) and whether
to write a profiler document. Long-running operations periodically check whether they've been **interrupted** —
at yield points, between batches — which is how `killOp` and `maxTimeMS` work: they set a flag that the
operation notices at its next check, then it cleans up and returns `Interrupted` or `MaxTimeMSExpired`.

`serverStatus` gathers counters maintained by each subsystem, many as lock-free per-CPU counters so that counting
millions of operations doesn't itself become a bottleneck. FTDC samples `serverStatus`, `replSetGetStatus` and a
few others on a timer and stores them with delta compression, which is why a day of once-per-second samples takes
only megabytes.

## Common mistakes

- **Monitoring only CPU and memory** and missing replication lag, cache pressure and queueing.
- **Profiler level 2 in production.**
- **Looking at averages only.** One slow query shape at the 99th percentile is invisible in a mean.
- **No `appName`, no `comment`**, so `$currentOp` shows anonymous connections.
- **Killing operations blindly**, including internal ones (replication, balancer, index builds).
- **Discarding `diagnostic.data` and logs** after an incident.
- **Alerting on a single sample** of a noisy metric instead of a sustained condition.

## Exercise

1. Write a script that samples `serverStatus` every 5 seconds and prints per-second rates for inserts, updates and
   queries, plus average read and write latency over the interval.
2. Run the transfer writer and a deliberately unindexed query loop. Find the query in the slow log, then group all
   slow log lines from the run by `queryShapeHash` and count them. Add the index and repeat.
3. Enable profiling at level 1 with `sampleRate: 0.1` and `slowms: 0` for one minute under load. Use the profile
   collection to find the three most expensive query shapes by total `millis`.
4. Start a long aggregation with `appName=reports` in its connection string. Write a "kill switch" script that kills
   only operations from that app running longer than 30 seconds.
5. Design the alert thresholds for the dashboard table above for a hypothetical service handling 2,000 operations
   per second, and write down what an on-call engineer should check first for each alert.
