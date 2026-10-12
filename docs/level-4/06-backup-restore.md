---
description: "MongoDB backups that restore: mongodump --oplog under live load, why replay matters, point-in-time recovery before a bad deleteMany, file copies."
---

# 06 · Backup, Restore & Point-in-Time Recovery

Replication is not a backup. A replica set faithfully copies every change to every member — including the
`deleteMany({})` someone ran against production instead of staging, which reaches all secondaries within
milliseconds. A backup is a copy *from before* the mistake, plus a way to get from that copy to the moment you
actually need.

Level 1 · 09 introduced `mongodump` and `mongorestore` for moving data around. This lesson uses them as a backup
system on a live replica set: taking a consistent dump while transactions are running, proving why the oplog
matters by restoring without it, recovering to the exact second before a destructive command, and taking a
file-level copy. Tools: MongoDB 9.0.2 and Database Tools 100.19.1, on the three-node set from Lesson 01.

## The test: an invariant that must survive

A `bank.accounts` collection holds 500,000 accounts with a balance of 1,000 each. A Python writer moves 7 units
between random accounts inside transactions, so **the total must always be exactly 500,000,000**. Any backup that
restores to a different total captured a state that never existed.

```python
while time.time() < end:
    x, y = random.sample(range(500000), 2)
    with c.start_session() as s:
        s.with_transaction(lambda s: (a.update_one({"_id": x}, {"$inc": {"balance": -7}}, session=s),
                                      a.update_one({"_id": y}, {"$inc": {"balance": 7}}, session=s)))
```

## Dumping a live replica set with `--oplog`

With the writer running, dump from a secondary (keeping the load off the primary):

```bash
mongodump --uri "mongodb://127.0.0.1:27301,127.0.0.1:27302,127.0.0.1:27303/?replicaSet=rs4" \
          --readPreference secondary --oplog --out dump2
```

```text
done dumping `bank.accounts` (500000 documents)
writing captured oplog to ``
	dumped 812 oplog entries
writer transfers: 1216
```

A dump is not instantaneous: it reads each collection with a cursor while writes continue, so different documents
are copied at different moments. `--oplog` fixes that. Before starting, `mongodump` notes the newest oplog
timestamp; when it finishes, it also copies every oplog entry written *during* the dump into `oplog.bson`.
Replaying those entries on restore rolls every document forward to the same moment — the end of the dump.

Restoring the same dump into two empty servers, once without and once with replay:

```bash
mongorestore --port 27601 --nsInclude 'bank.*' dump2
mongorestore --port 27602 --oplogReplay dump2
```

```text
500000 document(s) restored successfully. 0 document(s) failed to restore.
applied 1624 oplog entries
27601 total balance: 499999958
27602 total balance: 500000000
```

Without replay, the restored bank is **42 units short**: the dump caught six transfers half-way, with one side
copied before the transfer and the other after. With replay, the total is exact. (812 oplog entries became 1,624
applied operations because each transaction's entry carries both of its updates.)

Honesty note: the first attempt used 50,000 accounts, and that dump finished in 0.4 seconds — both restores
balanced, because no transfer happened to straddle a copied pair. A small test passing doesn't prove consistency;
the larger dump did.

Two operational details:

- `--oplog` works only on a whole-instance dump (no `--db`/`--collection`), and the restore refuses filters with
  replay: `Failed: cannot use --oplogReplay with includes specified`. Restore everything to a scratch server, then
  copy out what you need.
- `--oplog` gives a consistent snapshot of a **replica set**. For a sharded cluster, `mongodump` through `mongos`
  can't produce a cluster-wide consistent point; sharded clusters need coordinated snapshots (managed services,
  Ops Manager, or careful file-level procedures).

## Point-in-time recovery

The dump finished at 18:44:43 UTC. Life goes on — 309 more transfers — and then:

```javascript
db.getSiblingDB("bank").accounts.deleteMany({_id: {$lt: 1000}})
```

