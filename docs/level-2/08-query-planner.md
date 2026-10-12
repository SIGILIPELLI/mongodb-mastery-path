---
description: "How MongoDB's planner races candidate plans, how the plan cache reuses winners, and how to steer it with hints, query settings and $indexStats."
---

# 08 · The Query Planner, Plan Cache & Hints

When more than one index could answer a query, MongoDB has to pick. It doesn't use a detailed cost model
built on statistics the way PostgreSQL does (see the
[PostgreSQL Mastery Path](https://sigilipelli.github.io/postgresql-mastery-path/) for contrast). Instead it
runs a short **race** between the candidate plans and caches the winner for queries of the same shape.
That design is simple and usually good — and it has one well-known failure mode. This lesson shows the race,
the cache, and the tools for when you need to overrule the planner.

Examples use `idx.orders` (lesson 05), temporarily with extra single-field indexes on `city` and `total`.

## The race

The query: delivered orders in Pune over 490.

```javascript
const q = s => ({ status: s, city: "Pune", total: { $gt: 490 } });
db.orders.find(q("delivered")).explain("allPlansExecution")
```

```text
fresh delivered: status_1_total_-1_placedAt_1 keys 7079 docs 7079 n 906 candidates 4
    status_1_total_-1_placedAt_1 advanced 101 works 815 isEOF 0
    status_1_placedAt_1_total_-1 advanced 2 works 815 isEOF 0
    city_1 advanced 10 works 815 isEOF 0
    total_1 advanced 80 works 815 isEOF 0
```

Four indexes could serve part of the filter, so the planner built four candidate plans and ran them **in
round-robin**, one unit of work each, until one of them produced **101 results** (the default first-batch
size) or finished. The `{status, total, placedAt}` plan got there first: it can apply `status` and `total`
as index bounds and only has to check `city` after fetching. The `city_1` plan, which bounds only on city,
had produced just 10 results in the same number of works.

The winner isn't perfect — it examined 7,079 keys and documents to return 906 — because the ideal index
doesn't exist. Given one designed by ESR:

```text
status_1_city_1_total_1 906 906 906
```

The planner can only choose among the indexes you created. Good index design comes first; the planner
then usually picks the right one.

## The plan cache

Racing costs time, so the winner is cached for the query's **shape** — the filter, sort and projection
structure with the literal values stripped out. After one run:

```javascript
db.orders.aggregate([{ $planCacheStats: {} }])
```

```text
entries 1
{"shape":"782ACF54","key":"AD527657","isActive":false,"works":{"high":0,"low":495,...},"idx":"status_1_total_-1_placedAt_1"}
```

The first run creates an **inactive** entry recording how much work the winner needed (`works: 495`).
Running the same shape again activates it:

```text
{"isActive":true,"works":{"high":0,"low":495,...}}
```

From then on, queries of this shape skip the race and use the cached plan. The literal values don't matter
— `status: "delivered"` and `status: "cancelled"` share the entry:

```text
delivered uses cached? true status_1_total_-1_placedAt_1 keys 7079 n 906
true     ← both queries have the same planCacheShapeHash
```

## When the cache goes wrong

Sharing a plan across values is the failure mode. Imagine a shape where the best index depends on the value:
for `status: "cancelled"` (5% of orders) a status-first index is excellent, but for a value covering 70% of
the collection a different index would be far better. Whichever value runs first decides the plan, and every
query of that shape uses it.

MongoDB defends against this: each time a cached plan runs, the server checks whether it needed much more work
than recorded (more than 10 times the cached `works` by default). If so, it evicts the entry and races again —
**replanning**. That catches the dramatic cases. It doesn't catch plans that are consistently a bit worse than
they could be, and repeated replanning shows up as latency spikes and `replanned: true` in slow-query logs
(Level 4 · 07).

The cache is also cleared when indexes are created or dropped on the collection, and on server restart. You can
clear it by hand:

```javascript
db.orders.getPlanCache().clear()
```

## Overruling the planner

### hint()

`hint()` forces an index for one query:

```javascript
db.orders.find(q("delivered")).hint({ city: 1 }).explain("executionStats")
```

```text
hint city: keys 62491 docs 62491
```

Here the hint makes things much worse (62,491 documents examined vs 7,079) — a good reminder that the planner is
usually right. Hinting an index that doesn't exist is an error:

```text
2 error enumerating plans for query: ns=idx.ordersTree: $and
    city $eq "Pune"
 ...
```

Hints live in application code, so they have to be deployed and they break if an index is renamed or dropped.
Use them sparingly, name the indexes they refer to explicitly, and document why.

### Persistent query settings

Since MongoDB 8.0 you can attach settings to a **query shape** on the server, without changing application
code:

```javascript
db.adminCommand({
  setQuerySettings: { find: "orders", filter: q("delivered"), $db: "idx" },
  settings: { indexHints: { ns: { db: "idx", coll: "orders" }, allowedIndexes: ["city_1"] } } })
```

```text
setQuerySettings ok: 1
now uses: city_1 {"indexHints":[{"ns":{"db":"idx","coll":"orders"},"allowedIndexes":["city_1"]}]}
```

The setting was registered with the value `"delivered"`, but the explain above was for `status: "placed"` — it
still applied, because settings attach to the **shape**. They persist across restarts and are replicated, and
`$querySettings` lists them:

```javascript
db.getSiblingDB("admin").aggregate([{ $querySettings: {} }])
db.adminCommand({ removeQuerySettings: { find: "orders", filter: q("delivered"), $db: "idx" } })
```

After removal the planner went back to `status_1_total_-1_placedAt_1`. Query settings can also **reject** a
shape entirely (`reject: true`) — a useful emergency brake for a runaway query from a buggy deployment. They
replace the older, non-persistent *index filters* (`planCacheSetFilter`).

## Finding indexes nothing uses

Every index taxes writes (lesson 05). `$indexStats` reports how often each index has been used since the server
started (or since the index was created):

```javascript
db.orders.aggregate([{ $indexStats: {} }]).toArray().map(i => ({ name: i.name, ops: i.accesses.ops }))
```

```text
[
  { name: 'total_1', ops: Long('0') },
  { name: 'city_1', ops: Long('0') },
  { name: 'status_1_placedAt_1_total_-1', ops: Long('3') },
  { name: '_id_', ops: Long('0') },
  { name: 'placedAt_1_status_1_total_-1', ops: Long('0') },
  { name: 'status_1_total_-1_placedAt_1', ops: Long('2') },
  { name: 'customerId_1_placedAt_-1', ops: Long('0') }
]
```

Counters are per server and reset on restart, so look at **every** member of a replica set (secondaries may
serve reads the primary never sees), over a period that includes your monthly reports and batch jobs. Then
hide (lesson 06) before dropping.

## How It Actually Works

Planning happens in three steps. **Canonicalisation** normalises the query (sorting `$and` children,
simplifying nested logic) and computes its shape hash. **Enumeration** asks, for each predicate, which indexes
could provide bounds for it, and builds candidate plans — one per usable index, plus combinations for `$or`
and, rarely, index intersection. The number of candidates is capped so a query with many indexed fields
doesn't explode into hundreds of plans.

If there is exactly one candidate, it runs directly. Otherwise the **multi-planner** runs a trial: it calls
each plan's root stage in turn (the `works` you saw), buffering any results produced, until one plan hits
EOF, returns a full batch (101 documents) or the trial reaches its work limit (based on collection size). Each
plan is scored mainly by *productivity* — results advanced per unit of work — with small bonuses for avoiding
a fetch or an in-memory sort. The best plan continues from where it left off (its buffered results aren't
wasted), and its works count is stored in the plan cache entry.

On later executions the cached plan runs as a **CachedPlan** stage with a budget proportional to the stored
works. If it exhausts the budget without finishing its first batch, the server concludes that the data no longer
fits the cached plan, evicts it and runs the multi-planner again. That's replanning — cheap insurance, but not
an optimiser with statistics, which is why skewed data plus a shared shape is the classic case where you reach
for query settings.

## Common mistakes

- **Hinting by habit.** The planner is right far more often than intuition; measure first.
- **Hinting by index name in code** and then renaming or dropping the index.
- **Ignoring data skew**: one shape, values with wildly different selectivity, one cached plan.
- **Reading `$indexStats` on one member only** and dropping an index a secondary depends on.
- **Assuming the plan you saw in development is the one production uses.** Different data, different race
  result.
- **Too many overlapping indexes**, which make every race longer and the outcome less predictable.

## Exercise

1. Clear the plan cache, run `{status: "paid", city: "Delhi"}` twice and inspect `$planCacheStats`. Then run the
   same shape with different values and confirm it reuses the entry.
2. Create a skewed collection: 1,000,000 documents where `type` is `"a"` for 99% and `"b"` for 1%, with indexes
   on `{type: 1}` and `{createdAt: 1}`. Run `find({type, createdAt: {$gte: ...}})` first with `"b"`, then with
   `"a"`. Use slow-query logging (`db.setProfilingLevel(0, {slowms: 0})` and the log) to look for replanning.
3. Use `setQuerySettings` to reject a query shape, run the query and record the error. Remove the setting.
4. List all indexes on `idx.orders` that have had zero uses since you started your server, hide them, and rerun
   lesson 05's queries to see whether anything changed.
