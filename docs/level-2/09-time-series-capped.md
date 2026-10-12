---
description: "MongoDB time series collections (buckets, compression, metaField, granularity) and bounded logs in capped collections with tailable cursors."
---

# 09 · Time Series & Capped Collections

Two kinds of data don't fit the "one document per entity" model well. **Measurements over time** —
sensor readings, metrics, prices, clicks — arrive in huge numbers, are almost never updated, and are
queried by time range and source. **Rolling logs** — the last N events, a recent-activity feed — only
matter while they're recent. MongoDB has a special collection type for each.

## Time series collections

Lesson 03 built the bucket pattern by hand: one document per sensor per hour, readings appended to an
array. A **time series collection** does that for you inside the server, and adds columnar compression on
top. You insert ordinary one-reading documents; the server groups them into buckets behind the scenes.

```javascript
db.createCollection("weather", {
  timeseries: { timeField: "ts", metaField: "sensor", granularity: "minutes" }
});
```

- **`timeField`** (required) — the BSON date of each measurement.
- **`metaField`** — what identifies the *source*: a sensor id, a host, a stock symbol. Measurements are
  bucketed only with others that have an identical `metaField` value. It can be a sub-document, as here.
- **`granularity`** — `"seconds"`, `"minutes"` or `"hours"`: roughly how often one source reports. It sets how
  much time one bucket spans. (You can instead set `bucketMaxSpanSeconds` and `bucketRoundingSeconds`
  explicitly.)

A week of minute readings from 10 sensors, with random-walk temperature and humidity so the data is
realistically noisy, inserted into both a time series collection and an ordinary one:

```javascript
for (let s = 0; s < 10; s++) { let temp = 20 + s, hum = 50;
  for (let m = 0; m < 24 * 60 * 7; m++) {
    temp += (Math.random() - 0.5) * 0.2; hum += (Math.random() - 0.5) * 0.5;
    docs.push({ ts: new Date(start + m * 60000 + Math.floor(Math.random() * 2000)),
                sensor: { id: "s" + s, site: s < 5 ? "north" : "south" },
                temp: Math.round(temp * 100) / 100, hum: Math.round(hum * 10) / 10 }); } }
db.weather.insertMany(docs, { ordered: false });
db.weather_plain.insertMany(docs, { ordered: false });
db.weather_plain.createIndex({ "sensor.id": 1, ts: 1 });
db.adminCommand({ fsync: 1 });   // flush to disk so storage sizes are real
```

```text
ts:    storageMB 0.37 indexMB 0.025 buckets 110 closedDueToCount 100
plain: storageMB 2.60 indexMB 3.924
```

The same 100,800 readings take about **7× less disk** and over **150× less index** in the time series
collection. They were stored in 110 buckets; 100 of them were closed because they reached the per-bucket
measurement limit (1,000 by default), the rest are still open for new data. On real-world data the ratio
depends heavily on how regular your values are — an earlier run of this experiment with perfectly periodic
synthetic values compressed far more, which would have been a misleading number to publish.

### It still looks like documents

```javascript
db.weather.find({}, { _id: 0 }).sort({ ts: 1 }).limit(1)
```

```text
{"ts":"2026-10-01T00:00:00.286Z","sensor":{"id":"s9","site":"south"},"hum":50.1,"temp":28.93}
```

Queries and aggregations see individual measurements; the server unpacks buckets on the fly. Hourly
averages for one sensor on one day:

```javascript
db.weather.aggregate([
  { $match: { "sensor.id": "s3", ts: { $gte: new Date("2026-10-03"), $lt: new Date("2026-10-04") } } },
  { $group: { _id: { $dateTrunc: { date: "$ts", unit: "hour" } }, avg: { $avg: "$temp" }, max: { $max: "$temp" } } },
  { $sort: { _id: 1 } }])
```

```text
{"_id":"2026-10-03T00:00:00.000Z","avg":20.541999999999998,"max":21.19}
{"_id":"2026-10-03T01:00:00.000Z","avg":20.445833333333333,"max":20.74}
...
```

(Printed as JSON, first two hours. The data is random, so your numbers will differ.)

The collection was created with an index on `{sensor: 1, ts: 1}` automatically, and the bucket store is
clustered by bucket id, so the explain of the `$match` showed a `CLUSTERED_IXSCAN` stage feeding `$_internalUnpackBucket` — a range scan over
buckets rather than over individual readings. Add secondary indexes on measurement fields if you filter on
them (`createIndex({ "sensor.site": 1, ts: -1 })` works as usual).

### Expiring old data

Time series collections take `expireAfterSeconds` directly, and remove **whole buckets** whose newest
measurement is older than the limit:

```javascript
db.createCollection("weather", { timeseries: { ... }, expireAfterSeconds: 60 * 60 * 24 * 365 * 2 })
// or later: db.runCommand({ collMod: "weather", expireAfterSeconds: ... })
```

Deleting whole buckets is far cheaper than deleting 100,000 individual documents a day.

### What doesn't fit