```text
OOPS deleted: 1000
writer transfers: 311
live now: 499000 docs, total 498999993
```

Restoring last night's dump would lose every transfer since. What you want is the database **as it was one
operation before the delete**. That's the dump plus the oplog up to that operation.

**Step 1: find the bad operation in the oplog.** Note where the dump's own oplog ends, then search after it. In
this version a multi-document delete is written as `applyOps` batches, so search inside them:

```javascript
// last entry of dump2/oplog.bson: {'$timestamp': {'t': 1791830683, 'i': 93}}
db.getSiblingDB("local").oplog.rs.find({ts: {$gt: Timestamp(1791830683, 93)}, "o.applyOps.op": "d"})
  .sort({$natural: 1}).limit(3)
```

```text
1791830713:1 c admin.$cmd ops in entry: 10 first: {"_id":9}  lsid? false 2026-10-12T18:45:13.472Z
1791830713:2 c admin.$cmd ops in entry: 10 first: {"_id":19} lsid? false 2026-10-12T18:45:13.473Z
1791830713:3 c admin.$cmd ops in entry: 10 first: {"_id":29} lsid? false 2026-10-12T18:45:13.473Z
delete-bearing entries: 100
```

The delete begins at timestamp `1791830713:1` — 100 entries of 10 deletes each. In a real incident you'd narrow
the search by `wall` time ("around 18:45 UTC") and by namespace.

**Step 2: save the oplog since the dump** — before it rolls over, since the oplog is a fixed-size buffer:

```bash
mongodump --host rs4/127.0.0.1:27301,127.0.0.1:27302 --readPreference secondary -d local -c oplog.rs \
  --query '{"ts": {"$gt": {"$timestamp": {"t": 1791830683, "i": 93}}}}' --out oplogdump
mkdir replay && cp oplogdump/local/oplog.rs.bson replay/oplog.bson
```

```text
done dumping `local.oplog.rs` (1046 documents)
```

**Step 3: restore the dump to a scratch server, then replay up to — not including — the bad timestamp:**

```bash
mongorestore --port 27603 --oplogReplay dump2
mongorestore --port 27603 --oplogReplay --oplogLimit 1791830713:1 replay
```

```text
applied 1624 oplog entries
501679 document(s) restored successfully. 0 document(s) failed to restore.
replaying oplog
applied 1236 oplog entries
recovered: 500000 docs, total 500000000 | _id<1000 present: 1000
```

All 500,000 accounts, the exact total, and every transfer up to the moment of the delete. `--oplogLimit` replays
entries strictly *before* the given timestamp.

**Step 4: put the lost documents back**, copying only them from the scratch server to production:

```bash
mongodump --port 27603 -d bank -c accounts --query '{"_id": {"$lt": 1000}}' --out fix
mongorestore --uri "$PROD_URI" fix
```

```text
done dumping `bank.accounts` (1000 documents)
1000 document(s) restored successfully. 0 document(s) failed to restore.
live after putting them back: 500000 docs, total 499999993
```

Every account is back — and production is **7 units short**. Between the delete and the fix, a transfer touched
one of the deleted accounts: `update_one` on a missing document matched nothing and silently did nothing, while
the other half of the transfer succeeded. Restoring data cannot undo what applications did *because* data was
missing. The real fix was in the writer — check `matched_count` and abort the transaction if it isn't 1 — and the
recovery runbook should include reconciling anything written during the incident window.

## File-level copies

For large databases, copying data files is much faster than dumping documents. The files must be consistent; on
a secondary you can briefly stop writes to disk with `fsyncLock`:

```javascript
db.fsyncLock()      // on a secondary: flush and block writes
```

```bash
cp -R rs/n1 snap    # copy the whole dbpath
```

```javascript
db.fsyncUnlock()
```

```text
Long('1')
unlock lockCount: Long('0')
locked for 1.4s
73M
snapshot copy (standalone): 500000 docs, total 499999993
still has replset config: 1
```

The copy started as a standalone server with all data. Notes:

