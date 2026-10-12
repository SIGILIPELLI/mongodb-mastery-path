---
description: "Build a three-node MongoDB replica set, read the oplog, kill the primary mid-write, and force a real rollback to see what w:1 writes can lose."
---

# 01 · Replica Sets: Elections, the Oplog & Failover

Every lab so far has used a single-node replica set — enough to unlock transactions and change streams, but
with none of the reason replica sets exist. A real replica set is several `mongod` processes holding copies of
the same data: one **primary** accepts writes, the **secondaries** copy them, and if the primary disappears the
survivors elect a new one within seconds. This lesson builds a three-node set on one machine, reads the log
that carries every change between nodes, and then breaks it on purpose: a clean step-down, a hard kill in the
middle of a stream of writes, and a failure that makes acknowledged writes vanish.

Everything here ran on MongoDB 9.0.2 with PyMongo 4.18.3 on a laptop. Election timings depend on the defaults
shown below, not on hardware, so yours should be close.

## Three nodes on one machine

Each member is an ordinary `mongod` with its own port, its own data directory and the same `--replSet` name:

```bash
mkdir -p n1 n2 n3
for p in 1 2 3; do
  mongod --replSet rs4 --port 2730$p --dbpath n$p --bind_ip 127.0.0.1 \
         --wiredTigerCacheSizeGB 0.25 --oplogSize 64 --logpath n$p.log &
done
```

`--wiredTigerCacheSizeGB 0.25` stops three servers from each claiming half the laptop's memory (Lesson 08
explains the default), and `--oplogSize 64` keeps the replication log small for a lab. The processes start but
refuse writes until the set is initiated — once, on one member:

```javascript
rs.initiate({_id: "rs4", members: [
  {_id: 0, host: "127.0.0.1:27301"},
  {_id: 1, host: "127.0.0.1:27302"},
  {_id: 2, host: "127.0.0.1:27303"}]})
```

```text
{ ok: 1, ... }
127.0.0.1:27301 SECONDARY
127.0.0.1:27302 SECONDARY
127.0.0.1:27303 PRIMARY
```

All three started as secondaries; within a few seconds one called an election and won. Which one wins is not
something to rely on — here it was the third. The host names in the config are what members use to reach each
other **and what drivers are told to connect to**, so in a real deployment they must be names every client can
resolve. `127.0.0.1` only works because everything is on one machine.

Applications connect with a seed list and the set name, never to one host:

```text
mongodb://127.0.0.1:27301,127.0.0.1:27302,127.0.0.1:27303/?replicaSet=rs4
```

The driver contacts any seed, asks it for the current topology (`hello`), and from then on sends writes to
whoever is primary and tracks changes itself. Any one reachable seed is enough.

## The settings that decide failover speed

```javascript
const c = rs.conf()
```

```text
{
  protocolVersion: Long('1'),
  electionTimeoutMillis: 10000,
  heartbeatIntervalMillis: 2000,
  catchUpTimeoutMillis: -1,
  chainingAllowed: true
}
```

Members send each other heartbeats every 2 seconds. A secondary that hasn't heard from a primary for
`electionTimeoutMillis` — 10 seconds — calls an election. That number is the floor for unplanned failover:
lowering it makes failover faster and false alarms (a GC pause, a busy network, a laptop going to sleep) more
likely. `chainingAllowed` lets a secondary copy from another secondary instead of the primary.

## The oplog: replication is replaying a log

The primary records every write in a capped collection, `local.oplog.rs`; secondaries copy entries from it and
apply them to their own data. A few writes:

```javascript
const app = db.getSiblingDB("app")
app.stock.insertMany([{_id: "pen", qty: 10}, {_id: "ink", qty: 4}, {_id: "pad", qty: 7}])
app.stock.updateMany({}, {$inc: {qty: -1}})
app.stock.deleteOne({_id: "pad"})
db.getSiblingDB("local").oplog.rs.find({ts: {$gte: start}}).sort({$natural: 1})
```

