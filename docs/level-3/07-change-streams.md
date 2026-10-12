---
description: "MongoDB change streams: event shapes, filtering, updateLookup vs pre-images, resume tokens for crash-safe consumers, and invalidation."
---

# 07 · Change Streams

Applications constantly need to react when data changes: push a notification when a ticket is assigned,
refresh a cache when a product changes, update a search index, refresh the rollup from lesson 05 only for days
that changed, stream changes to another system. Polling (`find` every few seconds for `updatedAt > last`) is
slow, wasteful and misses deletes. **Change streams** give you an ordered, resumable feed of every insert,
update, replace and delete on a collection, a database or the whole deployment.

They require a replica set or sharded cluster, because they're built on the oplog (Level 4 · 01). Examples use
PyMongo against the lab replica set.

## Watching a collection

```python
import threading, time
from pymongo import MongoClient

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0")
db = client.cs
db.drop_collection("tickets")
db.create_collection("tickets")

def writer():
    time.sleep(0.5)
    db.tickets.insert_one({"_id": 1, "title": "Printer on fire", "status": "open", "priority": "high"})
    db.tickets.insert_one({"_id": 2, "title": "Mouse squeaks", "status": "open", "priority": "low"})
    db.tickets.update_one({"_id": 1}, {"$set": {"status": "in_progress"}, "$inc": {"updates": 1}})
    db.tickets.update_one({"_id": 2}, {"$set": {"status": "closed"}})
    db.tickets.delete_one({"_id": 2})

threading.Thread(target=writer).start()
pipeline = [{"$match": {"operationType": {"$in": ["insert", "update", "delete"]}}}]
with db.tickets.watch(pipeline, full_document="updateLookup", max_await_time_ms=500) as stream:
    seen, t0 = 0, time.time()
    while seen < 5 and time.time() - t0 < 10:
        ch = stream.try_next()
        if ch is None:
            continue
        seen += 1
        desc = ch.get("updateDescription", {})
        print(f"{ch['operationType']:7} _id={ch['documentKey']['_id']}"
              f" updated={desc.get('updatedFields')} full={ch.get('fullDocument')}")
```

```text
insert  _id=1 updated=None full={'_id': 1, 'title': 'Printer on fire', 'status': 'open', 'priority': 'high'}
insert  _id=2 updated=None full={'_id': 2, 'title': 'Mouse squeaks', 'status': 'open', 'priority': 'low'}
update  _id=1 updated={'status': 'in_progress', 'updates': 1} full={'_id': 1, 'title': 'Printer on fire', 'status': 'in_progress', 'priority': 'high', 'updates': 1}
update  _id=2 updated={'status': 'closed'} full={'_id': 2, 'title': 'Mouse squeaks', 'status': 'closed', 'priority': 'low'}
delete  _id=2 updated=None full=None
```

Each change event is a document with:

| Field | Contents |
|-------|----------|
| `_id` | the **resume token** — a position in the stream |
| `operationType` | `insert`, `update`, `replace`, `delete`, plus `drop`, `rename`, `invalidate` and others |
| `ns` | database and collection |
| `documentKey` | the changed document's `_id` (and shard key on sharded clusters) |
| `fullDocument` | the new document for inserts/replaces; for updates only if requested |
| `updateDescription` | for updates: `updatedFields`, `removedFields`, `truncatedArrays` |
| `clusterTime` | when the change happened |

`watch()` accepts an aggregation pipeline of filtering and reshaping stages (`$match`, `$project`, `$set`, …), applied
**on the server**, so uninteresting events never cross the network. You can watch a collection, `db.watch()` for a
whole database, or `client.watch()` for everything.

## updateLookup is "now", not "then"

By default an update event carries only the delta. `full_document="updateLookup"` makes the server look up the
document's **current** version when it sends the event. That's convenient, but note the timing: the lookup happens
when the event is delivered, not when the update happened. In the run above, the update to ticket 2 came back with
the full document because the lookup happened before the delete; if the consumer had been slower, `fullDocument`
for that update would have been `None` (the document no longer existed) — and for a document updated twice in quick
succession, both events can show the second state. Use `updatedFields` when you need what changed in *this* event.

## Pre-images: what did it look like before?

For deletes there's no current document at all. To see the state *before* a change, enable pre- and post-images on
the collection and ask for them:

```python
db.command("collMod", "events", changeStreamPreAndPostImages={"enabled": True})
db.events.update_one({"_id": 5}, {"$set": {"note": "about to go"}})
with db.events.watch([{"$match": {"operationType": "delete"}}],
                     full_document_before_change="required") as stream:
    db.events.delete_one({"_id": 5})
    ch = stream.next()
    print("deleted", ch["documentKey"], "before:", ch["fullDocumentBeforeChange"])
```

```text
deleted {'_id': 5} before: {'_id': 5, 'note': 'about to go'}
```

The server stores a copy of each document's previous version in a system collection when it changes, so this costs
extra writes and storage; enable it only on collections that need it, and configure how long pre-images are kept
(`changeStreamOptions.preAndPostImages.expireAfterSeconds`). `"required"` fails the stream if a pre-image is
missing; `"whenAvailable"` returns null instead.

## Resuming after a crash

A consumer that restarts must continue exactly where it stopped — not miss events written while it was down, and
not reprocess everything. Save the **resume token** after processing each event, and pass it to `watch()` on restart:

