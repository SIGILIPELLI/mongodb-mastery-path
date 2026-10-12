---
description: "Read MongoDB explain() output: verbosity modes, plan stages, index bounds, keys vs docs examined, covered queries and blocking sorts that spill."
---

# 07 · Reading explain()

An index you *think* a query uses is worth nothing until `explain()` confirms it. `explain()` is how you
see what the server actually did: which plan it chose, which index bounds it scanned, how many index keys
and documents it touched, and where the time went. Learning to read it quickly is the most practical
performance skill in MongoDB.

Everything here runs against the 500,000-document `idx.orders` collection from lesson 05, with an extra
index `{customerId: 1, placedAt: -1}`.

## Three verbosity levels

```javascript
db.orders.find(q).explain()                     // "queryPlanner": chosen plan, not executed
db.orders.find(q).explain("executionStats")     // runs the winning plan and reports counts and times
db.orders.find(q).explain("allPlansExecution")  // also reports the trial runs of rejected plans
```

`queryPlanner` is safe on any query because it doesn't run it. `executionStats` **runs the query to
completion** — fine for reads, but remember that explaining an expensive query costs as much as running it.
(Explaining an `update` or `remove` with `executionStats` evaluates it without applying the writes.)

Aggregations use `db.orders.explain("executionStats").aggregate([...])` — Level 3 · 05.

## Anatomy of an explain

"A customer's orders, newest first":

```javascript
db.orders.find({ customerId: 12345 }).sort({ placedAt: -1 }).explain("executionStats")
```

The trimmed output, top half:

```text
queryPlanner: {
  namespace: 'idx.orders',
  parsedQuery: { customerId: { '$eq': 12345 } },
  winningPlan: {
    stage: 'FETCH',
    inputStage: {
      stage: 'IXSCAN',
      keyPattern: { customerId: 1, placedAt: -1 },
      indexName: 'customerId_1_placedAt_-1',
      isMultiKey: false,
      direction: 'forward',
      indexBounds: { customerId: [ '[12345, 12345]' ], placedAt: [ '[MaxKey, MinKey]' ] }
    }
  },
  rejectedPlans: [ ]
}
```

Read the plan **from the innermost stage outwards** — data flows up. `IXSCAN` walks the index; `FETCH`
loads each document. There's no `SORT` stage: the index already delivers `placedAt` descending. The
**index bounds** say exactly which part of the index was scanned: one `customerId` value, all `placedAt`
values (`MaxKey` to `MinKey` because the field is descending). Bounds are the first thing to check when an
indexed query is slow — `[MinKey, MaxKey]` on your first field means the index is being scanned end to end.

The bottom half:

```text
executionStats: {
  executionSuccess: true,
  nReturned: 11,
  executionTimeMillis: 13,
  totalKeysExamined: 11,
  totalDocsExamined: 11,
  executionStages: {
    stage: 'FETCH', nReturned: 11, works: 12, advanced: 11, needTime: 0, needYield: 0, isEOF: 1,
    docsExamined: 11,
    inputStage: { stage: 'IXSCAN', nReturned: 11, keysExamined: 11, seeks: 1, ... }
  }
}
```

## The three numbers that matter

| Number | Meaning | Ideal |
|--------|---------|-------|
| `nReturned` | documents returned | — |
| `totalKeysExamined` | index entries read | ≈ `nReturned` |
| `totalDocsExamined` | documents loaded from the collection | ≈ `nReturned`, or 0 if covered |

Here all three are 11: perfect. The ratios tell you what's wrong when it isn't:

- **keys ≫ returned**: the index bounds are too wide — wrong field order or a missing equality field.
- **docs ≫ returned, keys ≈ docs**: the index finds candidates but a filter applied *after* the fetch
  rejects most of them — add that field to the index.
- **docs = collection size, keys = 0**: `COLLSCAN`, no usable index.