```text
c app.$cmd    {"create":"stock","idIndex":{"v":2,"key":{"_id":1},"name":"_id_"}}
c admin.$cmd  {"applyOps":[{"op":"i","ns":"app.stock",...,"o":{"_id":"pen","qty":10},...},{"op":"i",...
u app.stock   {"$v":2,"diff":{"u":{"qty":9}}}
u app.stock   {"$v":2,"diff":{"u":{"qty":3}}}
u app.stock   {"$v":2,"diff":{"u":{"qty":6}}}
d app.stock   {"_id":"pad"}
```

Four details explain most of replication's behaviour:

- **One `updateMany` became three entries**, one per document. The oplog is per-document, so a multi-document
  update of a million documents is a million oplog entries — and a million things for every secondary to apply.
- **`$inc: {qty: -1}` was recorded as "set qty to 9"**. Oplog entries are rewritten into an *idempotent* form:
  applying the same entry twice gives the same result. That property lets a secondary that crashes mid-batch
  simply re-apply from a safe point.
- **The batched insert became one `applyOps` command** containing all three inserts, applied atomically on
  secondaries.
- The collection's own creation is an entry too. Index builds, drops and renames all replicate as commands.

A full entry also carries `ts` (a timestamp plus counter that orders every entry in the set), `t` (the election
term), `wall` (wall-clock time), and for retryable writes the session id and `txnNumber`:

```text
{ lsid: { id: UUID('6a4d4626-...') }, txnNumber: Long('2'), op: 'd', ns: 'app.stock',
  ui: UUID('768e247c-...'), o: { _id: 'pad' }, stmtId: 0,
  ts: Timestamp({ t: 1791829347, i: 6 }), t: Long('1'), v: Long('2'),
  wall: ISODate('2026-10-12T18:22:27.209Z') }
```

The oplog is capped by size, so it covers a window of time that depends on write volume. `rs.printReplicationInfo()`
reports it; a secondary that falls further behind than that window cannot catch up and must be resynced from
scratch. Size the oplog for hours or days of writes, not minutes.

## Failure 1: a planned step-down

A Python writer inserts a numbered document every 20 ms, timing each insert, while the primary is asked to step
down — what happens during maintenance or a rolling upgrade:

```python
client = MongoClient(URI, serverSelectionTimeoutMS=30000)
...
MongoClient(f"mongodb://127.0.0.1:{old}/?directConnection=true").admin.command("replSetStepDown", 30)
```

```text
stepping down primary on port 27301 with replSetStepDown at t=2.00s
inserts acknowledged: 208, errors raised to the app: 0 []
longest single insert: 0.03s (started at t=0.00s)
new primary: 27302
documents stored: 208 | gaps in sequence: 0
```

Nothing visible happened. A stepping-down primary waits for a secondary to catch up, then that secondary wins
an election immediately — no 10-second timeout, because nobody had to *notice* a failure. The slowest insert
was the very first one (opening connections), not one during the switch.

## Failure 2: kill -9 the primary

Same writer; this time the primary process is killed with `SIGKILL` — a crash, a power cut, a kernel panic:

```text
killing primary on port 27303 (pid 86990) with SIGKILL at t=2.00s
inserts acknowledged: 197, errors raised to the app: 0 []
longest single insert: 10.33s (started at t=2.00s)
new primary: 27301
documents stored: 197 | gaps in sequence: 0
```

The application saw **no error**, but one insert took 10.33 seconds. The new primary's log shows where the time
went:

```text
23:52:52.360  Starting an election, since we've seen no PRIMARY in election timeout period  10000
23:52:52.374  Election succeeded, assuming primary role
23:52:52.387  Transition to primary complete; database writes are now permitted
```

