---
description: "Change a live MongoDB without downtime: rolling restarts and upgrades, featureCompatibilityVersion, online index builds, batched migrations."
---

# 09 · Upgrades, FCV & Online Schema Migrations

A production database is never finished. Servers need upgrades and patches, workloads need new indexes, and
schemas change as the application does. With a replica set and a little discipline, all of these happen while the
application keeps running. This lesson does each one on the three-node lab set with a writer running throughout,
measuring what the application actually experienced.

Lab: MongoDB 9.0.2, PyMongo 4.18.3. One limitation up front: only one server version was installed for this
course, so the binary swap in a version upgrade was **not** performed here. The rolling procedure below is
identical with or without a new binary, and it was run for real; the `featureCompatibilityVersion` steps were run
on 9.0.2 itself.

## Rolling restarts: the core procedure

Patching the OS, changing a startup option, rotating certificates, upgrading MongoDB — all follow the same
pattern: **one member at a time, secondaries first, primary last**.

1. Restart one secondary (with its new binary or configuration). Wait until it's `SECONDARY` and caught up.
2. Repeat for every other secondary.
3. Step the primary down (`replSetStepDown`), wait for a new primary, restart the old one.

A script did exactly that while a thread inserted a document every 20 ms:

```python
for p in (27301, 27302, 27303):
    if p != primary:
        log(f"restarting secondary {p}"); restart(p)        # shutdown, start mongod, wait for state 1 or 2
log(f"stepping down primary {primary}")
direct(primary).admin.command("replSetStepDown", 60)
restart(primary)
```

```text
t=  2.0s  restarting secondary 27301
t= 18.8s  27301 back as SECONDARY
t= 18.8s  restarting secondary 27302
t= 35.9s  27302 back as SECONDARY
t= 35.9s  stepping down primary 27303
t= 36.0s  new primary 127.0.0.1:27301; restarting 27303
t= 52.6s  27303 back as SECONDARY
t= 67.7s  primary now 127.0.0.1:27303
inserts: 2037, errors: 0 []
slowest inserts: 0.07s at t=28.2s, 0.05s at t=0.0s, 0.04s at t=35.1s
```

Every member restarted, there were two elections, and the application saw **zero errors** and no insert slower than
70 ms. Two details:

- The stepped-down member came back and **took over again** (t = 67.7 s). It had `priority: 2` from Lesson 01, so a
  priority takeover followed as soon as it caught up — a second, unplanned-looking election. Harmless here, but
  know your priorities before maintenance so the last step doesn't surprise you.
- Most of each member's ~17 seconds was a clean shutdown and startup. Wait for each member to be healthy and caught
  up (`rs.status()` optimes) before touching the next; restarting two at once drops below a majority.

For a **sharded cluster**, the order is: stop the balancer, upgrade the config server replica set (rolling), then
each shard (rolling), then the `mongos` routers, then re-enable the balancer.

## Version upgrades and featureCompatibilityVersion

Upgrades go one major release at a time (for example 7.0 → 8.0 → 9.0, never skipping), using the rolling
procedure to replace binaries. But new binaries alone don't turn on new features that change on-disk formats or
replication behaviour. That's controlled by the **featureCompatibilityVersion** (FCV):

```javascript
db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1})
```

```text
binary: 9.0.2 | FCV: {"version":"9.0"}
```

The safe sequence:

1. Rolling-upgrade all members to the new binaries. FCV stays at the old version, so the new binaries behave
   compatibly with the old ones — you can still roll back to the old binaries.
2. Run the application on the new binaries for a while (days, typically).
3. Raise FCV: `db.adminCommand({setFeatureCompatibilityVersion: "9.0", confirm: true})`. New features become
   available; rolling back the binaries now requires lowering FCV first.

Lowering FCV on the 9.0.2 server:

```text
without confirm: Location7369100 - To proceed with the FCV downgrade, please re-run this command with 'confirm: true'.
downgrade ok: 1 in 16 ms | FCV: {"version":"8.0"}
to 7.0: Location4926900 - Invalid feature compatibility version value '7.0'. Expected one of the following versions: ''9.0', '8.3', '8.0'. ...
upgrade ok: 1 | FCV: {"version":"9.0"}
```