`executionTimeMillis` is useful but noisy (cache state, concurrent load, the first run warming the
cache). Trust the examined counts more than the milliseconds.

## Stages you'll see

| Stage | What it does |
|-------|--------------|
| `COLLSCAN` | read every document |
| `IXSCAN` | walk a range of an index |
| `FETCH` | load documents by RecordId, apply remaining filters |
| `SORT` | in-memory (blocking) sort |
| `LIMIT`, `SKIP` | as named |
| `PROJECTION_SIMPLE` / `PROJECTION_DEFAULT` | apply a projection to fetched documents |
| `PROJECTION_COVERED` | build results from index keys alone — no `FETCH` |
| `OR` | combine the results of several index scans for an `$or` |
| `SORT_MERGE` | merge several already-sorted index scans |
| `EXPRESS_IXSCAN` | fast path for an exact `_id` lookup or a single-result equality match on a unique index (on 9.0.2, `find({_id: x})` showed this stage) |
| `IDHACK` | the older servers' name for the `_id` fast path — you'll still meet it in explains from pre-8.0 deployments |

Newer servers may also run eligible queries in the **slot-based execution engine** (SBE); explain then shows
an additional `slotBasedPlan` section and `queryPlanner.winningPlan.queryPlan` nesting. The same
principles apply: find the scans, check bounds and examined counts.

## Four plans side by side

Using the same helper as lesson 05:

```text
covered                        PROJECTION_COVERED<-IXSCAN               n=11 keys=11 docs=0 ms=2
not covered (needs total)      PROJECTION_SIMPLE<-FETCH<-IXSCAN         n=11 keys=11 docs=11 ms=0
not covered (_id returned)     PROJECTION_SIMPLE<-FETCH<-IXSCAN         n=11 keys=11 docs=11 ms=0
filter after fetch             FETCH<-IXSCAN                            n=4 keys=11 docs=11 ms=1
```

The queries:

```javascript
o.find({ customerId: 12345 }, { _id: 0, customerId: 1, placedAt: 1 }).sort({ placedAt: -1 })   // covered
o.find({ customerId: 12345 }, { _id: 0, placedAt: 1, total: 1 }).sort({ placedAt: -1 })         // needs total
o.find({ customerId: 12345 }, { placedAt: 1 }).sort({ placedAt: -1 })                           // _id included
o.find({ customerId: 12345, items: { $gte: 4 } }).sort({ placedAt: -1 })                       // extra filter
```

**Covered query**: every field the query filters on and returns is in the index, so the server builds the
results from the index keys and examines **zero documents**. Note the second and third lines: asking for one
field not in the index (`total`), or forgetting to exclude `_id` (included by default, and not in this
index), forces a fetch of every document. Covered queries are the fastest reads MongoDB can do, and they're
often one projection change away. (An index on a multikey field can't cover queries on that field.)

**Filter after fetch**: `items` isn't in the index, so 11 documents are fetched and 7 are thrown away. With
11 that's irrelevant; with 11,000 it isn't. If this query is frequent, the index should include `items`.

## Blocking sorts

When no index provides the order, a `SORT` stage collects **every** input document before returning the
first one. All delivered orders sorted by city, then total:

```text
in-memory sort                 SORT<-FETCH<-IXSCAN                      n=349643 keys=349643 docs=349643 ms=861
SORT memLimit 104857600 totalDataSizeSorted 78874594 usedDisk false spills 0
```

349,643 documents, 79 MB sorted in memory, under the 100 MB limit (the server parameter
`internalQueryMaxBlockingSortMemoryUsageBytes`). To see what happens beyond the limit without generating
gigabytes of data, the lab server's limit was lowered to 10 MB for a moment:

```javascript
db.adminCommand({ setParameter: 1, internalQueryMaxBlockingSortMemoryUsageBytes: 10 * 1024 * 1024 });
o.find({ status: "delivered" }).sort({ city: 1, total: -1 }).allowDiskUse(false).toArray()
```

```text
31157 Location31157 No viable plan was found because all candidate plans failed.
```

With disk use disallowed, the sort fails. With the default (`allowDiskUseByDefault` is true since MongoDB
6.0), it spills to temporary files instead:

```text
usedDisk true spills 9 ms 876
```

Nine spills to disk — survivable, but it's I/O on the database server for every execution. (The parameter was
then set back to 100 MB.) Two better fixes:

- **Add a limit**: `sort(...).limit(10)` turns the blocking sort into a top-k sort that keeps only 10
  documents in memory. Its explain showed `limitAmount 10` and `totalDataSizeSorted 2280` bytes instead of 79
  MB — though it still had to *read* all 349,643 candidates (435 ms).
- **Index the sort**: `{status: 1, city: 1, total: -1}` removes the `SORT` stage and lets the scan stop after
  10 keys.

## allPlansExecution: why the planner chose what it chose

```javascript
o.find({ status: "cancelled", placedAt: { $gte: new Date("2026-03-01"), $lt: new Date("2026-06-01") } })
 .sort({ total: -1 }).limit(20).explain("allPlansExecution")
```

```text
candidates: 3
   status_1_total_-1_placedAt_1 works=219 advanced=20 keys=219
   status_1_placedAt_1_total_-1 works=219 advanced=0 keys=219
   placedAt_1_status_1_total_-1 works=219 advanced=0 keys=219
```

The planner ran all three candidate plans **for the same amount of work** (219 units) and compared how many
results each produced. The ESR index had already produced all 20; the other two were still scanning and
hadn't returned anything (they need to finish scanning before their `SORT` can output). That trial is the
subject of lesson 08.

## How It Actually Works

A query plan is a tree of **stages**, and execution is pull-based: the root stage asks its child for the next
result, which asks its child, down to the scans. Each request is a unit of **work**; a stage's response is
either *advanced* (here's a result), *needTime* (I did work but have nothing yet — e.g. an index key that
failed a filter), or *EOF*. That's what the `works`, `advanced` and `needTime` counters in
`executionStages` record, per stage. A `SORT` stage returns `needTime` until its child is exhausted, then
advances results from its buffer — which is why a blocking sort can't benefit from `limit` the way an
index-ordered scan can.

`keysExamined` counts index entries the `IXSCAN` stepped over; `seeks` counts jumps to a new position in the
index (more than one when bounds have several ranges, e.g. an `$in` list). `docsExamined` counts `FETCH`
lookups through the collection's B-tree by RecordId. Because every one of those is a potential cache miss
and disk read, the examined counts predict how a query will behave on a cold cache or a larger dataset far
better than milliseconds measured on a warm laptop.

## Common mistakes

- **Trusting that an index is used** because it exists. Explain it.
- **Looking only at the winning stage name.** An `IXSCAN` with `[MinKey, MaxKey]` bounds is a scan in
  disguise.
- **Comparing milliseconds from single runs** instead of examined counts.
- **Forgetting `_id: 0`** in a projection meant to be covered.
- **Leaving blocking sorts on hot queries** because they "fit in memory" on today's data.
- **Running `executionStats` on a very expensive query in production** during peak load.

## Exercise

1. For each query, predict the stages and the three numbers, then check with `explain("executionStats")`:
   `{customerId: 777}`; `{customerId: 777}` projected to be covered; `{city: "Pune"}`;
   `{status: "paid"}` sorted by `total` descending with `limit(5)`.
2. Find a query on `idx.orders` where `totalKeysExamined` is more than 100 × `nReturned`. Design an index that
   brings the ratio close to 1.
3. Make the "delivered orders by city and total" query run with no `SORT` stage, and compare its explain with
   the blocking-sort version above.
4. Run an `$or` query on `customerId` and `city` with indexes on each and find the `OR` stage in the plan.
   What happens to the plan if one branch has no index?
