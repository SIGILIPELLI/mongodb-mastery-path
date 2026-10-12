---
description: "Measure MongoDB write concerns on a three-node set, then watch stale secondary reads, causal sessions and read concern majority behave under induced lag."
---

# 02 · Write Concern, Read Concern & Read Preference

Lesson 01 ended with five acknowledged writes being rolled back. Whether that can happen to your data is
decided by three knobs that every MongoDB operation carries, whether you set them or not:

- **Write concern** — how many members must have a write before the server says "done".
- **Read concern** — which version of the data a read may return: the newest, or only what can never be rolled back.
- **Read preference** — which member the driver sends a read to.

The defaults are sensible. This lesson measures what each alternative costs and shows, on the same three-node
set, exactly what each one lets you see. All numbers come from MongoDB 9.0.2 and PyMongo 4.18.3 on a laptop where
all three members share one SSD; on real networks the gaps between options grow, but their order doesn't.

## Write concern: the options and their price

```python
client = MongoClient("mongodb://127.0.0.1:27301,127.0.0.1:27302,127.0.0.1:27303/?replicaSet=rs4")
print("default write concern:", client.admin.command("getDefaultRWConcern")["defaultWriteConcern"])

def bench(label, wc, n=300):
    coll = db.get_collection("t", write_concern=wc)
    t = time.perf_counter()
    for i in range(n):
        coll.insert_one({"i": i})
    ...

bench("w=0 (unacknowledged)", WriteConcern(w=0))
bench("w=1", WriteConcern(w=1))
bench("w=1, j=true", WriteConcern(w=1, j=True))
bench("w=majority", WriteConcern(w="majority"))
bench("w=3", WriteConcern(w=3))
```

```text
default write concern: {'w': 'majority', 'wtimeout': 0}
w=0 (unacknowledged)   300 inserts in    10.9 ms  (0.04 ms each)
w=1                    300 inserts in   131.6 ms  (0.44 ms each)
w=1, j=true            300 inserts in  2857.9 ms  (9.53 ms each)
w=majority             300 inserts in  2893.4 ms  (9.64 ms each)
w=3                    300 inserts in   288.5 ms  (0.96 ms each)
```

A second run gave the same shape (9.33 ms per `majority` insert, 1.04 ms per `w=3`). Reading it row by row:

- **`w: 0`** doesn't wait for any reply. The driver can't even tell you about a duplicate key. Almost never right.
- **`w: 1`** waits until the primary has applied the write in memory. Fast, and exactly what was rolled back in
  Lesson 01.
- **`j: true`** waits for the primary's journal to be flushed to disk. That's where the time goes on this machine:
  about 9 ms per `fsync`.
- **`w: "majority"`** costs the same as `j: true`, because with `writeConcernMajorityJournalDefault: true`
  (the default) a majority acknowledgement means "journaled on a majority".
- **`w: 3`** — all three members — was *faster* than `majority`, which surprises most people. A numeric `w` only
  waits for the members to *apply* the write, not to journal it; three in-memory copies are not the same as
  durable copies. If you want "every member, on disk", write `{w: 3, j: true}`.

This is why batching matters for durable writes: one `insert_many` of 300 documents pays for one majority
acknowledgement, not 300. The transaction benchmark in Level 3 · 06 was the same effect.

## When members are missing

One secondary is killed, then each write concern is tried with a 3-second `wtimeout`:

```python
for label, wc in [("w=majority", WriteConcern(w="majority", wtimeout=3000)),
                  ("w=3", WriteConcern(w=3, wtimeout=3000))]:
    try:
        db.get_collection("t", write_concern=wc).insert_one({"tag": label})
    except WriteConcernError as e:
        ...
print("documents tagged w=3 that exist anyway:", db.t.count_documents({"tag": "w=3"}))
```

```text
w=majority: ok in 349 ms
w=3: WTimeoutError after 3015 ms -> {'w': 3, 'wtimeout': 3000, 'provenance': 'clientSupplied'}
documents tagged w=3 that exist anyway: 1
```