```python
# 1. A consumer processes two events, saves its resume token after each, then "crashes".
with db.events.watch() as stream:
    db.events.insert_many([{"_id": 1}, {"_id": 2}])
    for _ in range(2):
        ch = stream.next()
        print("consumer A processed", ch["documentKey"]["_id"])
        with open(TOKEN_FILE, "w") as f:
            json.dump(ch["_id"], f)

# 2. Writes happen while no consumer is running.
db.events.insert_many([{"_id": 3}, {"_id": 4}, {"_id": 5}])

# 3. A new consumer resumes from the saved token and gets exactly the missed events.
with open(TOKEN_FILE) as f:
    token = json.load(f)
with db.events.watch(resume_after=token, max_await_time_ms=300) as stream:
    while (ch := stream.try_next()) is not None:
        print("consumer B resumed and got", ch["documentKey"]["_id"])
```

```text
consumer A processed 1
consumer A processed 2
consumer B resumed and got 3
consumer B resumed and got 4
consumer B resumed and got 5
```

Nothing lost, nothing repeated. In production, store the token in the same place and transaction as the effect of
processing the event (e.g. in the database you're updating), so "processed" and "token saved" can't diverge. If you
save the token *before* processing, a crash loses an event; *after*, a crash repeats one — so make processing
idempotent and save after.

Drivers also resume automatically after transient errors such as a primary failover, using the last token they
received. Your code only needs the saved token across process restarts.

### The oplog window

A resume token is a position in the **oplog**, which is a capped collection (Level 2 · 09) of limited size. If a
consumer is down longer than the oplog covers, its position has been overwritten and resuming fails with a
`ChangeStreamHistoryLost` error. Then the only option is a full resync (re-read the collection) followed by a new
stream. Monitor how far behind consumers are and size the oplog accordingly (Level 4 · 01).

## Invalidation

Some events end a stream permanently:

```python
with db.events.watch() as stream:
    db.drop_collection("events")
    for _ in range(2):
        ch = stream.next()
        print("event:", ch["operationType"])
    print("stream alive after invalidate:", stream.alive)
```

```text
event: drop
event: invalidate
stream alive after invalidate: False
```

Dropping or renaming a watched collection (or dropping its database) produces an `invalidate` event and closes the
stream. To continue past it, open a new stream with `start_after=<invalidate event's token>` (not `resume_after`,
which refuses to resume from an invalidate). Database- and deployment-level streams don't invalidate on collection
drops — another reason to watch at a higher level and filter by `ns`.

## Design notes

- **Change streams deliver events in commit order per shard, at least once per resumed position** — design consumers
  to be idempotent.
- **Each stream is a cursor on the server.** Hundreds of streams with complex pipelines cost real CPU; prefer a few
  database-level streams fanned out in your application over one stream per user.
- **A stream sees committed data only**, and with the default majority read concern, only data that's durable — a
  change that's later rolled back during a failover is never delivered.
- **Transactions appear as their individual operations**, each carrying the transaction's `lsid` and `txnNumber`, all
  delivered after the transaction commits.

## How It Actually Works

Every write on a replica set is recorded in the **oplog** (`local.oplog.rs`) on the primary and copied to
secondaries. A change stream is an aggregation over the oplog: the server opens a tailable cursor on it, filters
entries to the watched namespace, transforms each raw oplog entry into a change-event document (an update's diff
becomes `updateDescription`, for instance), applies your pipeline, and delivers results via `getMore` — waiting up to
`maxAwaitTimeMS` for new entries, like the tailable cursors of Level 2 · 09.

The resume token encodes the cluster time and position of the event in the oplog. Resuming means reopening the
oplog scan at that position, which is why it only works while that position still exists. Because streams read with
majority read concern, they only advance over entries replicated to a majority, so events never "un-happen". On a
sharded cluster, `mongos` opens a stream on every shard and merges them in cluster-time order before delivering them
to you. `fullDocument: "updateLookup"` is implemented as an extra `find` by `_id` per update event, which is why it
returns the document as it is at lookup time, and why it adds read load.

## Common mistakes

- **Saving the resume token before processing**, losing events on crash.
- **Non-idempotent consumers** that break when an event is delivered twice after a restart.
- **Trusting `updateLookup` to show the state at event time.**
- **Enabling pre-images everywhere** "just in case", doubling write load.
- **Ignoring `invalidate`** and leaving a consumer silently dead after a collection rename.
- **No monitoring of consumer lag** until the oplog window is exceeded.

## Exercise

1. Write a consumer that maintains a `ticket_counts` document (`{open: n, in_progress: n, closed: n}`) from a change
   stream on `tickets`, handling inserts, status updates and deletes (enable pre-images so deletes know which status to
   decrement). Make it resumable with a saved token.
2. Kill the consumer with Ctrl+C in the middle of a burst of 1,000 writes, restart it, and verify the counts match a
   fresh aggregation.
3. Use a change stream to drive the incremental rollup from lesson 05: collect the days touched by order changes for
   one second at a time, then re-run the rollup for just those days.
4. Open a database-level stream with a `$match` on `ns.coll` and confirm it survives dropping one collection.
5. Measure how long a resume token stays valid on your lab: write to another collection in a loop and check
   `db.getReplicationInfo()` for the oplog's time window.