Time series collections are optimised for append-mostly data. Updates and deletes of individual
measurements are supported (with restrictions in older versions) but are much more expensive than in an
ordinary collection, because the server must rewrite a compressed bucket. Choose the `metaField` carefully:
it's the grouping key, every distinct value gets its own buckets, and putting something that varies per
reading (like a request id) in it destroys the bucketing. And measurements that arrive badly out of order or
far in the past may open extra buckets — check `numBucketsClosedDueTo...` counters in `stats().timeseries`
if storage looks worse than expected.

## Capped collections

A capped collection has a fixed maximum size (in bytes, and optionally in documents) and keeps documents in
insertion order. When it's full, the oldest documents are overwritten.

```javascript
db.createCollection("recent_log", { capped: true, size: 1024 * 1024, max: 5 });
for (let i = 1; i <= 8; i++) db.recent_log.insertOne({ n: i, msg: "event " + i });
db.recent_log.find()                                   // natural (insertion) order
db.recent_log.find().sort({ $natural: -1 }).limit(3)   // newest first
```

```text
4,5,6,7,8
8,7,6
```

Eight inserts, five kept, the oldest three gone with no delete calls. Sorting by `$natural` uses insertion
order with no index at all.

On this server, `deleteOne({n: 6})` succeeded and an update that grew a document by 100 bytes also
succeeded. Older versions (before 5.0) rejected deletes and size-changing updates on capped collections, so
you'll see advice saying so; check against your deployment's version.

### Tailable cursors

Capped collections support **tailable cursors**, which behave like `tail -f`: after returning the existing
documents, the cursor stays open and returns new ones as they're inserted. A Python reader with a writer
thread adding three events 0.3 s apart:

```python
cur = db.recent_log.find({}, cursor_type=CursorType.TAILABLE_AWAIT).max_await_time_ms(500)
t0, live = time.time(), 0
while cur.alive and live < 3 and time.time() - t0 < 5:
    for doc in cur:
        print(f"{time.time()-t0:4.1f}s got {doc['n']:>3} {doc['msg']}")
        live += doc["n"] >= 100
```

```text
 0.0s got   4 event 4
 0.0s got   5 event 5
 0.0s got   6 event 6
 0.0s got   7 event 7
 0.0s got   8 event 8
 0.3s got 100 live 100
 0.6s got 101 live 101
 0.9s got 102 live 102
```

Existing documents arrived immediately, then each new event about 0.3 s apart, as written. `TAILABLE_AWAIT`
makes the server hold each `getMore` open for up to `max_await_time_ms` waiting for new data, instead of the
client polling in a tight loop.

Capped collections and tailable cursors were MongoDB's original mechanism for building queues and
replication (the oplog is a capped collection). For application-level "notify me when data changes" you'll
usually want **change streams** instead (Level 3 · 07): they work on any collection, survive failovers and can
resume after a disconnect. Capped collections remain handy for bounded logs and recent-activity buffers.

## How It Actually Works

A time series collection is really two things: a **view** with the name you created (`weather`) and a hidden
bucket collection (`system.buckets.weather`). Inserts go to an in-memory **bucket catalog**, which finds an open
bucket for the measurement's `metaField` value and time range, or opens a new one. Buckets are closed when they
reach the measurement count limit, a size limit, or when a measurement arrives outside their time span — the
`numBucketsClosedDueTo*` statistics count each reason. Each bucket document stores `control.min` and
`control.max` for every field, the `meta` value once, and the measurements in a **columnar** layout: all the
`ts` values together, all the `temp` values together, compressed with delta and delta-of-delta encoding. Columns
of similar, slowly changing numbers compress extremely well, which is where the 7× came from.

When you query, the planner rewrites predicates on `timeField` and `metaField` into predicates on the bucket's
`control.min`/`control.max` and `meta`, so whole buckets can be skipped without decompressing them, and then
unpacks only the surviving buckets (the `$_internalUnpackBucket` stage you'll see in aggregation explains).
That's why queries by source and time range are cheap and queries on arbitrary measurement values without an
index are not.

A capped collection is an ordinary collection with a size limit enforced on insert: when an insert would exceed
it, the server deletes the oldest documents first. Its documents are stored in insertion order, which is what
makes `$natural` order meaningful and lets a tailable cursor simply remember its position and continue from it.

## Common mistakes

- **Per-reading values in the `metaField`**, so every measurement gets its own bucket.
- **Wrong granularity** — `"seconds"` for data that arrives hourly creates many tiny buckets.
- **Treating time series collections as general-purpose**, with frequent updates and deletes of single
  measurements.
- **Publishing compression ratios measured on synthetic, perfectly regular data.**
- **Using a capped collection as a durable queue.** Unread messages are silently overwritten when it fills.
- **Polling a collection in a loop** for new data instead of a tailable cursor or a change stream.

## Exercise

1. Create a time series collection for stock ticks (`symbol` as metaField, granularity `"seconds"`) and insert
   one day of ticks for 20 symbols with random-walk prices. Compare storage with an ordinary collection.
2. Repeat with `granularity: "hours"` and compare `bucketCount` and query time for "one symbol, one hour".
3. Write an aggregation that produces OHLC (open, high, low, close) candles per symbol per 5 minutes using
   `$dateTrunc` with `binSize: 5`, `$first` and `$last` after sorting by time.
4. Build a "last 100 audit events" capped collection and a Python tail script that prints events as they're
   written from `mongosh`. What happens to the tail if you insert 200 events faster than it reads?