`majority` (two of three) still succeeds. `w: 3` can't, and without `wtimeout` it would **wait forever** — the
default `wtimeout` is 0, meaning no limit. And the failed write *exists*: it was applied on the primary and the
surviving secondary. A write concern error means "I couldn't confirm the level you asked for", never "the write
was undone". Code that re-inserts on `WTimeoutError` creates duplicates; use idempotent writes (upserts on a
natural key) if you need to retry.

This is also why `w` should be `majority` rather than the number of members: it keeps working through the loss
of a minority, which is the whole point of having replicas.

## Read preference: where reads go

By default every read goes to the primary, so you read what you (or anyone) last wrote. Read preferences
`primaryPreferred`, `secondary`, `secondaryPreferred` and `nearest` let the driver use secondaries — and
secondaries can be behind.

To make lag visible, the experiment freezes one secondary with `fsync` + `lock`, which stops it applying the
oplog, and tags members so a read can be aimed at a specific node:

```javascript
const c = rs.conf(); c.members.forEach((m, i) => m.tags = {node: "n" + (i + 1)}); rs.reconfig(c)
```

```python
from_n1 = client.shop.get_collection("prices", read_preference=Secondary(tag_sets=[{"node": "n1"}]))

n1.admin.command("fsync", lock=True)                            # n1 stops applying the oplog
prices.update_one({"_id": "widget"}, {"$set": {"price": 120}})  # majority = primary + n2
print("primary:", prices.find_one()["price"], "| secondary n1:", from_n1.find_one()["price"])
```

```text
primary: 120 | secondary n1: 100
```

The update succeeded with the default `majority` write concern — the primary and `n2` were enough — and a read
routed to `n1` returned the old price. Nothing errors; the data is simply older. Real lag comes from heavy write
bursts, a slow disk, an index build, or a member in another region.

Tag sets are also how you route reads deliberately — `{dc: "mumbai"}` for local reads, `{use: "reporting"}` for
an analytics node — and `maxStalenessSeconds` tells the driver to skip secondaries estimated to be too far behind.
It has a floor, which the driver enforces:

```text
ConfigurationError maxStalenessSeconds must be at least heartbeatFrequencyMS + 10 seconds.
maxStalenessSeconds is set to 10, heartbeatFrequencyMS is set to 10000.
```

(The server-selection specification sets the minimum at 90 seconds.) It's a coarse guard against a badly lagging
node, not a freshness guarantee.

## Causal consistency: reading your own writes from a secondary

A causally consistent session tracks the cluster time of your last operation and sends it with the next one as
`afterClusterTime`. A member that hasn't reached that point **waits** rather than answering with older data:

```python
with client.start_session(causal_consistency=True) as s:
    prices.update_one({"_id": "widget"}, {"$set": {"price": 130}}, session=s)
    try:
        from_n1.find_one({}, session=s, max_time_ms=2000)
    except ExecutionTimeout:
        print(...)
    ok = client.shop.get_collection("prices", read_preference=Secondary(tag_sets=[{"node": "n2"}]))
    print("causal read on caught-up n2:", ok.find_one({}, session=s)["price"])
```

```text
causal read on lagging n1 waited, then timed out after 2.0s
causal read on caught-up n2: 130
```

The frozen secondary could not show this session its own write, so it blocked until `max_time_ms` expired; the
caught-up one returned 130. Within one session you never go backwards in time — but only within that session,
and the guarantee only holds across failures if writes use `majority` and reads use `majority` read concern.

## Read concern: newest or durable?

Both secondaries are now frozen and the primary takes a `w: 1` write — the exact situation that produced a
rollback in Lesson 01:

```python
n2.admin.command("fsync", lock=True)
prices.with_options(write_concern=WriteConcern(w=1)).update_one({"_id": "widget"}, {"$set": {"price": 999}})
local = prices.with_options(read_concern=ReadConcern("local")).find_one()["price"]
major = prices.with_options(read_concern=ReadConcern("majority")).find_one()["price"]
```

```text
on the primary -> local: 999 | majority: 130
after unlock -> majority: 999 | n1: 999
```

On the same server, at the same moment:

- **`local`** (the default for reads) returned 999 — the newest data, which could still be rolled back.
- **`majority`** returned 130 — the newest version acknowledged by a majority, which can never be rolled back.

