---
title: "MongoDB Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free MongoDB cheat sheet: 19 quick-reference tables from every level of the MongoDB Mastery Path, each linked to its full lesson."
---

# MongoDB Cheat Sheet

Quick-reference tables from **19 lessons** of the free MongoDB Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · MongoDB Foundations

### [01 · How MongoDB Is Built: mongod, Storage &amp; a Local Lab](level-1/01-architecture-and-setup.md)

*The pieces*

| Program | What it is | Used in |
|---------|------------|---------|
| `mongod` | The database server. Stores data, runs queries, replicates. | every lesson |
| `mongos` | A router that sits in front of a *sharded* cluster and forwards queries to the right shards. | Level 4 |
| `mongosh` | The interactive JavaScript shell. Shipped separately from the server. | every lesson |
| Database tools | `mongodump`, `mongorestore`, `mongoimport`, `mongoexport`, `mongostat`, `mongotop`, `bsondump`. Also shipped separately, with their own version numbers (100.x). | Level 1 · 09, Level 4 |
| Drivers | Libraries for each language (PyMongo, the Node.js `mongodb` package, Java, Go, C#…). | Level 1 · 08 onwards |

### [03 · Documents, BSON &amp; Data Types](level-1/03-documents-bson-types.md)

*The types you will actually use*

| BSON type | `$type` alias | Shell / driver value | Use it for |
|-----------|---------------|----------------------|------------|
| Double | `"double"` | `1.5` | Measurements, scores — anything where tiny rounding is fine |
| 32-bit integer | `"int"` | `1`, `NumberInt(7)` | Counts, small whole numbers |
| 64-bit integer | `"long"` | `NumberLong("7")` | Large IDs, counters that can pass 2 billion |
| Decimal128 | `"decimal"` | `NumberDecimal("0.10")` | Money and anything that must be exact in base 10 |
| String | `"string"` | `"hi"` | UTF-8 text |
| Boolean | `"bool"` | `true` | Flags |
| Date | `"date"` | `new Date()` | Points in time (milliseconds since the Unix epoch, UTC) |
| ObjectId | `"objectId"` | `ObjectId()` | Default `_id` values |
| Embedded document | `"object"` | `{ a: 1 }` | Grouping related fields |
| Array | `"array"` | `[1, 2]` | Lists — and they get special query and index behaviour (lesson 05) |
| Null | `"null"` | `null` | An explicitly empty value |
| Binary | `"binData"` | `BinData(...)`, `UUID()` | Hashes, UUIDs, small files |

### [04 · Inserting &amp; Querying Documents](level-1/04-insert-and-query.md)

*Filters: equality and comparison*

| Operator | Meaning | Example |
|----------|---------|---------|
| (none) | equals | `{ author: "Weir" }` |
| `$eq`, `$ne` | equals / not equals | `{ inStock: { $ne: true } }` |
| `$gt`, `$gte`, `$lt`, `$lte` | ranges | `{ pages: { $lt: 300 } }` |
| `$in`, `$nin` | in / not in a list | `{ author: { $in: ["Weir", "Asimov"] } }` |
| `$exists` | field present or not | `{ inStock: { $exists: false } }` |
| `$type` | BSON type | `{ rating: { $type: "double" } }` |
| `$regex` | pattern match | `{ title: /^the /i }` |

### [06 · Update Operators, Upserts &amp; Array Updates](level-1/06-update-operators.md)

*Field operators*

| Operator | Does | Example |
|----------|------|---------|
| `$set` | set a field (creating it, and any missing parents) | `{$set: {"address.city": "Pune"}}` |
| `$unset` | remove a field | `{$unset: {views: ""}}` |
| `$inc` | add (negative to subtract) | `{$inc: {stock: -1}}` |
| `$mul` | multiply | `{$mul: {price: NumberDecimal("1.1")}}` |
| `$min` / `$max` | set only if the new value is lower / higher | `{$min: {low: 7}}` |
| `$rename` | rename a field | `{$rename: {status: "state"}}` |
| `$currentDate` | set to the server's current date | `{$currentDate: {updatedAt: true}}` |
| `$setOnInsert` | set only when an upsert inserts | see below |

### [09 · mongoimport, mongoexport, mongodump &amp; mongorestore](level-1/09-import-export-dump.md)

| Pair | Format | Preserves BSON types? | Use for |
|------|--------|------------------------|---------|
| `mongoimport` / `mongoexport` | JSON, CSV, TSV (text) | Only with Extended JSON; CSV loses them unless you declare types | Exchanging data with other systems and people |
| `mongodump` / `mongorestore` | BSON (binary) + metadata | Yes, exactly, plus indexes and collection options | Copies and logical backups between MongoDB deployments |

### [10 · Project — Recipe Box Data Layer](level-1/10-project-recipe-box.md)

*The code*

| Requirement | Method | Feature |
|-------------|--------|---------|
| Unique slugs | `__init__`, `add` | unique index + catching `DuplicateKeyError` (code 11000) — lesson 08 |
| All ingredients | `with_all_ingredients` | dot path into an array of documents + `$all` — lesson 05 |
| Quick recipes | `quick` | array "contains" + range + compound sort with tie-breaker — lessons 04, 05 |
| Ratings | `rate` | `find_one_and_update` with `$inc` and `ReturnDocument.AFTER` — lesson 06 |
| Tags | `add_tag` | conditional `$push` with `$each` + `$sort` — lesson 06 |
| Quantities | `set_qty` | positional `$` — lesson 06 |
| Paging | `page` | keyset pagination on `_id` — lesson 04 |
| Delete | `delete` | `deleted_count` — lesson 07 |

## Level 2 · Data Modeling & Indexes

### [01 · Thinking in Documents: Embed or Reference?](level-2/01-embed-or-reference.md)

*The question to ask first*

| Operation | Frequency | Data needed |
|-----------|-----------|-------------|
| Show a blog post page | very high | post + its comments |
| Show the homepage list | very high | titles of recent posts |
| Add a comment | medium | one post |
| Show a user's comment history | low | comments by one user across posts |
| Rename a user | rare | every place the name appears |

### [02 · Modeling Relationships](level-2/02-modeling-relationships.md)

*Trees*

| Shape | Stores | Cheap | Expensive |
|-------|--------|-------|-----------|
| Parent reference | `parent` | children of a node; moving a node | all descendants; the full path |
| Child references | `children: [...]` | children of a node | parents; unbounded for wide trees |
| Array of ancestors | `parent`, `ancestors: [...]` | all descendants, breadcrumbs | moving a subtree |
| Materialized path | `path: ",a,b,c,"` | descendants via prefix, sorting by path | moving a subtree |

### [07 · Reading explain()](level-2/07-reading-explain.md)

*The three numbers that matter*

| Number | Meaning | Ideal |
|--------|---------|-------|
| `nReturned` | documents returned | — |
| `totalKeysExamined` | index entries read | ≈ `nReturned` |
| `totalDocsExamined` | documents loaded from the collection | ≈ `nReturned`, or 0 if covered |

### [10 · Project — Model &amp; Index a Store Catalog](level-2/10-project-store-catalog.md)

*The access patterns*

| # | Operation | Frequency | Shape |
|---|-----------|-----------|-------|
| Q1 | Category page: active products in a category, cheapest first, 24 per page | very high | equality, equality, sort |
| Q2 | Brand page: a brand's active products, best rated first | high | equality, equality, sort |
| Q3 | Spec filter: e.g. laptops with at least 32 GB RAM | high | equality + attribute range |
| Q4 | Product page by SKU | very high | unique equality |
| Q5 | Low-stock report: active products with fewer than 5 in stock | a few times a day | equality + range + sort on the range |

## Level 3 · Aggregation & Application Features

### [01 · Aggregation Pipeline Fundamentals](level-3/01-aggregation-fundamentals.md)

*$group: one output document per group*

| Accumulator | Result |
|-------------|--------|
| `$sum`, `$avg`, `$min`, `$max` | arithmetic over the group |
| `$count` | number of documents (`{ $count: {} }`) |
| `$first`, `$last` | first/last value in the order documents arrive — sort first! |
| `$push`, `$addToSet` | array of all values / distinct values |
| `$top`, `$bottom`, `$topN`, `$bottomN` | best N by an explicit `sortBy`, no prior `$sort` needed |
| `$stdDevPop`, `$median`, `$percentile` | statistics |

### [07 · Change Streams](level-3/07-change-streams.md)

*Watching a collection*

| Field | Contents |
|-------|----------|
| `_id` | the **resume token** — a position in the stream |
| `operationType` | `insert`, `update`, `replace`, `delete`, plus `drop`, `rename`, `invalidate` and others |
| `ns` | database and collection |
| `documentKey` | the changed document's `_id` (and shard key on sharded clusters) |
| `fullDocument` | the new document for inserts/replaces; for updates only if requested |
| `updateDescription` | for updates: `updatedFields`, `removedFields`, `truncatedArrays` |
| `clusterTime` | when the change happened |

## Level 4 · Operations & Production

### [02 · Write Concern, Read Concern &amp; Read Preference](level-4/02-read-write-concerns.md)

*A sensible default set*

| Situation | Write concern | Read concern | Read preference |
|-----------|---------------|--------------|-----------------|
| Most application data | `majority` (default) | `local` (default) | `primary` (default) |
| Must-not-lose and must-not-un-happen (payments, orders) | `majority` | `majority` | `primary` |
| Read-your-writes from secondaries | `majority` | `majority` | `secondary*` + causal session |
| Analytics / reports that tolerate minutes of lag | n/a | `local` | `secondary` with tags |
| Logs or metrics you can afford to lose | `w: 1` | `local` | `primary` |

### [03 · Sharding Architecture](level-4/03-sharding-architecture.md)

*The parts*

| Component | What it does | In this lab |
|-----------|--------------|-------------|
| **Shards** | Each shard is a replica set holding a subset of the data. | `shA`, `shB`, `shC` (single-node replica sets) |
| **Config servers** | A replica set storing the cluster's metadata: which shards exist, which collections are sharded, which ranges live where. | `cfg` (one node) |
| **`mongos`** | Stateless router. Applications connect to it; it caches the routing table and forwards each operation to the right shard(s). | port 27400 |

### [04 · Choosing &amp; Changing a Shard Key](level-4/04-shard-keys.md)

*Comparing candidate keys for orders*

| Key | Write distribution | "Orders for customer X" | "Customer X, last 30 days" | Divisibility |
|-----|--------------------|-------------------------|----------------------------|--------------|
| `{_id: 1}` (ObjectId) | all inserts to one shard (monotonic) | broadcast | broadcast | fine |
| `{city: 1}` | 8 hot spots | broadcast | broadcast | **8 chunks max** |
| `{customerId: "hashed"}` | even | one shard | one shard | fine, but no range targeting on customerId |
| `{customerId: 1, placedAt: 1}` | spread across customers | one shard | one shard | fine — a big customer splits by date |

### [05 · Authentication, Roles &amp; TLS](level-4/05-security.md)

*What the built-in roles actually allow*

| Role | Scope | Can |
|------|-------|-----|
| `read` / `readWrite` | one database | query / query, insert, update, delete, **create and drop collections and indexes** |
| `dbAdmin` | one database | indexes, stats, profiler, validation — but not read data |
| `userAdmin` / `userAdminAnyDatabase` | one / all databases | create users and roles, grant roles (including to itself) |
| `clusterMonitor` | cluster | read-only monitoring commands (`serverStatus`, `replSetGetStatus`, …) |
| `backup` / `restore` | cluster | what `mongodump` / `mongorestore` need |
| `root` | everything | everything |

### [06 · Backup, Restore &amp; Point-in-Time Recovery](level-4/06-backup-restore.md)

*Choosing a strategy*

| Method | Good for | Weak at |
|--------|----------|---------|
| `mongodump --oplog` | Small-to-medium replica sets; logical copies portable across versions | Slow for large data; restore rebuilds every index; not consistent for sharded clusters |
| Volume snapshots | Large data; fast restore | Tied to storage; must cover the whole dbpath atomically |
| Oplog archiving + base backup | Point-in-time recovery to any second | Must keep the oplog archive continuous — a gap breaks the chain |
| Managed backups (Atlas, Ops Manager) | Sharded clusters, scheduled PITR | Not available in a plain Community install; not demonstrated here |

### [07 · Monitoring &amp; the Profiler](level-4/07-monitoring-profiler.md)

*What to put on a dashboard*

| Signal | Source | Why |
|--------|--------|-----|
| Operation rates and latencies by type | `opcounters`, `opLatencies` | load and the first sign of trouble |
| Slow queries per shape | slow log / profiler (`queryShapeHash`) | which query to fix |
| Replication lag per secondary | `rs.status()` optimes | stale reads, failover risk, oplog window |
| Oplog window (hours) | `rs.printReplicationInfo()` | how long a member can be down |
| Cache used / dirty, pages read | `wiredTiger.cache` | working set vs memory |
| Connections current / created | `connections` | pool misconfiguration, churn |
| Queued operations, tickets | `queues.execution` | saturation |
| Disk space, IOPS, CPU | OS | everything else depends on them |
| Elections | log, `rs.status().term` | instability (Lesson 01's laptop had 104 overnight) |

### [10 · Capstone — A Secured Sharded Cluster](level-4/10-capstone-sharded-cluster.md)

| Component | Members | Ports |
|-----------|---------|-------|
| Config server replica set `cfg` | 3 | 27901–27903 |
| Shard replica set `shardA` | 3 | 27911–27913 |
| Shard replica set `shardB` | 3 | 27921–27923 |
| Router `mongos` | 1 | 27900 |

---

Go deeper with the full [MongoDB Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
