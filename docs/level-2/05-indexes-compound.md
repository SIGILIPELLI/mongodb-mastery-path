---
description: "How MongoDB B-tree indexes turn scans into lookups, how compound indexes and prefixes work, and the Equality-Sort-Range rule measured on 500,000 orders."
---

# 05 · How Indexes Work & Compound Index Design

Everything in Level 1 ran as a **collection scan**: to answer a query, the server read every document.
With ten books that's instant. With half a million orders it's not — and with fifty million, under load,
it's an outage. Indexes are the single biggest performance lever in MongoDB, and compound index design is
the skill that separates a database that copes from one that doesn't.

## A test collection

The script below creates 500,000 orders with a seeded random generator, so you get the same data every
time (`customerId` 0–49,999, five statuses with about 70% `delivered` and 5% `cancelled`, `placedAt`
spread across 2025–2026, `total` up to 500.00):

```javascript
const d = db.getSiblingDB("idx"); d.orders.drop();
let seed = 42; const rnd = () => { seed |= 0; seed = seed + 0x6D2B79F5 | 0;
  let t = Math.imul(seed ^ seed >>> 15, 1 | seed); t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
  return ((t ^ t >>> 14) >>> 0) / 4294967296; };
const statuses = ["placed","paid","shipped","delivered","cancelled"], weights = [0.05,0.1,0.1,0.7,0.05];
const pick = () => { let r = rnd(), a = 0; for (let i = 0; i < 5; i++) { a += weights[i]; if (r < a) return statuses[i]; } return "delivered"; };
const cities = ["Pune","Mumbai","Delhi","Chennai","Hyderabad","Bengaluru","Kolkata","Jaipur"];
const start = new Date("2025-01-01").getTime(), span = 640 * 86400000;
let batch = [];
for (let i = 0; i < 500000; i++) {
  batch.push({ customerId: Math.floor(rnd() * 50000), status: pick(), city: cities[Math.floor(rnd() * 8)],
    placedAt: new Date(start + Math.floor(rnd() * span)), total: Math.round(rnd() * 50000) / 100,
    items: Math.floor(rnd() * 5) + 1 });
  if (batch.length === 10000) { d.orders.insertMany(batch); batch = []; }
}
```

(The first version of this generator used a textbook linear congruential formula with a power-of-two
modulus. It produced only 5,784 distinct customer ids out of 50,000 — its low bits repeat in short cycles.
If your synthetic data looks oddly clumpy, check the generator before blaming the database.)

To compare plans, a small helper prints the plan's stages, how many **index keys** and **documents** the
server examined, and the time:

```javascript
const run = (label, q, sort, hint) => {
  let c = o.find(q).sort(sort || {}).limit(20); if (hint) c = c.hint(hint);
  const e = c.explain("executionStats"), s = e.executionStats;
  const stages = []; let p = e.queryPlanner.winningPlan; while (p) { stages.push(p.stage); p = p.inputStage; }
  print(`${label.padEnd(34)} ${stages.join("<-").padEnd(36)} keys=${s.totalKeysExamined} docs=${s.totalDocsExamined} ms=${s.executionTimeMillis}`);
};
```

Lesson 07 explains `explain()` in detail; for now, **fewer keys and documents examined is better**, and
the ideal is examining exactly as many as you return.

## The first index

"Orders of customer 12345":

```text
customer, no index                 LIMIT<-COLLSCAN                      keys=     0 docs=500000 ms=134
```

```javascript
db.orders.createIndex({ customerId: 1 })
```

```text
customer, {customerId:1}           LIMIT<-FETCH<-IXSCAN                 keys=    11 docs=    11 ms=5
```

From reading 500,000 documents to reading 11 — the customer's 11 orders. The time difference (134 ms → 5
ms) understates it: the collection scan's cost grows with the collection, the index lookup's cost grows
only with the number of results. Building the index took under half a second here (a single-field index on
`city` built in 440 ms on this collection); on large collections builds take longer but don't block reads
and writes for their duration.

## Compound indexes and the order of fields

A compound index sorts entries by its first field, then by the second within equal first values, and so on
— like a phone book sorted by surname, then first name. That makes field order decisive. Take a realistic
query: *the 20 largest cancelled orders placed between March and June 2026*.

```javascript
const q2 = { status: "cancelled", placedAt: { $gte: new Date("2026-03-01"), $lt: new Date("2026-06-01") } };
db.orders.find(q2).sort({ total: -1 }).limit(20)
```

Three fields: `status` is an **E**quality condition, `total` is the **S**ort, `placedAt` is a **R**ange.
Three candidate indexes with the same fields in different orders, each forced with `hint()`:

```text
R,E,S {placedAt,status,total}      FETCH<-SORT<-IXSCAN                  keys= 71514 docs=    20 ms=64
E,R,S {status,placedAt,total}      FETCH<-SORT<-IXSCAN                  keys=  3480 docs=    20 ms=3
E,S,R {status,total,placedAt}      LIMIT<-FETCH<-IXSCAN                 keys=   219 docs=    20 ms=0
```

- **Range first** (`placedAt, status, total`): the index scan covers *every* order in the three-month
  range — 71,514 keys, checking each one's status — then sorts the matches in memory (`SORT` stage).
- **Equality, range, sort**: jumps straight to `cancelled`, scans the 3,480 cancelled orders in the date
  range, then still has to **sort** them all in memory to find the top 20.
- **Equality, sort, range**: jumps to `cancelled`, walks the entries **already in `total` descending
  order**, checks each one's date, and stops after finding 20 in range. 219 keys, no `SORT` stage, no
  limit on how big the result set could be.

