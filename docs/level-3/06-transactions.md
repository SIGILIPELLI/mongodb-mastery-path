---
description: "Multi-document ACID transactions with PyMongo: the callback API, snapshot isolation, write conflicts and retries, limits, and what they really cost."
---

# 06 · Multi-Document Transactions

Single-document writes are atomic (Level 1 · 06), and good modeling puts data that changes together in one
document (Level 2 · 01). But some operations genuinely span documents: moving money between two accounts,
inserting an order and decrementing stock, adding a review and updating the product's rating. Since 4.0
MongoDB supports multi-document **ACID transactions** on replica sets (and on sharded clusters since 4.2). This
lesson uses them correctly — which mostly means handling the errors they're designed to produce.

Transactions need a replica set or sharded cluster; the course lab is a single-node replica set, which is
enough. All examples are Python with PyMongo 4.18.

## A transfer

```python
from decimal import Decimal
from bson import Decimal128
from pymongo import MongoClient
from pymongo.read_concern import ReadConcern
from pymongo.write_concern import WriteConcern

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0")
db = client.bank
db.accounts.insert_many([{"_id": "ana", "balance": Decimal128("100.00")},
                         {"_id": "ben", "balance": Decimal128("50.00")}])

def transfer(session, src, dst, amount):
    amt = Decimal128(Decimal(amount))
    neg = Decimal128(-Decimal(amount))
    res = db.accounts.update_one({"_id": src, "balance": {"$gte": amt}}, {"$inc": {"balance": neg}}, session=session)
    if res.modified_count != 1:
        raise ValueError(f"insufficient funds in {src}")
    db.accounts.update_one({"_id": dst}, {"$inc": {"balance": amt}}, session=session)
    db.ledger.insert_one({"from": src, "to": dst, "amount": amt}, session=session)

opts = dict(read_concern=ReadConcern("snapshot"), write_concern=WriteConcern("majority"))
with client.start_session() as s:
    s.with_transaction(lambda s: transfer(s, "ana", "ben", "30.00"), **opts)
```

```text
after transfer 30: {'ana': '70.00', 'ben': '80.00'} ledger: 1
```

Three things make this a transaction:

1. A **session** (`client.start_session()`), which carries the transaction's identity.
2. **`with_transaction(callback)`**, which starts the transaction, runs your callback, and commits.
3. **`session=s` on every operation** inside the callback. An operation without it runs *outside* the transaction —
   no error, just silently not part of it. This is the single most common transaction bug.

If the callback raises, the transaction is aborted and every write in it disappears:

```text
rejected: insufficient funds in ben -> {'ana': '70.00', 'ben': '80.00'} ledger: 1
```

Ben's balance wasn't touched — the conditional `$gte` failed first — but if it *had* been decremented before the
error, the abort would have undone it, along with any ledger entry.

## Isolation: nobody sees your changes until commit

```python
s = client.start_session()
s.start_transaction(**opts)
db.accounts.update_one({"_id": "ana"}, {"$inc": {"balance": Decimal128("1000")}}, session=s)
print("inside txn sees:", db.accounts.find_one({"_id": "ana"}, session=s)["balance"],
      "| outside sees:", db.accounts.find_one({"_id": "ana"})["balance"])
s.abort_transaction()
```

```text
inside txn sees: 1070.00 | outside sees: 70.00
after abort: {'ana': '70.00', 'ben': '80.00'}
```

Inside the transaction, reads see its own writes; outside, the world sees the last committed state. With read
concern `"snapshot"`, every read in the transaction sees data as of one point in time — a consistent snapshot even
while other clients commit changes.

## Write conflicts are normal

Two open transactions update the same document:

```python
s1.start_transaction(**opts); s2.start_transaction(**opts)
db.accounts.update_one({"_id": "ben"}, {"$inc": {"balance": Decimal128("1")}}, session=s1)
db.accounts.update_one({"_id": "ben"}, {"$inc": {"balance": Decimal128("2")}}, session=s2)
```

```text
second txn: 112 WriteConflict labels: ['TransientTransactionError']
after conflict: {'ana': '70.00', 'ben': '81.00'}
```

MongoDB doesn't make the second transaction wait for the first: it fails it immediately with code 112
**WriteConflict**, labelled **`TransientTransactionError`**. The label means "the whole transaction may succeed if
you retry it from the beginning". Only the first transaction's +1 committed.

This is why you should use `with_transaction` rather than `start_transaction`/`commit_transaction` by hand: it
retries the callback on `TransientTransactionError`, and retries the commit on `UnknownTransactionCommitResult`
(the commit may or may not have happened — e.g. a network error), for up to 120 seconds.

## Under load: retries happen, invariants hold

Eight threads each make 100 random transfers between five accounts holding 1,000.00 each. The callback counts how
often it runs:

```python
def worker(seed, n):
    r = random.Random(seed)
    for _ in range(n):
        src, dst = r.sample(names, 2)
        with client.start_session() as s:
            s.with_transaction(lambda s: transfer(s, src, dst, f"{r.randint(1, 50)}.00"),
                               read_concern=ReadConcern("snapshot"), write_concern=WriteConcern("majority"))
```

