---
description: "Use MongoDB from PyMongo and the Node.js driver: one shared client, connection pools, BSON type mapping, timeouts, errors and retryable writes."
---

# 08 · Drivers: PyMongo & the Node.js Driver

The shell is for people. Applications talk to MongoDB through a **driver** — an official library for
their language. Drivers all follow the same specifications, so once you understand one, the others
differ mainly in syntax. This lesson uses the two most common: **PyMongo** for Python and the
**`mongodb`** package for Node.js. (The language basics are covered in the
[Python](https://sigilipelli.github.io/python-mastery-path/) and
[Node.js](https://sigilipelli.github.io/nodejs-mastery-path/) Mastery Paths.)

Versions used here: PyMongo 4.18.3 on Python 3.14, and the Node.js driver 7.7.0 on Node 26.

```bash
python3 -m venv venv && venv/bin/pip install pymongo
npm install mongodb
```

## The one rule: create one client and share it

A `MongoClient` is not a connection. It is a connection **pool** plus background threads that monitor
every server in the deployment. It is expensive to create and designed to be shared by your whole
process for its whole life.

To show the cost, this script does 200 reads, first creating a new client for each read (a mistake that
appears surprisingly often in web handlers), then with one shared client:

```python
import time
from pymongo import MongoClient

URI = "mongodb://127.0.0.1:27017/?replicaSet=rs0"

t = time.perf_counter()
for i in range(200):
    with MongoClient(URI) as c:
        c.drivers.items.find_one()
per_request = time.perf_counter() - t

shared = MongoClient(URI)
t = time.perf_counter()
for i in range(200):
    shared.drivers.items.find_one()
reused = time.perf_counter() - t
print(f"new client per operation: {per_request*1000:.0f} ms for 200 reads")
print(f"one shared client:        {reused*1000:.0f} ms for 200 reads")
```

```text
new client per operation: 307 ms for 200 reads
one shared client:        31 ms for 200 reads
```

Ten times slower on localhost, where a TCP connection costs almost nothing. Over a real network with
TLS and authentication, every new connection adds several round trips plus a password-hashing
handshake, and the gap becomes enormous — and the server has to set up and tear down a connection each
time. In a web framework, create the client at start-up (module level, an app factory, or a dependency
that returns the same instance) and close it at shutdown.

## Python: PyMongo

```python
from datetime import datetime, timezone
from decimal import Decimal
from bson import Decimal128
from pymongo import MongoClient

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0")
coll = client.drivers.items       # attribute access; client["drivers"]["items"] also works
coll.drop()

res = coll.insert_one({
    "name": "kettle",
    "price": Decimal128(Decimal("29.99")),
    "qty": 3,
    "weight_kg": 1.2,
    "created": datetime(2026, 10, 11, 9, 30, tzinfo=timezone.utc),
    "tags": ["kitchen", "electric"],
})
doc = coll.find_one({"_id": res.inserted_id})
for k, v in doc.items():
    print(f"{k:10} {type(v).__name__:12} {v!r}")
```

```text
_id        ObjectId     ObjectId('6acbc7ea5646ac91f52b6466')
name       str          'kettle'
price      Decimal128   Decimal128('29.99')
qty        int          3
weight_kg  float        1.2
created    datetime     datetime.datetime(2026, 10, 11, 9, 30)
tags       list         ['kitchen', 'electric']
```

Method names are snake_case (`insert_one`, `find_one`, `update_many`); filters and update documents
are Python dicts with exactly the same operators you used in the shell.

Two type details trip people up:

**Decimals.** Python's `decimal.Decimal` is not encoded automatically:

```text
InvalidDocument Invalid document: cannot encode object: Decimal('1.10'), of type: <class 'decimal.Decimal'
```

Wrap it in `bson.Decimal128` on the way in and call `.to_decimal()` on the way out (or register a
custom type codec once).

**Datetimes.** The document above was stored with a timezone-aware UTC datetime, but it came back
*naive* — `datetime(2026, 10, 11, 9, 30)` with no `tzinfo`. BSON dates are always UTC, and by default
PyMongo returns them naive. Ask for aware datetimes when you create the client:

```python
aware = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0", tz_aware=True)
print(aware.drivers.items.find_one()["created"])
```

```text
2026-10-11 09:30:00+00:00
```

Mixing naive and aware datetimes in one codebase is a reliable source of off-by-five-and-a-half-hours
bugs. Pick `tz_aware=True` and stick with it.

PyMongo is synchronous. For `asyncio` applications PyMongo also ships an async API
(`from pymongo import AsyncMongoClient`) with the same method names as coroutines; it replaces the
older third-party Motor library, which MongoDB has deprecated.

## Node.js: the mongodb package

```javascript
import { MongoClient, Decimal128, MongoServerError } from "mongodb";

const client = new MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0", { maxPoolSize: 10 });
let opened = 0;
client.on("connectionCreated", () => opened++);

const items = client.db("drivers").collection("items");
await items.deleteMany({});
await items.insertMany([
  { name: "kettle", price: Decimal128.fromString("29.99"), qty: 3 },
  { name: "toaster", price: Decimal128.fromString("45.00"), qty: 0 },
  { name: "mug", price: Decimal128.fromString("6.50"), qty: 40 },
]);

for await (const doc of items.find({ qty: { $gt: 0 } }).sort({ name: 1 }).project({ _id: 0 })) {
  console.log(doc.name, doc.price.toString(), typeof doc.qty);
}
```

```text
kettle 29.99 number
mug 6.50 number
```

Every operation returns a Promise. Cursors are async iterables, so `for await` streams results batch
by batch without loading them all into memory — prefer it to `toArray()` for anything large. Forget an
`await` and you get a Promise instead of a document, with no error:

```javascript
const forgot = items.findOne({ name: "mug" });
console.log("without await:", forgot instanceof Promise);
```

```text
without await: true
```

Unlike `mongosh`, the driver does not hide this for you.

## The pool in action

The `connectionCreated` event counted how many connections the client opened while 50 reads ran
concurrently:

```javascript
await Promise.all(Array.from({ length: 50 }, () => items.findOne({ name: "mug" })));
console.log("connections opened for 50 concurrent reads:", opened);
```

```text
connections opened for 50 concurrent reads: 10
```

The pool never exceeds `maxPoolSize` (10 here; the default is 100). The other 40 operations waited for
a free connection — each connection runs one operation at a time. Sizing the pool is a trade-off: too
small and requests queue in your app; too large across hundreds of app instances and the server spends
its memory on idle connections. Level 4 · 08 returns to this.

## Errors

Both drivers raise typed errors with the server's numeric code:

```text
DuplicateKeyError code 11000                 # PyMongo
true 11000                                   # Node: e instanceof MongoServerError, e.code
```

Handle by **class and code**, never by parsing the message. The other error you must expect is "no
server available". With the server stopped, PyMongo waits for `serverSelectionTimeoutMS` (default 30 s;
2 s here) and then raises:

```python
dead = MongoClient("mongodb://127.0.0.1:27999/", serverSelectionTimeoutMS=2000)
dead.admin.command("ping")
```

```text
gave up after 2.0s: 127.0.0.1:27999: [Errno 61] Connection refused (configured timeouts: socketTimeoutMS: 20000.0ms
```

Notice that creating the client did **not** fail. Clients connect lazily and in the background; the
first operation is where you find out. If your application should refuse to start without a database,
run a `ping` command at start-up.

## Retryable writes and reads

Connection strings default to `retryWrites=true` and `retryReads=true`. If a write fails with a network
error or because the primary stepped down mid-operation, the driver retries it **once** on the new
primary. The server makes this safe by recording each retryable write's transaction number, so a retry
of a write that actually succeeded is recognised and not applied twice. This covers single-document
writes and `insertMany`/`bulkWrite`; `updateMany` and `deleteMany` are not retryable, because the
server can't tell which documents a half-finished multi-update already touched.

## How It Actually Works

When a client is created, it parses the URI and starts a **monitor** per seed host. Each monitor
connects and sends `hello`, learns the server's role (primary, secondary, standalone, mongos), the
replica set's member list and limits like `maxBsonObjectSize`, then keeps polling (by default every
10 seconds, or streaming updates on modern servers) so the client always has a current picture of
the **topology**.

Each operation goes through **server selection**: given the operation type and read preference, pick a
suitable server from the current topology — for a write, the primary. If none is suitable (the primary
is down mid-election), the client waits and re-checks until `serverSelectionTimeoutMS`. Then it checks
out a connection from that server's pool (creating one, with TLS and auth handshakes, if none is idle
and the pool isn't full), encodes the command to BSON, sends it as an `OP_MSG` wire-protocol message,
waits for the reply, decodes it, and returns the connection to the pool.

Everything that makes MongoDB "highly available" from the application's point of view — following a new
primary after failover, retrying a write once, reading from secondaries — happens in this layer, which
is why using a recent official driver and one long-lived client matters more than most tuning flags.

## Common mistakes

- **A new `MongoClient` per request or per function call.** Create one per process.
- **Forgetting `await`** in Node — no error, just a Promise where you expected data.
- **Naive datetimes in Python.** Use `tz_aware=True` and timezone-aware values.
- **Passing `decimal.Decimal` to PyMongo** without `Decimal128`.
- **`toArray()` / `list(cursor)` on unbounded queries.** Stream with iteration, or add a limit.
- **Discovering the database is unreachable on the first user request** — ping at start-up.
- **Catching errors by message text** instead of class and code.
- **Forking after creating a client** (e.g. pre-fork servers like Gunicorn): create the client in each
  worker after the fork; PyMongo warns if a client is used across a fork.

## Exercise

1. Write a Python module `db.py` that exposes one shared client (with `tz_aware=True`) and a
   `get_collection(name)` function. Use it from two other modules and confirm only one client is
   created.
2. Reproduce the 200-reads timing on your machine with both drivers. Then repeat it against a server
   on another machine (or a container) and compare the gap.
3. In Node, set `maxPoolSize: 2`, subscribe to the `connectionCheckOutStarted` and
   `connectionCheckedOut` events, and run 20 concurrent `find().toArray()` calls over a collection of
   100,000 documents. Log the time between "started" and "checked out" to watch operations queue for
   a connection, then raise the pool size and compare.
4. Insert a document with a Python `Decimal` price using `Decimal128`, read it back in Node, and print
   the price as a string. Then read a Node `Decimal128` back in Python and convert it to `decimal.Decimal`.
5. Make your Python app fail fast at start-up with a clear message when the database is unreachable.