The `confirm: true` requirement exists because FCV changes are hard to undo in practice. The server accepts only
the adjacent versions it can interoperate with — here 8.0, 8.3 (a rapid release) and 9.0 — and on an empty
standalone the change took 16 ms. On a large deployment, changing FCV can rewrite metadata and take longer; read
the release notes' compatibility section for every version you cross, since each release documents what its FCV
gates and which features must be removed before a downgrade.

Before any upgrade: check driver compatibility with the new server, read the compatibility-changes page, test the
application against the new version in staging, take (and test) a backup, and confirm the oplog window is long
enough to cover a member being down for its restart.

## Online index builds

Adding an index to a large collection used to be a planned outage. Since 4.2 index builds take an exclusive lock
only briefly at the start and end, and replica sets build on all data-bearing members at once, committing when a
**commit quorum** (by default all voting data-bearing members) is ready.

A writer updating random accounts in `bank.accounts` (500,000 documents), before and during a new index build:

```python
a.create_index([("owner", 1), ("balance", -1)], name="vip_balance")
```

```text
before  updates:   523 | median  9.53 ms | p99  15.57 ms | max   23.3 ms
during  updates:   146 | median 10.04 ms | p99  27.33 ms | max   40.5 ms
index build (all 3 members, commit quorum) took 1.6s
```

```text
"msg":"Index build: commit quorum satisfied"
"msg":"Index build: done building"
"msg":"Index build: completed successfully"
```

Writes continued throughout; the tail latency roughly doubled (the build competes for CPU, cache and disk), and
the median barely moved. On a collection hundreds of times larger the build runs for minutes or hours, so the
practical advice is: build during quieter periods, watch replication lag, and make sure every member has the disk
space. Hiding a new index from the planner first is possible too — `hideIndex` makes an existing index invisible
to queries without dropping it, which is the safe way to test whether an index can be removed.

## Online schema migrations

MongoDB doesn't need `ALTER TABLE`, but data still has a shape the code depends on. Changing it safely is an
application-level process. The example: 200,000 users stored as

```json
{"_id": 4242, "name": "Ravi Shah", "phone": "9895420477"}
```

must become

```json
{"_id": 4242, "name": {"first": "Ravi", "last": "Shah"}, "phones": ["9895420477"], "schemaVersion": 2}
```

without stopping the application. The steps:

**1. Deploy code that understands both shapes.** Here the code that adds a phone number checks the document's
version and writes the matching shape — and checks that its update still matched, because the migration may have
converted the document between the read and the write:

```python
def add_phone(uid, phone):
    while True:
        doc = users.find_one({"_id": uid}, {"schemaVersion": 1})
        if doc.get("schemaVersion") == 2:
            res = users.update_one({"_id": uid, "schemaVersion": 2}, {"$addToSet": {"phones": phone}})
        else:   # still v1: write the v1 shape; the migration will carry it over
            res = users.update_one({"_id": uid, "schemaVersion": {"$exists": False}}, {"$set": {"phone": phone}})
        if res.matched_count == 1:
            return
        races[0] += 1           # migrated between our read and our write: read again
```

**2. Migrate in batches**, each an update-with-pipeline that transforms a document in one atomic step, filtered so
that already-migrated documents are skipped:

```python
to_v2 = [{"$set": {
    "name": {"first": {"$arrayElemAt": [{"$split": ["$name", " "]}, 0]},
             "last":  {"$arrayElemAt": [{"$split": ["$name", " "]}, 1]}},
    "phones": ["$phone"], "schemaVersion": 2}}, {"$unset": "phone"}]

while True:
    ids = [d["_id"] for d in users.find({"_id": {"$gt": last_id}}, {"_id": 1}).sort("_id", 1).limit(2000)]
    if not ids: break
    users.update_many({"_id": {"$gte": ids[0], "$lte": ids[-1]}, "schemaVersion": {"$exists": False}}, to_v2)
    last_id = ids[-1]
    time.sleep(0.01)          # throttle: leave room for the application and for replication
```

