---
description: "Make MongoDB pipelines fast: how the optimizer pushes $match and $sort into indexed queries, what blocks it, and precomputed rollups with $merge and $out."
---

# 05 · Pipeline Performance, $out & $merge

A pipeline that takes 20 ms on 30,000 orders can take 20 minutes on 30 million. Whether it does depends
mostly on two things: **how much data enters the pipeline** (decided by the first stages, and whether they
use an index) and **whether you recompute the same answer over and over** (instead of storing it). This
lesson covers both — reading aggregation explains, and building precomputed rollups with `$merge`.

Examples use the `shop` database, which has indexes on `{customerId: 1, placedAt: -1}`, `{placedAt: 1}` and
`{"items.sku": 1}`.

## What the optimizer does for you

Units sold per category in June 2026, written two ways — filter first, and (carelessly) filter after
`$unwind`:

```javascript
const d0 = new Date("2026-06-01"), d1 = new Date("2026-07-01");
const tail = [{ $unwind: "$items" }, { $group: { _id: "$items.category", units: { $sum: "$items.qty" } } }];
db.orders.aggregate([{ $match: { placedAt: { $gte: d0, $lt: d1 } } }, ...tail]);
db.orders.aggregate([{ $unwind: "$items" }, { $match: { placedAt: { $gte: d0, $lt: d1 } } },
                     { $group: { _id: "$items.category", units: { $sum: "$items.qty" } } }]);
```

```text
match first                              index=placedAt_1  docsExamined=2603 ms=7
match after unwind (optimizer moves it?) index=placedAt_1  docsExamined=2603 ms=7
```

Identical. The optimizer noticed that the `$match` only references `placedAt`, which `$unwind` doesn't change,
moved it to the front, and pushed it into the query layer, where it used the `placedAt_1` index. It will even
**split** a `$match`: with `{placedAt: ..., "items.qty": {$gte: 2}}` after `$unwind`, the explain showed

```text
["$cursor","$unwind","$match","$group"]
{"$and":[{"placedAt":{"$lt":"2026-07-01T00:00:00.000Z"}},{"placedAt":{"$gte":"2026-06-01T00:00:00.000Z"}}]}
```

— the `placedAt` part moved into the initial `$cursor` (the indexed query), while the `items.qty` part stayed
after `$unwind`, where it has to be, because it refers to the unwound element.

## What the optimizer can't do

It can't move a filter past a stage that **creates** the field being filtered:

```javascript
db.orders.aggregate([{ $set: { month: { $month: "$placedAt" } } }, { $match: { month: 6 } }, ...tail])
```

```text
match on computed field after $set       index=- COLLSCAN docsExamined=30000 ms=20
```

Every document was read, given a `month`, and then most were thrown away. Express the filter on the stored field
(a `placedAt` range) and it becomes indexable again. (This version also matches June of *every* year — another
reason to filter on real date ranges.)

The same applies to `$group`: anything computed by a `$group` can only be filtered after it, and a `$group`
early in a pipeline reads its whole input:

```text
sort placedAt + limit 5            index=placedAt_1 stages=LIMIT,FETCH,IXSCAN docs=5 ms=1
group, sort, limit                 index=- stages=GROUP,COLLSCAN docs=30000 ms=8
```

A leading `$sort` + `$limit` on an indexed field reads 5 documents. "Top 5 customers by order count" must
group all 30,000 orders first — no index can give you the top of a computed count. That's fine for 30,000; for
300 million it's a batch job, which is where rollups come in.

### Reading an aggregation explain

`db.orders.explain("executionStats").aggregate(pipeline)` returns either a `stages` array whose first element is
`$cursor` (the part pushed into the query layer, with a normal `queryPlanner`/`executionStats` inside), or — when the
whole pipeline runs in the slot-based engine — a top-level `queryPlanner` and `executionStats` like a `find`
(`explainVersion: "2"`). In both, look for the same things as Level 2 · 07: which index, how many documents
examined, and blocking stages.

### Rules of thumb

1. **`$match` first**, on stored fields, with an index that supports it.
2. **`$sort` early** if it can use an index (and before `$group` only if you need the order there).
3. **`$project`/`$unset` early only to drop big fields** you won't use — the optimizer already tracks which fields
   later stages need, so manual early projections rarely help otherwise.
4. **`$limit` as early as correctness allows.**
5. **`$lookup` after you've reduced the documents**, with an index on the foreign field (lesson 03).
6. **Don't `$unwind` just to aggregate within a document** (lesson 02).

## Precomputed rollups with $merge

Dashboards ask the same question every few seconds: revenue by day, city and category. Recomputing it from raw
orders every time is wasted work. Compute it once into a **rollup collection**, and update it incrementally:

```javascript
const rollup = (from, to) => [
  { $match: { status: "delivered", placedAt: { $gte: from, $lt: to } } },
  { $unwind: "$items" },
  { $group: { _id: { day: { $dateTrunc: { date: "$placedAt", unit: "day", timezone: "Asia/Kolkata" } },
                     city: "$city", category: "$items.category" },
              units: { $sum: "$items.qty" },
              revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } } } },
  { $set: { refreshedAt: "$$NOW" } },
  { $merge: { into: "daily_sales", on: "_id", whenMatched: "replace", whenNotMatched: "insert" } }
];
db.orders.aggregate(rollup(new Date("2025-06-30T18:30:00Z"), new Date("2026-07-01T18:30:00Z")));
```

The full build took about 200 ms and wrote 8,679 documents — one per (IST day, city, category):