This is the **ESR rule**: in a compound index, put fields used for **E**quality first, then the fields you
**S**ort on, then fields with **R**ange conditions. Left to choose for itself with all three indexes
available, the planner picked the same one:

```text
planner's own choice               LIMIT<-FETCH<-IXSCAN                 keys=   219 docs=    20 ms=1
```

ESR is a rule of thumb, not a law. If the range is extremely selective (one minute out of two years) and
there's no limit, E-R-S can win because it examines fewer keys in total. Measure with your real data and
query shapes.

## Index prefixes

An index on `{a, b, c}` can serve queries on `{a}`, `{a, b}` and `{a, b, c}` — its **prefixes** — but not
on `{b}` or `{c}` alone, just as a phone book sorted by surname can't find everyone named "Priya":

```text
{status: "cancelled"}          → status_1_placedAt_1_total_-1
{placedAt: {$gte: ...}}        → placedAt_1_status_1_total_-1
{total: {$gt: 400}}            → COLLSCAN
```

No index starts with `total`, so the last query scans. Because prefixes are served by longer indexes, a
separate index on `{status: 1}` would be redundant next to `{status: 1, total: -1, placedAt: 1}` — every
extra index costs memory and slows every write.

## Sort direction

For a single-field index, direction doesn't matter: the index can be walked backwards. For compound
indexes, the **relative** directions matter: `{status: 1, total: -1}` serves `sort({status: 1, total: -1})`
and its exact reverse `sort({status: -1, total: 1})`, but not `sort({status: 1, total: 1})`. Match the
index to the sorts you actually run.

## What indexes cost

```text
{
  _id_: 12439552,
  'placedAt_1_status_1_total_-1': 17547264,
  'status_1_placedAt_1_total_-1': 11968512,
  'status_1_total_-1_placedAt_1': 9293824
}
```

Each of these is roughly 9–17 MB for 500,000 documents — in the same range as the whole `_id` index. (Sizes
differ partly because of how well each key order compresses: indexes that start with a low-cardinality
field like `status` get excellent prefix compression.) Every index:

- must be updated on every insert and delete, and on every update that touches an indexed field;
- competes for the WiredTiger cache — an index that doesn't fit in memory is read from disk;
- adds to replication and backup work.

A collection with fifteen indexes "just in case" is a slow-writing collection. Create indexes for the
queries you actually run, and remove ones nothing uses (lesson 08 shows how to find them with
`$indexStats`).

## Building indexes on a live system

```javascript
db.orders.createIndex({ customerId: 1, placedAt: -1 }, { name: "customer_recent" })
```

Since MongoDB 4.2 index builds hold an exclusive lock only briefly at the start and end; in between,
reads and writes continue. The build still consumes CPU, I/O and cache, so build big indexes off-peak. On a
replica set the build runs on all data-bearing members simultaneously and commits once the commit quorum
— by default, all voting data-bearing members — is ready (Level 4 · 09 measures one under load).
Set `name` explicitly for important indexes: the auto-generated name changes if you change a field's
direction, which makes hints and monitoring brittle.

## How It Actually Works

A MongoDB index is a **B-tree** (WiredTiger's B+tree variant) whose keys are the indexed values and whose
payload is a **RecordId** — the internal address of the document in the collection's own B-tree. For a
compound index the key is the concatenation of the field values in index order, encoded so that a simple
byte comparison sorts them correctly (descending fields are encoded inverted).

An `IXSCAN` computes **index bounds** from the query — for the ESR index, `status: ["cancelled",
"cancelled"]`, `total: [MaxKey, MinKey]` (all, descending), `placedAt: [2026-03-01, 2026-06-01)` — seeks to
the first key inside the bounds (a tree descent of a few pages), then walks keys in order. For each key it
checks the remaining bounds; for passing keys, the `FETCH` stage uses the RecordId to load the document from
the collection and apply any filter conditions the index couldn't check.

That walk is why order matters. Equality fields first means the scan starts in a narrow region. Sort fields
next means the keys come out **already in the requested order**, so no `SORT` stage is needed and the scan
can stop as soon as `limit` is satisfied. Range fields last means the range is checked per key during the
walk instead of fragmenting the scan. A `SORT` stage, in contrast, must consume **all** of its input before
it can return the first document, and it is limited in memory (100 MB by default before it must spill to
disk or, if disk use is disallowed, fail).

## Common mistakes

- **Creating single-field indexes on every queried field** and hoping the server combines them. It rarely
  does well; design compound indexes for your query shapes.
- **Putting range fields before sort fields**, forcing in-memory sorts.
- **Redundant prefix indexes** (`{a}` next to `{a, b}`).
- **Low-selectivity single-field indexes** alone (an index on `status` with five values still leaves 350,000
  "delivered" documents to fetch).
- **Never removing indexes.** Each one taxes every write.
- **Benchmarking with synthetic data that doesn't resemble production** — distributions decide which plan
  wins.

## Exercise

Using `idx.orders`:

1. Write the best single index for: "a customer's orders, newest first" and verify that `explain` shows no
   `SORT` stage.
2. For "delivered orders in Pune with total over 400, sorted by `placedAt` descending", design the index
   using ESR and compare it with two alternative orders using `hint()`. Record keys/docs examined.
3. Find a query where the E-R-S order beats E-S-R on your data. (Hint: a very narrow date range with no
   `limit`.)
4. Create `{status: 1, total: -1}` and run `sort({status: 1, total: 1})`. Is there a `SORT` stage? What about
   `sort({status: -1, total: 1})`?
5. Measure insert throughput (insert 50,000 documents, time it) with only the `_id` index, then with five
   secondary indexes.