- **Copy the entire `dbpath`**, including `WiredTiger*`, `journal/`, and every `collection-*`/`index-*` file. A
  partial copy is unusable.
- While locked, the secondary falls behind; keep it short, and never lock the primary.
- The copy still contains the replica set config in `local`. To use it as a seed for a new member that's fine; to
  use it as an independent server, start it without `--replSet` (as here).
- **Filesystem or volume snapshots** (LVM, ZFS, cloud disk snapshots) are the production form of this technique.
  If the journal is on the same volume, an atomic snapshot is consistent even without `fsyncLock` because
  WiredTiger recovers from its journal at startup, just as after a crash.

## Choosing a strategy

| Method | Good for | Weak at |
|--------|----------|---------|
| `mongodump --oplog` | Small-to-medium replica sets; logical copies portable across versions | Slow for large data; restore rebuilds every index; not consistent for sharded clusters |
| Volume snapshots | Large data; fast restore | Tied to storage; must cover the whole dbpath atomically |
| Oplog archiving + base backup | Point-in-time recovery to any second | Must keep the oplog archive continuous — a gap breaks the chain |
| Managed backups (Atlas, Ops Manager) | Sharded clusters, scheduled PITR | Not available in a plain Community install; not demonstrated here |

Whatever you use, the rule is the same as for every database: **a backup you haven't restored is a hope, not a
backup**. Schedule restore tests, measure how long they take (that's your real recovery time), and check
invariants like the bank total — not just that the restore "succeeded".

## How It Actually Works

`mongodump` is an ordinary client: it lists collections and reads each with a cursor, writing BSON to files and
index definitions to metadata JSON. With `--oplog` it first reads the newest oplog entry's timestamp, dumps
collections, then queries `local.oplog.rs` for everything after that starting timestamp and writes it to
`oplog.bson`. Each document in the dump reflects some moment between start and end. Because oplog entries are
idempotent (Lesson 01) — "set balance to 993", not "subtract 7" — replaying entries over documents that already
include some of those changes is harmless, and every document ends at the state as of the last replayed entry.
Point-in-time recovery is the same mechanism with more oplog and a stop point.

`mongorestore` inserts documents in batches, then builds indexes from the metadata (index builds dominate restore
time for large collections), then — with `--oplogReplay` — applies the entries with `applyOps`. A file copy skips
all of that: WiredTiger's files are the database. At startup it finds the last checkpoint, then replays its
journal to recover anything written after it, which is why a crash-consistent snapshot of the whole dbpath
restores cleanly.

## Common mistakes

- **Treating replicas as backups.** They replicate the mistake too.
- **Dumping a busy replica set without `--oplog`**, or restoring without `--oplogReplay`, producing states that
  never existed.
- **An oplog window shorter than your detection time**, so the operations you need to replay are already gone.
  Archive the oplog continuously if you need PITR.
- **Never testing restores**, or testing only that the tool exits successfully.
- **File copies of a running primary** without a lock or atomic snapshot.
- **Restoring over production** instead of into a scratch server and copying out what's needed.
- **Forgetting side effects** during the incident window — writes that failed or did nothing because data was
  missing.

## Exercise

1. Reproduce the invariant test: dump a busy `bank.accounts` without `--oplog` and with it, restore both, and
   compare totals. Make the collection large enough that the difference shows up.
2. Write a script `pitr.sh <dump_dir> <limit_ts> <scratch_port>` that performs steps 2 and 3 automatically. Test it
   by recovering from an accidental `updateMany({}, {$set: {balance: 0}})`.
3. Fix the writer so that a transfer involving a missing account aborts its transaction, rerun the incident, and
   show the total is exact after the fix.
4. Measure: how long do `mongodump` and `mongorestore` take for your 500,000 accounts, and how much of the restore
   is index building? Add two secondary indexes and measure again.
5. Write a nightly job outline that keeps a dump plus a continuous oplog archive, with a retention policy and a
   monthly automated restore test that checks an invariant.