```text
{ _id: { day: ISODate('2026-06-09T18:30:00.000Z'), city: 'Chennai', category: 'kitchen' },
  units: 7, revenue: Decimal128('5645') }
```

(`2026-06-09T18:30Z` is midnight on 10 June in India — the day boundary in UTC.)

`$merge` writes each output document into the target collection: `on` names the field(s) that identify a document
(they need a unique index — `_id` always has one), `whenMatched` says what to do if it exists (`replace`, `merge`
fields, `keepExisting`, `fail`, or a custom update pipeline), and `whenNotMatched` whether to `insert`, `discard` or
`fail`.

### Querying the rollup

Revenue by category for April–June 2026 (IST), from raw orders and from the rollup:

```text
["books=4450184","electronics=169242891","kitchen=12046031","toys=7353295"]
["books=4450184","electronics=169242891","kitchen=12046031","toys=7353295"]
raw docs examined 7548 rollup docs 2167
```

Same answer, from 2,167 small documents instead of 7,548 orders that also need unwinding. The ratio grows with
volume: the rollup has at most days × cities × categories documents no matter how many orders there are.

**The first attempt at this comparison did not match.** The raw query used UTC month boundaries
(`2026-04-01T00:00Z`) while the rollup's days were IST days, so a few hours of orders at each end fell on different
sides — totals differed by a few thousand rupees per category. A rollup and the queries against it must agree on
the time zone and boundaries; write that down next to the rollup's definition.

### Incremental refresh

When orders change — new deliveries, a status correction — re-run the rollup for just the affected window:

```javascript
db.orders.aggregate(rollup(new Date(day.getTime() - 86400000), new Date(day.getTime() + 86400000)))
```

That recomputed two days in 12 ms. Because `$merge` with `whenMatched: "replace"` overwrites the affected
`(day, city, category)` documents, re-running a window is **idempotent** — safe to repeat after a failure. The
remaining edge case is a group that should now be *empty* (every order of a day/city/category cancelled): replace
won't delete it. Handle it by deleting the window's rollup documents before re-running, or by emitting zero rows
for the window.

Typical schedules: re-run "today and yesterday" every few minutes, and a full rebuild nightly. Change streams
(lesson 07) can trigger refreshes only for days that actually changed.

## $out: replace a collection wholesale

`$out` writes the pipeline's results to a collection, **replacing** it entirely:

```javascript
db.orders.aggregate([{ $group: { _id: "$customerId", orders: { $sum: 1 } } }, { $out: "customer_order_counts" }])
```

That wrote 2,000 documents in 40 ms. The replacement is atomic: `$out` writes to a temporary collection and renames
it over the target at the end, so readers see either the old results or the new ones, never a half-written mix — and
if the pipeline fails, the old collection is untouched. Indexes on the target are preserved across runs:

```text
indexes after re-$out: ["_id_","orders_-1"]
```

Both `$out` and `$merge` must be the **last** stage:

```text
40601 $out can only be the final stage in the pipeline
```

Use `$out` for full rebuilds of a derived collection; `$merge` for incremental updates, for writing into a
collection that has other data, or for writing to a different database.

## How It Actually Works

When the server builds an aggregation, it first runs a sequence of rewrites on the stage list: `$match` stages are
swapped ahead of stages that don't modify the fields they reference (`$unwind`, `$sort`, `$project`, `$set` of other
fields, `$lookup` on other fields), adjacent stages are coalesced (`$sort` + `$limit` → top-k; `$lookup` + `$unwind`
on its output → a single join stage that never materialises the array; `$match` + `$match` → one), and dependency
analysis records which fields the pipeline actually needs. The longest prefix of the pipeline that the query system
can execute — filter, sort, projection, and in the slot-based engine often `$group` and `$lookup` too — is pushed
down and planned with indexes like a `find`. Everything after that prefix runs in the document-at-a-time pipeline
engine.

`$merge` performs, for each output document, the equivalent of an update or insert keyed on the `on` fields, in
batches, with the target's unique index enforcing that each key maps to at most one document. It isn't one big
transaction: if the pipeline fails half-way, the documents already merged stay. That's why idempotent refresh windows
are the right design. `$out` instead builds a whole new collection and swaps it in with a rename, which gives
all-or-nothing replacement at the cost of rewriting everything each time.

## Common mistakes

- **Filtering on computed fields** that could have been expressed on stored, indexed fields.
- **`$group` at the start of a hot query** on a large collection — it reads everything; precompute instead.
- **Rollups and live queries using different time zones or boundaries.**
- **Assuming `$merge` is atomic.** Make refresh windows idempotent.
- **`$out` into a collection other code writes to** — it replaces all of it.
- **Manual `$project` at the start "for performance"** that only makes the pipeline harder to read.

## Exercise

1. Take three pipelines from lessons 01–04 and explain each one. Which use an index? Rewrite one that doesn't so it
   does.
2. Build a `monthly_customer_spend` rollup with `$merge` keyed on `{customerId, month}`. Use it to find each customer's
   best month.
3. Simulate late changes: cancel 50 random delivered orders from last week, re-run the rollup for the affected days
   only, and verify against a fresh full rebuild. Handle the "group became empty" case.
4. Compare `$out` and `$merge` for a full rebuild of `daily_sales`: time both, and check what a concurrent reader sees
   during each (run a reader loop in another shell).
5. Write a `$merge` with `whenMatched` as a pipeline that **adds** new revenue to existing documents instead of
   replacing them. Why is that version not idempotent?
