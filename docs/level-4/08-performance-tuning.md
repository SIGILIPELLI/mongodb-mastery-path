---
description: "Tune MongoDB from measurements: WiredTiger cache and working set, compression trade-offs, connection pools, per-request clients, OS settings."
---

# 08 · Performance Tuning: WiredTiger, Memory & Connections

Most MongoDB performance problems are solved by the work in Levels 2 and 3 — the right indexes, a schema that
matches the queries, pipelines that filter early. This lesson covers what's left once those are done: how the
storage engine uses memory, what compression costs and saves, how connections behave, and which server and OS
settings matter. Every claim is backed by a measurement on this course's lab, with an honest note on how a laptop
differs from a production server.

Lab: MongoDB 9.0.2 on a laptop with 8 GB of RAM and 8 cores, PyMongo 4.18.3.

## The WiredTiger cache

WiredTiger keeps recently used data and index pages in its own cache, **uncompressed**. Its default size is the
larger of 256 MB and 50% of (RAM − 1 GB). Started with no cache option on the 8 GB machine:

```text
RAM MB: Long('8192') cores: 8
default cache MB: 3584
```

(8,192 − 1,024) × 0.5 = 3,584 MB — exactly the formula. The rest of memory isn't wasted: the operating system's
file cache holds recently read **compressed** data files, and every connection, sort, aggregation and index build
uses memory outside the cache.

Two consequences:

- **Never run two `mongod`s on one host with default settings.** Each claims half the RAM. That's why every lab in
  this course sets `--wiredTigerCacheSizeGB 0.25`.
- **In containers, set the cache explicitly** (or make sure the container's memory limit is visible to `mongod`).
  A cache sized from the host's RAM inside a small container gets the process killed.

## Working set: what has to fit

The **working set** is the data and indexes your workload touches regularly. The test server has a 256 MB cache
and a collection of 1.2 million documents with 400 random characters each:

```text
loaded 1200000 docs in 27s | data 490 MB | on disk 497 MB | _id index 13 MB
```

Random characters don't compress, so the collection is almost twice the cache. After a restart (empty WiredTiger
cache), 20,000 random `_id` lookups, first restricted to 5% of the documents, then across all of them:

```python
def run(label, hi, n=20000):
    r = random.Random(1); p0, _ = cache(); t = time.perf_counter()
    for _ in range(n):
        coll.find_one({"_id": r.randrange(hi)})
    ...
```

```text
hot 5% (60k docs), cold cache         9014 reads/s | pages read into cache   1280 | cache used    29 MB
hot 5% (60k docs), warm               9567 reads/s | pages read into cache      0 | cache used    29 MB
whole collection (1.2M docs)          4644 reads/s | pages read into cache  14524 | cache used   209 MB
whole collection, again               7774 reads/s | pages read into cache  12855 | cache used   218 MB
```

- The hot 5% (about 25 MB) fit easily: after the first pass it was entirely cached and **no pages were read**.
- Reading across the whole collection, the cache filled to about 210–220 MB — WiredTiger keeps it around 80% full
  and evicts to stay there — and **kept reading pages in**: 12,855 page reads on the second pass, because the
  working set no longer fits. Each read evicts something that may be needed again soon.

The honest caveat: the gap here (about 7,800 vs 9,600 reads/s once warm) is small because this laptop's operating
system had the whole 497 MB file in its own memory, so a cache miss cost a copy and decompression rather than a
disk read. On a server whose data is many times larger than RAM, a miss is a real disk I/O — tens or hundreds of
microseconds on SSDs, milliseconds on network storage — and the same pattern becomes an order-of-magnitude
slowdown. Watch **`pages read into cache`** per second: if it stays high under normal load, the working set doesn't
fit, and you need more memory, smaller documents, better indexes (fewer pages touched), or sharding.

Indexes deserve special attention: an index used by every query must stay in memory to be fast. Compare
`totalIndexSize` across collections with the cache size; unused indexes (find them with `$indexStats`) cost cache
and write time for nothing.

## Compression: disk versus CPU

WiredTiger compresses data blocks on disk with **snappy** by default; `zstd`, `zlib` and `none` are options per
collection. The same 200,000 order-like documents in each:

```javascript
p.createCollection("c_" + c, {storageEngine: {wiredTiger: {configString: "block_compressor=" + c}}})
```

```text
none    data 40.0 MB | on disk 41.7 MB | ratio 0.96 | insert 5417 ms
snappy  data 40.0 MB | on disk 13.3 MB | ratio 3.00 | insert 5393 ms
zlib    data 40.0 MB | on disk 7.2 MB | ratio 5.55 | insert 5404 ms
zstd    data 40.0 MB | on disk 8.6 MB | ratio 4.65 | insert 5391 ms
```

On this repetitive, text-heavy data, snappy cut disk use by two-thirds, and zstd and zlib by over 80% — with zlib
slightly ahead at default levels on this sample. The insert times didn't differ because the mongosh script
generating documents was the bottleneck, not the server; compression CPU cost is real but small next to
everything else in this test. The random-character collection above, by contrast, didn't compress at all
(497 MB on disk for 490 MB of data). Measure with your own documents.

Choosing:

- **snappy** (default): cheap and fast; a fine default.
- **zstd**: noticeably smaller files for modest extra CPU; good for large, cold or archival collections, and for
  network compression between client and server (`compressors=zstd` in the connection string).
- **none**: rarely worth it.
- Compression affects disk and the OS file cache, not the WiredTiger cache — pages are decompressed in memory.
  Smaller files mean more data fits in the OS file cache, which itself improves performance when the working set
  overflows WiredTiger's cache.

