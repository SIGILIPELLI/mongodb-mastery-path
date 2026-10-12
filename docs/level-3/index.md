---
description: "MongoDB Level 3: the aggregation pipeline, expressions, $lookup, window functions, $merge rollups, transactions, change streams, geo and text search."
---

# Level 3 · Aggregation & Application Features

With a good schema and the right indexes (Level 2), Level 3 turns to what applications *do* with their data.
Most of it is the **aggregation pipeline**: reports, dashboards, analytics, joins and precomputed rollups. The
rest is the set of features that make MongoDB an application platform rather than just a store — multi-document
transactions, change streams that push every change to your code, location queries and text search.

The lessons share one seeded dataset — an online shop with 2,000 customers, 200 products and 30,000 orders — so
results build on each other. You'll see a `$lookup` go from examining 300,025 documents to 2,967 by adding one index,
a rollup that matched raw data only after its time zones were aligned, eight threads of bank transfers that needed
492 retries and still balanced to the paisa, a change-stream consumer resume after a "crash" without losing an event,
and a "local" store placed 7,005 km away by swapping latitude and longitude.

**Prerequisites:** Levels 1 and 2, the lab replica set (transactions and change streams need one), and Python with
PyMongo for lessons 06, 07 and 10.

## Modules

1. **[Aggregation Pipeline Fundamentals](01-aggregation-fundamentals.md)** — `$match`, `$group`, accumulators,
   `$unwind`, `$sort`, `$limit`, and why stage order matters.
2. **[Reshaping Documents with Expressions](02-reshaping-expressions.md)** — `$map`, `$filter`, `$reduce`,
   conditions, time zones, `$replaceWith` and safe type conversion.
3. **[Joins & Unions: $lookup, $graphLookup, $unionWith](03-lookup-joins.md)** — join forms and strategies, the
   foreign-field index, hierarchies and archives.
4. **[Analytics: $facet, $bucket & Window Functions](04-analytics-windows.md)** — multi-facet results, histograms,
   running totals, moving averages, ranks and gap filling.
5. **[Pipeline Performance, $out & $merge](05-pipeline-performance.md)** — what the optimizer does, what blocks it,
   and incremental rollups.
6. **[Multi-Document Transactions](06-transactions.md)** — the callback API, isolation, write conflicts and retries,
   limits and real costs.
7. **[Change Streams](07-change-streams.md)** — event shapes, pre-images, resume tokens, the oplog window and
   invalidation.
8. **[Geospatial Queries](08-geospatial.md)** — GeoJSON, 2dsphere, `$near`, `$geoNear`, zones and the
   longitude-first trap.
9. **[Text Search & Collation](09-text-search-collation.md)** — text indexes and their limits; case- and
   accent-insensitive comparison and sorting.
10. **[Project — Order Analytics Backend](10-project-order-analytics.md)** — a tested dashboard backend with a rollup
    kept exact by a change-stream worker.

## After Level 3

You can build substantial applications on MongoDB. Level 4 is about running them: replica sets and failover, write
and read concerns, sharding, security, backups, monitoring, tuning and upgrades — ending with a secured, sharded
cluster built and tested from scratch.