Ten seconds of *noticing*, then 27 ms of electing. Meanwhile the driver kept the in-flight insert waiting for a
new primary (up to `serverSelectionTimeoutMS`) and then **retried it once** — retryable writes are on by default
in modern drivers. The retry is safe because the write carries a session id and transaction number (visible in
the oplog entry above): if the old primary had already applied it before dying, the new primary would
recognise the number and not apply it twice.

The same test with `retryWrites=false` in the connection string:

```text
killing primary on port 27302 (pid 86988) with SIGKILL at t=2.00s
inserts acknowledged: 172, errors raised to the app: 1 ['AutoReconnect']
longest single insert: 11.17s (started at t=2.04s)
new primary: 27303
documents stored: 172 | gaps in sequence: 1
```

Now the in-flight insert surfaced as an `AutoReconnect` error and left a gap. Worse, the application *cannot know*
whether that write happened — here it didn't, but a write that reached the primary just before the kill would have
replicated and been stored. Leave retryable writes on, and make sure anything you retry by hand is idempotent.

A restarted old primary simply rejoins as a secondary and copies what it missed:

```text
127.0.0.1:27301 PRIMARY    2026-10-12T18:22:57.000Z
127.0.0.1:27302 SECONDARY  2026-10-12T18:22:57.000Z
127.0.0.1:27303 SECONDARY  2026-10-12T18:22:57.000Z
```

## Failure 3: acknowledged writes that disappear

What if the primary accepts writes that never reach a secondary, then dies? To force it: kill both secondaries,
write with `w: 1` (acknowledge as soon as the primary has it), kill the primary, and bring the two secondaries
back first so they elect a leader from the data they have.

```text
majority write ok                                  # _id 0, written while all three were up
both secondaries killed
w:1 acknowledged: 5 | still primary: true          # _id 1..5, acknowledged to the client
old primary killed
127.0.0.1:27301 PRIMARY
127.0.0.1:27302 (not reachable/healthy)
127.0.0.1:27303 SECONDARY
orders on new primary: [0]
new write with _id 1 ok                            # a different document, same _id
```

Five writes the client was told succeeded are not on the new primary, and an application has already reused one
of their `_id`s. When the old primary restarts, it finds its history has diverged from the set's and **rolls
back**:

```text
Transition to ROLLBACK
Finding common point
...
Preparing to write deleted documents to a rollback file
Rolling back to the stable timestamp
Operations reverted by rollback
Rollback complete
Rollback summary
Transition to SECONDARY
```

```text
orders on old primary now: [{"_id":0,"note":"before"},{"_id":1,"note":"written after failover"}]
n2/rollback/3e0fa3a6-82f5-4131-8eae-ac5f3a0d44ee/removed.2026-10-12T18-25-30.1.bson
```

The five documents are gone from the database; the only trace is a BSON file in the data directory:

```bash
bsondump --quiet n2/rollback/3e0fa3a6-*/removed*.bson
```

```text
{"_id":{"$numberInt":"2"},"note":"w:1 only"}
{"_id":{"$numberInt":"3"},"note":"w:1 only"}
{"_id":{"$numberInt":"5"},"note":"w:1 only"}
{"_id":{"$numberInt":"4"},"note":"w:1 only"}
{"_id":{"$numberInt":"1"},"note":"w:1 only"}
```

Nobody is alerted; someone has to notice the file and decide what to do with it. The fix is not "watch for
rollback files" but **write concern `majority`**: a write acknowledged by a majority is on a node that will win any
future election, and cannot be rolled back. That is the next lesson's subject, and the reason `majority` is the
default write concern since 5.0. This demonstration had to override it explicitly.

!!! warning "A first attempt that didn't roll back — and why it matters"
    The first version of this experiment *paused* the secondaries with `SIGSTOP` instead of killing them. The
    "w:1 only" writes — and even a `majority` write that failed with `WriteConcernTimeout` — all survived on the
    new primary: the secondaries' replication cursors had already been sent the data, which sat in socket
    buffers until the processes resumed. Two lessons: failure tests must actually cut the path, and a
    **write-concern timeout does not mean the write failed** — only that confirmation didn't arrive in time.