Once the secondaries resumed and caught up, `majority` advanced to 999. Use `majority` reads when acting on data
that must not "un-happen" — a payment status, an inventory reservation, anything you show the user as final.

The other levels: **`available`** (like local, relevant for sharded clusters where it can return orphaned
documents), **`snapshot`** (a consistent point-in-time view, used in transactions — Level 3 · 06), and
**`linearizable`**, which confirms with a majority that the primary is still primary before returning:

```text
linearizable read on primary ok
NotWritablePrimary cannot satisfy linearizable read concern on non-primary node
```

It only works on the primary, adds a round trip, and should be paired with `maxTimeMS`.

## A sensible default set

| Situation | Write concern | Read concern | Read preference |
|-----------|---------------|--------------|-----------------|
| Most application data | `majority` (default) | `local` (default) | `primary` (default) |
| Must-not-lose and must-not-un-happen (payments, orders) | `majority` | `majority` | `primary` |
| Read-your-writes from secondaries | `majority` | `majority` | `secondary*` + causal session |
| Analytics / reports that tolerate minutes of lag | n/a | `local` | `secondary` with tags |
| Logs or metrics you can afford to lose | `w: 1` | `local` | `primary` |

Reading from secondaries is **not** a free scaling strategy: every secondary applies every write, so read capacity
added by secondaries comes with stale reads, and if one member fails the remaining ones must absorb its reads.
Scale reads with indexes and caching first; scale writes with sharding (Lessons 03–04).

## How It Actually Works

Every write gets an oplog timestamp. Each secondary continuously reports to its sync source the timestamps it has
**applied**, **written to its journal**, and so on; the primary combines these into a **majority commit point** —
the newest timestamp that a majority of voting members have durably. A write waiting for `w: "majority"` is parked
until the commit point passes its timestamp; a write waiting for `w: 3` is parked until three members report
having applied it.

Reads use WiredTiger's multi-version storage. A `local` read uses the newest data. A `majority` read uses a
storage snapshot at the majority commit point — the server keeps enough old versions in its cache (and the history
store) to serve it, which is why a set whose majority can't advance accumulates cache pressure. Causal consistency
rides on the **cluster time**: every response carries `operationTime`, the driver remembers the largest it has
seen in the session and sends it as `afterClusterTime`, and the receiving member waits until its own applied
oplog has reached that timestamp. Read preference, finally, is purely client-side: the driver monitors every
member (`hello` every `heartbeatFrequencyMS`, 10 seconds by default), keeps a round-trip average for each, and
picks from eligible members within a latency window (`localThresholdMS`, 15 ms by default) of the fastest.

## Common mistakes

- **`w: <number of members>`** instead of `majority`, so one failed node stops all writes (or hangs them, with no
  `wtimeout`).
- **Treating `WTimeoutError` as failure** and retrying a non-idempotent insert.
- **Believing `w: 3` means three durable copies.** Add `j: true` if that's what you need.
- **Reading from secondaries to "scale" and then being surprised by stale data**, for example a user who doesn't
  see the comment they just posted.
- **Using `majority` writes but `local` reads** for decisions that must not be reversed.
- **`linearizable` reads without `maxTimeMS`**, which can hang during a partition.
- **Per-operation overrides scattered through code.** Set concerns once on the client or collection and override
  rarely, on purpose.

## Exercise

1. Re-run the write-concern benchmark with `insert_many` batches of 100. How much does `majority` cost per document
   now?
2. Change the cluster-wide default to `w: 1` with `setDefaultRWConcern`, confirm `getDefaultRWConcern` reports it,
   and change it back. When would a cluster-wide override be justified?
3. With one secondary frozen, run a loop that writes and immediately reads back from `secondaryPreferred` without a
   causal session. How often does the read miss the write? Repeat with a causal session.
4. Write a function `read_final_status(order_id)` that returns an order's status only if it can never be rolled
   back. Prove it with the frozen-secondaries setup: a `w: 1` status change must not be visible through it.
5. Add a fourth member tagged `{use: "reporting"}` with `priority: 0`, and route an aggregation to it with a tag set.
   What happens to that aggregation when the reporting member is down, with `secondary` versus
   `secondaryPreferred`?