```text
v1 users: 200000
app writes that lost the race and retried: 0
migrated 200000 in 100 batches, 7.6s, while the app made 620 writes
left on v1: 0 | v2 docs with a stray 'phone' field: 0
sample: {'_id': 4242, 'name': {'first': 'Ravi', 'last': 'Shah'}, 'phones': ['9895420477'], 'schemaVersion': 2}
re-running the migration modifies: 0
```

Every document converted, concurrent application writes landed in the right shape, and re-running the migration
changed nothing — it's **idempotent**, so a crash halfway is fixed by running it again. In this run the race the
retry loop guards against never fired (0 of 620 writes), because the window between read and write is tiny; an
earlier version of the code without the `matched_count` check would have silently dropped a phone number whenever
it did. Rare is not never.

**3. Lock in the new shape** with validation once nothing writes v1 any more:

```python
c.app.command("collMod", "users", validator={"$jsonSchema": {"required": ["schemaVersion", "name", "phones"], ...}},
              validationLevel="strict", validationAction="error")
```

```text
v1-shaped insert after validation: WriteError
```

**4. Remove the v1 code paths** in a later deploy.

Why batches and not one `updateMany({}, ...)`? A single huge update produces one oplog entry per document in a
burst that secondaries must absorb (Lesson 01), can't be throttled or paused, and holds resources for its whole
duration. Batches let you watch replication lag and slow down. For the very large migrations, an alternative is
**lazy migration**: convert each document whenever the application next writes it, plus a slow background job for
the rest.

## How It Actually Works

A rolling restart works because each member can be absent without losing the majority, and because drivers treat a
restart like any other failover: they notice the topology change, rediscover the primary, and retry the write once.
A step-down is gentler than a crash because the primary first waits for a secondary to catch up, then that
secondary wins an election immediately.

FCV is stored as a document in the `admin.system.version` collection, replicated like any other write. New code
paths in the server check the FCV before using new on-disk formats or protocol features, so a set at the old FCV
writes nothing an old binary can't read. Raising FCV runs an upgrade procedure that may convert metadata; lowering
it runs the reverse and refuses if data exists that the old version couldn't handle.

Hybrid index builds scan the collection without holding an exclusive lock, while concurrent writes to the
collection are recorded in a side table; at the end, the build applies the side table and takes a brief exclusive
lock to commit. In a replica set the primary writes a `startIndexBuild` oplog entry so every member starts building,
each member votes when it's ready, and the primary writes `commitIndexBuild` once the commit quorum is met.

## Common mistakes

- **Restarting two members at once**, or the primary first.
- **Skipping major versions** or raising FCV the moment new binaries are in, removing the easy rollback.
- **Forgetting the balancer** during sharded-cluster maintenance.
- **Big-bang `updateMany` migrations** on large collections in production.
- **Non-idempotent migrations** that double-transform documents when re-run.
- **Code that only understands the new shape**, deployed before the migration finishes.
- **Read-then-write without checking `matched_count`**, silently losing writes during the migration.
- **Dropping an index without hiding it first** to check nothing depends on it.

## Exercise

1. Write your own rolling-restart script that refuses to proceed if any member is unhealthy or lagging more than
   10 seconds. Run it with a writer going, and add a deliberate failure (a member that doesn't come back) to check
   it stops.
2. Before Lesson 01's priority change, what would the end of the rolling restart have looked like? Reset all
   priorities to 1 and run it again.
3. Hide `vip_balance` with `hideIndex`, run a query that would use it and check explain, then unhide it.
4. Migrate `bank.accounts.balance` from integers to `Decimal128` in batches while the transfer writer from Lesson 06
   runs. Prove the total is unchanged afterwards.
5. Implement the lazy-migration variant for the users example: convert on every write, plus a background job
   limited to 500 documents per second. How long until no v1 documents remain?