(Here `transfer` returns `False` instead of raising when funds are insufficient, so a rejected transfer still commits
an empty transaction.)

```text
800 transfers in 1.8s, callback executions: 1292 (retries: 492)
ledger entries: 774 | total balance: 5000.00
```

With only five accounts, conflicts were constant: the callback ran 1,292 times for 800 transfers. Yet the total is
exactly 5,000.00 and the ledger has one entry per successful transfer (26 were rejected for insufficient funds) —
retried attempts left no trace. Two lessons:

- **Your callback may run more than once.** Everything in it must be safe to repeat: no sending emails, no
  incrementing in-memory counters you rely on, no calling external APIs. Do those after `with_transaction` returns.
- **Hot documents make transactions fight.** Five accounts shared by eight threads is a worst case; real contention
  is usually lower, but a single "global counter" document touched by every transaction serialises your whole system.

## What transactions cost

The same script then ran 300 transfers inside transactions and 300 by calling `transfer(None, ...)` — the same
three writes with no session:

```text
300 transfers: with transaction 2214 ms, without 5330 ms
```

The transactional version was *faster*. To see why, a second script compared single updates at two write
concerns with transactions of three updates:

```text
300 single updates, w=majority: 1722 ms
300 single updates, w=1: 78 ms
300 transactions of 3 updates (commit w=majority): 1990 ms
```

The surprise is that the transaction wasn't the expensive part. A single update with write concern `majority` took
about 5.7 ms because the server waits for the write to be journaled (and, on a real replica set, replicated) before
acknowledging; with `w: 1` the same updates took 0.26 ms each. A transaction of three updates waits for durability
**once**, at commit — so 300 transactions of three updates cost about the same as 300 single durable updates, and
the transfer benchmark ran faster with a transaction (2.2 s) than as three separate durable writes (5.3 s). Write concern is covered properly in Level 4 · 02; the point here is to measure before assuming that
transactions are slow.

The real costs are elsewhere: transactions hold WiredTiger snapshots open (pressure on the cache), they conflict and
retry under contention, and on sharded clusters a transaction touching several shards needs a two-phase commit.

## Limits and good practice

- **Keep them short.** By default a transaction that runs longer than `transactionLifetimeLimitSeconds` — 60 on this
  server — is aborted. Don't do slow work (HTTP calls, user prompts) inside one.
- **Keep them small.** Modifying thousands of documents in one transaction is possible but ties up cache and
  increases conflict probability. Batch large changes outside transactions, idempotently.
- **Most operations work inside transactions**, including creating collections and indexes on empty collections in
  recent versions, but not everything (e.g. writes to capped collections, some admin commands). Check the docs for
  your version if something errors.
- **Use read concern `snapshot` and write concern `majority`** for transactions whose results must survive a failover.
- **Prefer modeling over transactions.** If two pieces of data must always change together, ask first whether they
  belong in one document.

## How It Actually Works

A session is a server-side object identified by a logical session id (lsid). When a transaction starts, every
operation sent with that lsid and a transaction number is executed inside **one WiredTiger transaction**, reading
from a snapshot taken at the transaction's start. Writes are applied to the transaction's private view; WiredTiger's
multi-version concurrency control keeps other readers on committed versions.

Conflicts are detected **eagerly**: if a transaction tries to modify a document that another uncommitted transaction
has already modified (or that was committed after this transaction's snapshot), WiredTiger reports a write conflict
immediately, and MongoDB aborts the transaction with `WriteConflict` and the `TransientTransactionError` label
rather than blocking. That's optimistic concurrency: no deadlocks, but the application must retry — which the callback
API does for you.

At commit, all the transaction's writes are recorded together in the oplog (as a single `applyOps` entry, or a chain
of entries for large transactions) so secondaries apply them atomically too, and the commit is acknowledged according
to the write concern. On sharded clusters, a transaction touching several shards uses a coordinator and two-phase
commit to make every shard commit or none.

## Common mistakes

- **Forgetting `session=s`** on one operation, which then silently runs outside the transaction.
- **Non-idempotent side effects in the callback**, repeated on every retry.
- **Hand-rolled `start_transaction`/`commit` without retry logic** for `TransientTransactionError` and
  `UnknownTransactionCommitResult`.
- **Long transactions** that hit the 60-second limit or hold snapshots open under load.
- **Using transactions to compensate for a schema that splits data that always changes together.**
- **A hot document shared by every transaction** (a global counter), serialising all writes.

## Exercise

1. Implement `place_order(customer_id, items)` for the `shop` database: insert the order and decrement each product's
   `stock` (add a `stock` field first), all-or-nothing, rejecting the order if any product is out of stock. Test with
   an order containing one unavailable item and confirm nothing changed.
2. Run your `place_order` from 10 threads against a product with stock 5. Prove that exactly 5 orders succeed.
3. Revisit the Level 2 · 10 exercise: insert a review and update the product's `rating` and `recentReviews` in one
   transaction. What failure did the transaction remove?
4. Deliberately add a `time.sleep(70)` inside a callback and observe what error you get and what `with_transaction`
   does with it.
5. Measure your transfer throughput with 2, 5, 50 and 500 accounts (8 threads). Plot retries per transfer against
   the number of accounts.