## Priorities, arbiters and hidden members

Members can be weighted. Giving one node `priority: 2` makes it take over whenever it's healthy and caught up:

```javascript
c.members[2].priority = 2; rs.reconfig(c)
```

```text
Starting an election for a priority takeover
127.0.0.1:27303 PRIMARY
```

Other member options, briefly:

- **`priority: 0`** — never becomes primary (a node in a distant data centre, or a reporting node).
- **`hidden: true`** (requires priority 0) — invisible to drivers; used for backups or analytics.
- **`secondaryDelaySecs`** — a deliberately lagging copy, a safety net against a bad `deleteMany` (hidden, priority 0).
- **`votes: 0`** — doesn't vote. A set has at most 7 voting members (and 50 total).
- **Arbiters** hold no data and only vote. They make a two-data-node set "elect-able", but a primary-plus-arbiter
  set with one secondary down can't acknowledge `majority` writes. Prefer three data-bearing members.

## How It Actually Works

MongoDB's replication protocol (protocol version 1) is a Raft-style consensus design. Time is divided into
**terms**; each election increments the term, and each member votes at most once per term, for a candidate
whose oplog is at least as up to date as its own. A candidate needs votes from a strict majority of voting
members, which is why two halves of a partitioned set can never both elect a primary, and why a write
acknowledged by a majority survives: any future majority must include at least one member that has it, and the
up-to-date-log voting rule guarantees the winner has it too.

Secondaries **pull**: each runs an oplog fetcher that keeps a tailable, awaitData cursor open on its sync source's
oplog, receives batches, and hands them to an applier that applies them in parallel (grouped so writes to the same
document stay ordered). Each secondary reports how far it has applied back upstream; that's how the primary knows
when a write has reached a majority and moves the **majority commit point** forward.

Rollback is the consequence of a primary accepting writes beyond the commit point. When a former primary rejoins,
it compares its oplog with its sync source's, walks back to the last common entry, then uses WiredTiger's
**stable timestamp** — a point no later than the majority commit point, which the storage engine can restore to
directly — to discard everything after it, writing the affected documents to rollback files first. The sleep
anecdote from building this lab is the same machinery seen from the other side: an earlier three-node lab left
running on a laptop overnight had been through 104 elections by morning, because every sleep froze heartbeats for
longer than 10 seconds.

## Common mistakes

- **Connecting to a single host** instead of a seed list with `replicaSet=` — writes fail as soon as that host is
  no longer primary.
- **Two data nodes and no third voter.** Losing either node loses the majority, so the survivor can't become
  primary.
- **Member host names clients can't resolve** (container-internal names, `localhost` in a multi-machine set).
- **An oplog sized for minutes**, so a secondary down for maintenance can't catch up and needs a full resync.
- **Turning off retryable writes**, or retrying non-idempotent writes by hand.
- **Using `w: 1` for data you can't afford to lose**, and never checking for rollback files.
- **Treating `WriteConcernTimeout` as failure** and re-inserting, creating duplicates.

## Exercise

1. Build the three-node set. Run the kill test three times, killing a different member each time (including a
   secondary). How long is the stall when a *secondary* dies?
2. Set `electionTimeoutMillis` to 2000 with `rs.reconfig` and repeat the kill test. Then pause the primary with
   `kill -STOP` for 3 seconds and resume it. What happened, and what does that tell you about low timeouts?
3. Add a fourth member with `priority: 0, hidden: true, secondaryDelaySecs: 120`. Run `deleteMany({})` on a
   collection and recover its documents from the delayed member within two minutes.
4. Reproduce the rollback, then write a short script that restores the documents from the rollback file into an
   `orders_recovered` collection for human review.
5. Insert one million small documents with `insertMany`, then run one `updateMany({}, {$set: {flag: true}})`.
   Check how many oplog entries it produced and how long the secondaries took to catch up
   (`rs.printSecondaryReplicationInfo()`).
