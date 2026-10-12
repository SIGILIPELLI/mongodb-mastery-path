---
description: "MongoDB Level 2: embed vs reference, schema patterns, $jsonSchema validation, compound and special indexes, explain(), the planner and time series."
---

# Level 2 · Data Modeling & Indexes

Level 1 taught you to read and write documents correctly. Level 2 is about the two decisions that decide
whether a MongoDB application stays fast as it grows: **the shape of your documents** and **the indexes
behind your queries**. Both are made up front, both are expensive to change later, and both are routinely
gotten wrong by people who know the syntax perfectly well.

The lessons measure instead of asserting. You'll watch an embedded comment list hit the 16 MB wall at
180,000 comments, see a bucketed design use 29 times less index than one document per reading, compare three
orders of the same compound index on half a million orders (71,514 keys examined versus 219), and watch the
planner race four candidate plans. Every number was produced on MongoDB 9.0.2 on a laptop; your absolute
timings will differ, the proportions shouldn't.

**Prerequisites:** Level 1. You'll want the lab replica set running and about 1 GB of free disk for the test
collections.

## Modules

1. **[Thinking in Documents: Embed or Reference?](01-embed-or-reference.md)** — read cost, document growth, the
   16 MB wall, duplication and atomicity, measured.
2. **[Modeling Relationships](02-modeling-relationships.md)** — one-to-few through one-to-squillions,
   many-to-many, and four ways to store a tree.
3. **[Schema Design Patterns](03-schema-design-patterns.md)** — computed, subset, extended reference, bucket,
   attribute, outlier, polymorphic and schema versioning.
4. **[Schema Validation with $jsonSchema](04-schema-validation.md)** — validators, error details, cross-field
   rules, and rolling rules onto existing data.
5. **[How Indexes Work & Compound Index Design](05-indexes-compound.md)** — B-trees, prefixes, and the
   Equality-Sort-Range rule.
6. **[Multikey, Unique, Partial, TTL, Wildcard & Text Indexes](06-index-types.md)** — what each option does and
   the trap each one hides.
7. **[Reading explain()](07-reading-explain.md)** — stages, bounds, keys vs documents examined, covered queries
   and blocking sorts.
8. **[The Query Planner, Plan Cache & Hints](08-query-planner.md)** — the plan race, cached plans, replanning,
   hints, query settings and `$indexStats`.
9. **[Time Series & Capped Collections](09-time-series-capped.md)** — server-side bucketing and compression for
   measurements; bounded logs with tailable cursors.
10. **[Project — Model & Index a Store Catalog](10-project-store-catalog.md)** — 100,000 products, a validator,
    five indexed queries and an automated explain-based budget check.

## After Level 2

Your data has a good shape and your queries hit indexes. Level 3 turns to what you do *with* the data: the
aggregation pipeline for reports and analytics, joins, multi-document transactions, change streams,
geospatial queries and text search.