## Connections

### One client per process, not per request

```text
200 lookups, new client each:    298 ms, connections created: 602
200 lookups, shared client:       24 ms
```

Creating a `MongoClient` per request was **twelve times slower** and opened about three connections per client
(the driver's monitoring connection, its round-trip-time monitor, and one for the operation). Add TLS and
authentication and each new client costs several round trips plus CPU on both sides. A `MongoClient` is
thread-safe and maintains a pool; create it once at startup and share it. In serverless functions, create it
outside the handler so warm invocations reuse it.

### Pool size is not a throughput dial

64 Python threads doing random lookups through one client, with different pool limits:

```text
maxPoolSize=1       8154 reads/s  (server connections now: 6)
maxPoolSize=4       9512 reads/s  (server connections now: 9)
maxPoolSize=16      8653 reads/s  (server connections now: 21)
maxPoolSize=100     8885 reads/s  (server connections now: 69)
```

Throughput was flat: the bottleneck was the Python client (one interpreter, the GIL), not connections, so more
connections just meant more idle sockets. In other languages or with more client processes the curve rises
further before flattening, but the shape is universal: beyond the server's ability to execute operations in
parallel (bounded by cores and storage), more connections add memory and contention, not speed.

Budget connections across the fleet: 40 application instances × `maxPoolSize` 100 is 4,000 potential connections
per `mongod`, each with its own memory. Size pools from measured concurrency, set `maxIdleTimeMS` so bursts don't
leave thousands of idle sockets, and set `waitQueueTimeoutMS` / `timeoutMS` so a saturated pool fails fast rather
than queueing forever.

## Server and OS settings worth knowing

These follow MongoDB's production notes; the details change between versions, so check the notes for the version
you run. None were benchmarked here — a laptop can't meaningfully test them.

- **File system:** XFS is recommended for WiredTiger on Linux.
- **`ulimit`:** raise open files and processes (MongoDB suggests 64,000); each connection and each data/index file
  uses descriptors.
- **Swappiness:** keep it low (often 1) so the kernel doesn't swap out the cache.
- **Transparent huge pages:** the recommendation **reversed** in MongoDB 8.0 — older versions said disable THP;
  8.0 and later, with an upgraded TCMalloc, recommend enabling it. A runbook copied from an older guide gets this
  backwards.
- **NUMA:** on multi-socket machines, use interleaved memory allocation (`numactl --interleave=all`).
- **Read-ahead:** small values (around 8–32 sectors) on data volumes; large read-ahead wastes the file cache.
- **Clock sync:** NTP/chrony on every member; replication and logs depend on sane clocks.
- **Disks:** SSDs, and separate volumes for data and logs if your platform's I/O is shared.

`mongod` checks several of these at startup and prints warnings in the log (and in mongosh's startup banner).
Read them rather than suppressing them.

## A tuning method

1. **Find the expensive operations** — slow log and profiler grouped by query shape (Lesson 07).
2. **Fix the query or index first.** A `COLLSCAN` turned into an `IXSCAN` beats any server setting.
3. **Check the working set**: cache usage, dirty percentage, pages read per second.
4. **Check connections and queues**: tickets in use, queued operations, connection counts and churn.
5. **Check replication**: lag, and whether `majority` writes are waiting on a slow secondary.
6. **Change one thing, measure again**, on production-like data and load.

## How It Actually Works

WiredTiger stores each collection and index as a B-tree in its own file. Pages on disk are compressed blocks; when
a page is needed, it's read (from the OS file cache or disk), decompressed and expanded into an in-memory format in
the cache. Updates create new versions of records in memory (that's how readers get consistent snapshots without
locks), making pages **dirty**. Checkpoints (every 60 seconds by default) write dirty pages back as new compressed
blocks, and the journal records changes between checkpoints for crash recovery.

Background **eviction** threads keep the cache below its targets: they start evicting clean pages when usage
passes about 80% (`eviction_target`) and work harder on dirty content past its own threshold. If the cache reaches
its hard triggers (around 95% used, or too much dirty data), **application threads** are made to do eviction
before their operations proceed — the moment latency suddenly spikes in an overloaded system. That's why the test
above settled at about 210–220 MB of a 256 MB cache: eviction holding it near the target while new pages arrived.

## Common mistakes

- **Tuning server knobs before fixing queries and indexes.**
- **Default cache settings for several `mongod`s on one host**, or inside a memory-limited container.
- **Ignoring index size** relative to memory; keeping unused indexes.
- **A new `MongoClient` per request** or per function invocation.
- **Huge pool sizes on many app instances**, multiplying into tens of thousands of server connections.
- **Applying THP and other OS advice from a guide written for a different MongoDB version.**
- **Benchmarking on a laptop and extrapolating to production storage.** Use laptops to compare alternatives, as this
  lesson does, and production-like hardware to size.

## Exercise

1. Load a collection 3× larger than a 256 MB cache. Measure random-read throughput and pages read per second for
   hot ranges of 1%, 10%, 25% and 50%. Plot where performance falls off.
2. Repeat the compression comparison with your own application's documents (or a public data set). Add a read
   test: time a full collection scan for each compressor after a restart.
3. Write a "client per request" and a "shared client" version of a small web endpoint (Flask or FastAPI) and load
   test both. Count `totalCreated` connections on the server for each.
4. Run the pool-size test from Node.js with the async driver and 200 concurrent promises. Does the curve differ
   from Python's?
5. Read every startup warning your `mongod` prints, look up each in the production notes for your version, and
   decide which apply to your deployment.
