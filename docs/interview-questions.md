---
title: "MongoDB Interview Questions and Answers (179 Questions)"
description: "179 MongoDB interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the MongoDB Mastery Path."
---

# MongoDB Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free MongoDB Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · MongoDB Foundations

### [01 · How MongoDB Is Built: mongod, Storage &amp; a Local Lab](level-1/01-architecture-and-setup.md)

**Q1.** What is MongoDB's data model?

**Answer:** Flexible JSON-like (BSON) documents grouped into collections

Schema flexibility.

**Q2.** What's the hierarchy?

**Answer:** Deployment, database, collection, document

Namespaces.

**Q3.** What does mongod do?

**Answer:** Runs the database server process

mongosh is the shell.

**Q4.** What is a replica set?

**Answer:** A group of mongod instances maintaining the same data for availability

Primary plus secondaries.

### [02 · mongosh Like a Pro](level-1/02-mongosh-like-a-pro.md)

**Q5.** What does `use shop` do in mongosh?

**Answer:** Switches the current database (created lazily on first write)

Lazy creation.

**Q6.** What does `db.items.find().pretty()` style output help with?

**Answer:** Readable formatted results

mongosh formats by default.

**Q7.** What is mongosh?

**Answer:** The modern MongoDB shell, a JavaScript environment

Scripting.

**Q8.** How run a script file in mongosh?

**Answer:** mongosh --file script.js (or load())

Automation.

### [03 · Documents, BSON &amp; Data Types](level-1/03-documents-bson-types.md)

**Q9.** What is the _id field?

**Answer:** A unique primary key for every document (ObjectId by default)

Immutable.

**Q10.** What does an ObjectId embed?

**Answer:** A timestamp, random value and counter

Roughly time-ordered.

**Q11.** Which BSON types exist beyond JSON?

**Answer:** Date, ObjectId, Decimal128, Int32/Int64 and binary

Richer types.

**Q12.** Why use Decimal128 for money?

**Answer:** Floating point can't represent decimals exactly

Avoid rounding errors.

### [04 · Inserting &amp; Querying Documents](level-1/04-insert-and-query.md)

**Q13.** What does `insertOne` return?

**Answer:** An acknowledgement with the inserted _id

insertedId.

**Q14.** What does `find({age: {$gt: 30}})` do?

**Answer:** Returns documents where age is greater than 30

Query operators.

**Q15.** What does a projection do?

**Answer:** Limits which fields are returned

find({}, {name:1}).

**Q16.** What does `sort({age: -1})` mean?

**Answer:** Descending by age

1 is ascending.

### [05 · Querying Arrays &amp; Embedded Documents](level-1/05-arrays-embedded-queries.md)

**Q17.** How query a field inside an embedded document?

**Answer:** Dot notation: find({"address.city": "Oslo"})

Quotes required.

**Q18.** What does `{tags: "mongo"}` match for an array field?

**Answer:** Documents whose array contains that element

Array containment.

**Q19.** What does $elemMatch do?

**Answer:** Matches array elements satisfying multiple conditions together

Same-element conditions.

**Q20.** What does $size do?

**Answer:** Matches arrays with an exact length

Not a range.

### [06 · Update Operators, Upserts &amp; Array Updates](level-1/06-update-operators.md)

**Q21.** Which operator changes a field's value?

**Answer:** $set

Plus others.

**Q22.** What does $inc do?

**Answer:** Atomically increments a numeric field

Counters.

**Q23.** What does $push do?

**Answer:** Appends a value to an array

$addToSet avoids duplicates.

**Q24.** What does upsert: true do?

**Answer:** Inserts a document if none matches

Update or insert.

### [07 · Deletes, Replaces &amp; Bulk Writes](level-1/07-deletes-replaces-bulk.md)

**Q25.** What does `deleteMany({})` do?

**Answer:** Deletes every document in the collection

Dangerous.

**Q26.** What's the difference between replaceOne and updateOne?

**Answer:** replaceOne swaps the whole document (except _id); updateOne applies operators

Be careful.

**Q27.** What's bulkWrite for?

**Answer:** Sending many operations in one round trip

Efficiency.

**Q28.** What does ordered: false do in bulk?

**Answer:** Continues after an error, running remaining operations

Throughput.

### [08 · Drivers: PyMongo &amp; the Node.js Driver](level-1/08-drivers.md)

**Q29.** What does an official driver provide?

**Answer:** A language API to connect, query and manage MongoDB

Node, Python, Java.

**Q30.** What's a connection string?

**Answer:** A URI like mongodb://host:27017/db with options

+srv for Atlas.

**Q31.** Why reuse one MongoClient?

**Answer:** It manages a connection pool; creating many is wasteful

Create once.

**Q32.** What should you do with credentials?

**Answer:** Keep them in environment/config, never in code

Secrets.

### [09 · mongoimport, mongoexport, mongodump &amp; mongorestore](level-1/09-import-export-dump.md)

**Q33.** What does mongodump produce?

**Answer:** A BSON binary export of data

mongorestore reloads.

**Q34.** What does mongoexport produce?

**Answer:** JSON or CSV of a collection

mongoimport.

**Q35.** When use dump/restore vs export/import?

**Answer:** Dump for faithful backups (types/indexes); export for interchange

Types preserved.

**Q36.** What's lost in JSON export?

**Answer:** Some BSON type fidelity unless extended JSON is used

Use canonical mode.

### [10 · Project — Recipe Box Data Layer](level-1/10-project-recipe-box.md)

**Q37.** What does the recipe box project combine?

**Answer:** CRUD, embedded ingredients/steps, queries, updates and import/export

Level 1 project.

**Q38.** How model ingredients?

**Answer:** An embedded array of subdocuments

Read together.

**Q39.** How find recipes by ingredient?

**Answer:** Query the array with dot notation or $elemMatch

Index it later.

**Q40.** How avoid duplicates?

**Answer:** A unique index or upsert on a natural key

Constraint.

## Level 2 · Data Modeling & Indexes

### [01 · Thinking in Documents: Embed or Reference?](level-2/01-embed-or-reference.md)

**Q41.** When embed related data?

**Answer:** When it's read together, bounded in size and owned by the parent

One read.

**Q42.** When reference instead?

**Answer:** Large, unbounded or independently accessed/shared data

Avoid huge documents.

**Q43.** What's the document size limit?

**Answer:** 16 MB

Design around it.

**Q44.** What antipattern grows unboundedly?

**Answer:** Embedding ever-growing arrays in a document

Use buckets/references.

### [02 · Modeling Relationships](level-2/02-modeling-relationships.md)

**Q45.** How model one-to-many with few children?

**Answer:** Embed them in the parent document

Simple.

**Q46.** How model many-to-many?

**Answer:** Arrays of references on one or both sides

Design by queries.

**Q47.** What guides modeling?

**Answer:** The application's access patterns

Query-driven.

**Q48.** What's the trade-off of duplication?

**Answer:** Faster reads versus harder updates

Controlled denormalization.

### [03 · Schema Design Patterns](level-2/03-schema-design-patterns.md)

**Q49.** What does the bucket pattern do?

**Answer:** Groups many small related records (e.g. time series) into one document

Fewer documents.

**Q50.** What's the computed pattern?

**Answer:** Storing precomputed values to avoid expensive recomputation

Read-heavy.

**Q51.** What's the subset pattern?

**Answer:** Keeping the hot subset in the main document and the rest elsewhere

Working set.

**Q52.** What's the extended reference pattern?

**Answer:** Copying frequently needed fields from a referenced doc to avoid joins

Denormalize wisely.

### [04 · Schema Validation with $jsonSchema](level-2/04-schema-validation.md)

**Q53.** What does $jsonSchema validation do?

**Answer:** Enforces document structure and types on writes

Collection validator.

**Q54.** What does validationAction: warn do?

**Answer:** Logs violations without rejecting writes

Migrations.

**Q55.** Why validate if MongoDB is flexible?

**Answer:** Catch bad data early while keeping flexibility where needed

Guardrails.

**Q56.** When does validation apply?

**Answer:** On inserts and updates (validationLevel controls existing docs)

Configurable.

### [05 · How Indexes Work &amp; Compound Index Design](level-2/05-indexes-compound.md)

**Q57.** What is a compound index?

**Answer:** An index on multiple fields in a specific order

Order matters.

**Q58.** What's the ESR guideline?

**Answer:** Equality, Sort, Range for ordering compound index fields

Query shape.

**Q59.** What does a covered query do?

**Answer:** Answers from the index alone without reading documents

Fast.

**Q60.** What's the cost of indexes?

**Answer:** Slower writes and more storage/memory

Index only what's needed.

### [06 · Multikey, Unique, Partial, TTL, Wildcard &amp; Text Indexes](level-2/06-index-types.md)

**Q61.** What are TTL indexes?

**Answer:** Indexes that automatically delete documents after a time

Expiring data.

**Q62.** What does a partial index do?

**Answer:** Indexes only documents matching a filter

Smaller index.

**Q63.** What does a unique index enforce?

**Answer:** No duplicate values for the indexed field(s)

Constraint.

**Q64.** What does a text index support?

**Answer:** Full-text search over string fields

$text.

### [07 · Reading explain()](level-2/07-reading-explain.md)

**Q65.** What does explain('executionStats') show?

**Answer:** How a query ran: plan, docs examined and returned, time

Diagnosis.

**Q66.** What's a COLLSCAN?

**Answer:** A full collection scan (usually needs an index)

Slow at scale.

**Q67.** What ratio signals an inefficient query?

**Answer:** docsExamined much larger than nReturned

Selectivity.

**Q68.** What's IXSCAN?

**Answer:** A scan using an index

Good sign.

### [08 · The Query Planner, Plan Cache &amp; Hints](level-2/08-query-planner.md)

**Q69.** What does the query planner do?

**Answer:** Chooses an execution plan by trialing candidate indexes

Plan cache.

**Q70.** What is a winning plan?

**Answer:** The candidate that returned results fastest in the trial

Cached.

**Q71.** What can force an index?

**Answer:** hint() (use sparingly)

Debugging.

**Q72.** Why might a plan become bad?

**Answer:** Data distribution changes make the cached plan inefficient

Replan.

### [09 · Time Series &amp; Capped Collections](level-2/09-time-series-capped.md)

**Q73.** What are time series collections for?

**Answer:** Efficiently storing measurements over time with automatic bucketing

IoT/metrics.

**Q74.** What's a capped collection?

**Answer:** A fixed-size collection that overwrites oldest documents

Logs.

**Q75.** What defines a time series collection?

**Answer:** timeField, metaField and granularity

Options.

**Q76.** What's a capped collection's limit?

**Answer:** Max size (and optionally document count) with insertion-order retention

No deletes.

### [10 · Project — Model &amp; Index a Store Catalog](level-2/10-project-store-catalog.md)

**Q77.** What does the store catalog project combine?

**Answer:** Modeling, validation, indexes and explain-driven tuning

Level 2 project.

**Q78.** How justify an index?

**Answer:** By explain output showing improvement for real queries

Evidence.

**Q79.** How model product variants?

**Answer:** Embedded variants array or attribute pattern

Flexible.

**Q80.** How enforce SKU uniqueness?

**Answer:** A unique index on sku

Constraint.

## Level 3 · Aggregation & Application Features

### [01 · Aggregation Pipeline Fundamentals](level-3/01-aggregation-fundamentals.md)

**Q81.** What is an aggregation pipeline?

**Answer:** A sequence of stages transforming documents

$match, $group.

**Q82.** Which stage filters documents early?

**Answer:** $match (put it first to use indexes)

Performance.

**Q83.** What does $group do?

**Answer:** Groups documents by a key and computes accumulators

_id is the key.

**Q84.** What does $unwind do?

**Answer:** Deconstructs an array into one document per element

Flatten.

### [02 · Reshaping Documents with Expressions](level-3/02-reshaping-expressions.md)

**Q85.** What does $project/$addFields do?

**Answer:** Reshape documents by selecting or computing fields

Expressions.

**Q86.** What does $cond do?

**Answer:** Evaluates an if/then/else expression

Conditional fields.

**Q87.** What does $map do?

**Answer:** Applies an expression to each array element

Array transform.

**Q88.** What does $facet do?

**Answer:** Runs multiple sub-pipelines on the same input

Dashboards.

### [03 · Joins &amp; Unions: $lookup, $graphLookup, $unionWith](level-3/03-lookup-joins.md)

**Q89.** What does $lookup do?

**Answer:** A left outer join with another collection

Join-like.

**Q90.** Why index the foreign field?

**Answer:** Otherwise each lookup scans the other collection

Performance.

**Q91.** When prefer embedding over $lookup?

**Answer:** When data is read together so no join is needed

Model for access.

**Q92.** What does $unwind after $lookup do?

**Answer:** Flattens the joined array into documents

Common pairing.

### [04 · Analytics: $facet, $bucket &amp; Window Functions](level-3/04-analytics-windows.md)

**Q93.** What do window functions ($setWindowFields) add?

**Answer:** Running totals, moving averages and ranks without collapsing documents

Analytics.

**Q94.** What does $bucket do?

**Answer:** Groups values into ranges

Histograms.

**Q95.** What does $dateTrunc do?

**Answer:** Truncates dates to units like day or hour

Time grouping.

**Q96.** What does $sortByCount do?

**Answer:** Groups by a value and sorts by frequency

Top-N.

### [05 · Pipeline Performance, $out &amp; $merge](level-3/05-pipeline-performance.md)

**Q97.** How speed up pipelines?

**Answer:** Filter and limit early and use indexes for the first stages

Reduce data.

**Q98.** What does allowDiskUse do?

**Answer:** Lets stages spill to disk beyond memory limits

100MB stage limit.

**Q99.** What is stage memory limit?

**Answer:** 100 MB per blocking stage by default

Plan for it.

**Q100.** What prevents index use in $match?

**Answer:** Placing it after stages that change fields

Order matters.

### [06 · Multi-Document Transactions](level-3/06-transactions.md)

**Q101.** What do multi-document transactions provide?

**Answer:** ACID across multiple documents/collections

Replica sets/shards.

**Q102.** When avoid transactions?

**Answer:** When a single-document atomic update suffices (embedding)

Cost.

**Q103.** What do transactions require?

**Answer:** Replica set (or sharded cluster) and a session

Setup.

**Q104.** What's a write conflict?

**Answer:** Concurrent transactions modifying the same document; one aborts and retries

Retry logic.

### [07 · Change Streams](level-3/07-change-streams.md)

**Q105.** What do change streams provide?

**Answer:** Real-time notifications of data changes

Event-driven.

**Q106.** What do they require?

**Answer:** A replica set or sharded cluster (oplog)

Backed by oplog.

**Q107.** What are resume tokens for?

**Answer:** Resuming a stream after a disconnect without missing events

Reliability.

**Q108.** Use case?

**Answer:** Syncing caches, triggering workflows and auditing

Reactive.

### [08 · Geospatial Queries](level-3/08-geospatial.md)

**Q109.** Which index supports geospatial queries?

**Answer:** 2dsphere

GeoJSON.

**Q110.** What does $near do?

**Answer:** Finds documents near a point, sorted by distance

Maxdistance.

**Q111.** What does $geoWithin do?

**Answer:** Finds documents inside a shape/polygon

Areas.

**Q112.** What coordinate order does GeoJSON use?

**Answer:** [longitude, latitude]

Common bug.

### [09 · Text Search &amp; Collation](level-3/09-text-search-collation.md)

**Q113.** What does a text index support?

**Answer:** Word-based search with stemming and ranking

$text, $meta textScore.

**Q114.** What is collation?

**Answer:** Language-specific string comparison and sorting rules

Case/accent.

**Q115.** What's Atlas Search?

**Answer:** Lucene-based search with relevance features

Richer.

**Q116.** When is regex slow?

**Answer:** Unanchored or case-insensitive patterns can't use indexes efficiently

Prefer text search.

### [10 · Project — Order Analytics Backend](level-3/10-project-order-analytics.md)

**Q117.** What does the order analytics project combine?

**Answer:** Aggregation stages, lookups, windows and performance tuning

Level 3 project.

**Q118.** How validate pipeline performance?

**Answer:** explain on aggregations and check index use

Evidence.

**Q119.** How present results?

**Answer:** Grouped totals, trends and top customers

Insight.

**Q120.** How keep it maintainable?

**Answer:** Named stages, comments and tests with sample data

Clarity.

## Level 4 · Operations & Production

### [01 · Replica Sets: Elections, the Oplog &amp; Failover](level-4/01-replica-sets.md)

**Q121.** What does a replica set provide?

**Answer:** High availability and redundancy via automatic failover

Elections.

**Q122.** How many voting members tolerate one failure?

**Answer:** 3 (a majority of 2 remain)

Odd numbers.

**Q123.** What's the oplog?

**Answer:** A capped collection of operations replicated to secondaries

Replication source.

**Q124.** What's an arbiter?

**Answer:** A voting member holding no data (generally avoid)

Prefer data-bearing.

### [02 · Write Concern, Read Concern &amp; Read Preference](level-4/02-read-write-concerns.md)

**Q125.** What does w: "majority" ensure?

**Answer:** Writes are acknowledged by a majority of voting members

Durability.

**Q126.** What does readConcern majority return?

**Answer:** Data acknowledged by a majority (won't roll back)

Consistency.

**Q127.** What's the risk of w:1?

**Answer:** Acknowledged writes can be lost after failover

Trade-off.

**Q128.** What does readPreference secondary risk?

**Answer:** Stale reads due to replication lag

Choose carefully.

### [03 · Sharding Architecture](level-4/03-sharding-architecture.md)

**Q129.** What does sharding do?

**Answer:** Distributes data across shards for horizontal scale

Scale out.

**Q130.** What are the main components?

**Answer:** Shards, mongos routers and config servers

Architecture.

**Q131.** What does mongos do?

**Answer:** Routes queries to the right shards

Client entry.

**Q132.** What do config servers store?

**Answer:** Cluster metadata and chunk ranges

Replica set.

### [04 · Choosing &amp; Changing a Shard Key](level-4/04-shard-keys.md)

**Q133.** What makes a good shard key?

**Answer:** High cardinality, even distribution and aligned with query patterns

Avoid hotspots.

**Q134.** Why is a monotonically increasing key bad?

**Answer:** All inserts hit one shard (hotspot)

Hashed keys help.

**Q135.** What's a hashed shard key for?

**Answer:** Even distribution at the cost of range queries

Trade-off.

**Q136.** Can you change the shard key?

**Answer:** Yes in recent versions (resharding), but it's costly

Choose carefully.

### [05 · Authentication, Roles &amp; TLS](level-4/05-security.md)

**Q137.** What does authentication verify?

**Answer:** Who the client is (SCRAM, x.509, LDAP)

Enable it.

**Q138.** What do roles control?

**Answer:** What authenticated users can do (RBAC)

Least privilege.

**Q139.** What does TLS protect?

**Answer:** Data in transit

Encrypt connections.

**Q140.** What protects data at rest?

**Answer:** Encrypted storage engine/disk encryption and KMS

Compliance.

### [06 · Backup, Restore &amp; Point-in-Time Recovery](level-4/06-backup-restore.md)

**Q141.** What backup methods exist?

**Answer:** Snapshots, mongodump and continuous cloud backup

Consistent.

**Q142.** Why test restores?

**Answer:** A backup is unproven until restored

Drills.

**Q143.** What does point-in-time recovery need?

**Answer:** Oplog/continuous backup to replay to a moment

Fine recovery.

**Q144.** What's a consistent backup?

**Answer:** A snapshot with the oplog or from a quiesced node

Avoid corruption.

### [07 · Monitoring &amp; the Profiler](level-4/07-monitoring-profiler.md)

**Q145.** What does the profiler record?

**Answer:** Slow operations above a threshold

db.setProfilingLevel.

**Q146.** Which metrics matter?

**Answer:** Opcounters, replication lag, cache usage and connections

Health.

**Q147.** What's replication lag?

**Answer:** How far a secondary is behind the primary

Alert.

**Q148.** What tools monitor?

**Answer:** Atlas, Ops Manager, Prometheus exporters

Dashboards.

### [08 · Performance Tuning: WiredTiger, Memory &amp; Connections](level-4/08-performance-tuning.md)

**Q149.** First step in tuning?

**Answer:** Find slow queries with the profiler and explain

Evidence.

**Q150.** What is the working set?

**Answer:** Frequently accessed data and indexes that should fit in RAM

Performance.

**Q151.** What reduces document size?

**Answer:** Shorter field names and avoiding unnecessary fields

Efficiency.

**Q152.** What helps write throughput?

**Answer:** Fewer indexes and bulk operations

Trade-off.

### [09 · Upgrades, FCV &amp; Online Schema Migrations](level-4/09-upgrades-migrations.md)

**Q153.** How upgrade a replica set?

**Answer:** Rolling: secondaries first, step down the primary, then upgrade it

Zero downtime.

**Q154.** What is featureCompatibilityVersion?

**Answer:** A setting gating new features until the whole cluster is upgraded

Safety.

**Q155.** What should precede upgrades?

**Answer:** Backups and testing in staging

Risk.

**Q156.** What helps migrating data?

**Answer:** mongodump/restore, live migration tools or change streams

Plan cutover.

### [10 · Capstone — A Secured Sharded Cluster](level-4/10-capstone-sharded-cluster.md)

**Q157.** What does the sharded cluster capstone combine?

**Answer:** Sharding, replication, security, monitoring and backups

Production.

**Q158.** How prove readiness?

**Answer:** Failure drills, load tests and restore tests

Evidence.

**Q159.** What documentation helps?

**Answer:** Topology, runbooks and access policies

Operate.

**Q160.** What sustains it?

**Answer:** Monitoring, capacity planning and regular upgrades

Lifecycle.

## Mixed questions across all levels

**Q161.** A document model fits when related data is read together and bounded. What does MongoDB recommend?

**Answer:** Embed it in one document; reference when data is unbounded or shared

Model for access patterns.

**Q162.** An array in a document grows forever with every event. What's the risk and fix?

**Answer:** The 16 MB document limit and slow updates; use the bucket pattern or references

Unbounded arrays.

**Q163.** How do you query a field inside an embedded document?

**Answer:** Dot notation, e.g. find({"address.city": "Oslo"})

Quotes are required.

**Q164.** You need an element that satisfies two conditions at once ( sku = pen AND qty &gt; 5). Which operator?

**Answer:** $elemMatch

Conditions apply to the same element.

**Q165.** What happens if `updateOne` is used with upsert: true and nothing matches?

**Answer:** A new document is inserted

Update-or-insert.

**Q166.** How do you atomically increment a counter?

**Answer:** $inc

No read-modify-write race.

**Q167.** Why use Decimal128 for prices?

**Answer:** Binary floating point can't represent decimal fractions exactly

Avoid rounding errors.

**Q168.** A query is slow and explain shows COLLSCAN with docsExamined 1,000,000 and nReturned 10. What now?

**Answer:** Add an index matching the query (equality, sort, range)

Selectivity.

**Q169.** A compound index {status:1, createdAt:-1} serves which query best?

**Answer:** Filter status = x sorted by createdAt descending

Equality then sort.

**Q170.** Which index automatically removes old documents?

**Answer:** A TTL index

expireAfterSeconds.

**Q171.** A pipeline filters with $match after a $group. How can it be faster?

**Answer:** Move $match first so it can use an index and reduce documents

Filter early.

**Q172.** What does $lookup do and what should be indexed?

**Answer:** A left outer join; index the foreign field to avoid repeated scans

Or embed to avoid joins.

**Q173.** When do you need a multi-document transaction?

**Answer:** When several documents/collections must change atomically (single-document updates are already atomic)

Embedding often avoids them.

**Q174.** A service must react to changes in a collection in real time. What fits?

**Answer:** Change streams (with resume tokens)

Requires a replica set.

**Q175.** Which write concern avoids losing acknowledged writes after a failover?

**Answer:** w: "majority"

Durability trade-off.

**Q176.** Why avoid a monotonically increasing shard key?

**Answer:** All new inserts hit one shard (a hotspot)

Hashed keys spread writes.

**Q177.** Which shard key trait is essential?

**Answer:** High cardinality and even distribution aligned with queries

Avoid jumbo chunks.

**Q178.** A replica set has 3 voting members. How many failures can it tolerate and still elect a primary?

**Answer:** One (a majority of two must remain)

Use odd numbers.

**Q179.** How do you know a backup works?

**Answer:** Restore it regularly in a drill

Unrestored backups are unproven.

## Practical coding questions

### A mini query and aggregation engine

Implement matches (dot paths, comparison operators, $in, $elemMatch and array containment) and aggregate with $match, $unwind, $group ($sum, $max), $sort and $limit. The script exercises them on four orders.

Starter code:

```
import operator

OPS = {"$gt": operator.gt, "$gte": operator.ge, "$lt": operator.lt, "$lte": operator.le,
       "$eq": operator.eq, "$ne": operator.ne}

def get(doc, path):
    for part in path.split("."):
        if isinstance(doc, dict) and part in doc:
            doc = doc[part]
        else:
            return None
    return doc

def matches(doc, query):
    # supports equality, dot paths (via get), comparison operators, $in, $elemMatch, and array containment
    # ({tags: "gift"} matches when "gift" is in the array). Return True/False.
    return False

def aggregate(docs, pipeline):
    # support stages $match, $unwind, $group ($sum of 1 or a field, $max), $sort and $limit
    return docs

orders = [
    {"_id": 1, "customer": "ada", "status": "paid", "items": [{"sku": "pen", "qty": 3}, {"sku": "book", "qty": 1}], "total": 24.5, "tags": ["gift"]},
    {"_id": 2, "customer": "bo", "status": "paid", "items": [{"sku": "pen", "qty": 10}], "total": 15.0, "tags": []},
    {"_id": 3, "customer": "ada", "status": "refunded", "items": [{"sku": "book", "qty": 2}], "total": 40.0, "tags": ["gift", "late"]},
    {"_id": 4, "customer": "cy", "status": "paid", "items": [{"sku": "lamp", "qty": 1}], "total": 60.0, "tags": []},
]
print([d["_id"] for d in orders if matches(d, {"status": "paid", "total": {"$gte": 20}})])
print([d["_id"] for d in orders if matches(d, {"tags": "gift"})])
print([d["_id"] for d in orders if matches(d, {"items": {"$elemMatch": {"sku": "pen", "qty": {"$gt": 5}}}})])
print(aggregate(orders, [{"$match": {"status": "paid"}}, {"$group": {"_id": "$customer", "spent": {"$sum": "$total"}}}, {"$sort": {"spent": -1}}]))
lines = aggregate(orders, [{"$unwind": "$items"}])
flat = [{"sku": d["items"]["sku"], "qty": d["items"]["qty"]} for d in lines]
print(aggregate(flat, [{"$group": {"_id": "$sku", "units": {"$sum": "$qty"}}}, {"$sort": {"units": -1, "_id": 1}}, {"$limit": 2}]))
```

Expected output:

```text
[1, 4]
[1, 3]
[2]
[{'_id': 'cy', 'spent': 60.0}, {'_id': 'ada', 'spent': 24.5}, {'_id': 'bo', 'spent': 15.0}]
[{'_id': 'pen', 'units': 13}, {'_id': 'book', 'units': 3}]
```

??? success "Show a solution"

    ```
    import operator

    OPS = {"$gt": operator.gt, "$gte": operator.ge, "$lt": operator.lt, "$lte": operator.le,
           "$eq": operator.eq, "$ne": operator.ne}

    def get(doc, path):
        for part in path.split("."):
            if isinstance(doc, dict) and part in doc:
                doc = doc[part]
            else:
                return None
        return doc

    def matches(doc, query):
        for field, cond in query.items():
            value = get(doc, field)
            if isinstance(cond, dict):
                for op, arg in cond.items():
                    if op == "$in":
                        ok = value in arg
                    elif op == "$elemMatch":
                        ok = isinstance(value, list) and any(matches(e, arg) for e in value)
                    else:
                        ok = value is not None and OPS[op](value, arg)
                    if not ok:
                        return False
            elif isinstance(value, list):
                if cond not in value:                      # array containment, like {tags: "x"}
                    return False
            elif value != cond:
                return False
        return True

    def aggregate(docs, pipeline):
        for stage in pipeline:
            (name, spec), = stage.items()
            if name == "$match":
                docs = [d for d in docs if matches(d, spec)]
            elif name == "$unwind":
                field = spec.lstrip("$")
                docs = [{**d, field: v} for d in docs for v in d.get(field, [])]
            elif name == "$group":
                groups = {}
                for d in docs:
                    key = d[spec["_id"].lstrip("$")]
                    g = groups.setdefault(key, {"_id": key})
                    for out, acc in spec.items():
                        if out == "_id":
                            continue
                        (op, arg), = acc.items()
                        if op == "$sum":
                            g[out] = g.get(out, 0) + (1 if arg == 1 else d[arg.lstrip("$")])
                        elif op == "$max":
                            v = d[arg.lstrip("$")]; g[out] = v if out not in g else max(g[out], v)
                docs = list(groups.values())
            elif name == "$sort":
                for field, direction in reversed(list(spec.items())):
                    docs = sorted(docs, key=lambda d: d[field], reverse=direction < 0)
            elif name == "$limit":
                docs = docs[:spec]
        return docs

    orders = [
        {"_id": 1, "customer": "ada", "status": "paid", "items": [{"sku": "pen", "qty": 3}, {"sku": "book", "qty": 1}], "total": 24.5, "tags": ["gift"]},
        {"_id": 2, "customer": "bo", "status": "paid", "items": [{"sku": "pen", "qty": 10}], "total": 15.0, "tags": []},
        {"_id": 3, "customer": "ada", "status": "refunded", "items": [{"sku": "book", "qty": 2}], "total": 40.0, "tags": ["gift", "late"]},
        {"_id": 4, "customer": "cy", "status": "paid", "items": [{"sku": "lamp", "qty": 1}], "total": 60.0, "tags": []},
    ]
    print([d["_id"] for d in orders if matches(d, {"status": "paid", "total": {"$gte": 20}})])
    print([d["_id"] for d in orders if matches(d, {"tags": "gift"})])
    print([d["_id"] for d in orders if matches(d, {"items": {"$elemMatch": {"sku": "pen", "qty": {"$gt": 5}}}})])
    print(aggregate(orders, [{"$match": {"status": "paid"}}, {"$group": {"_id": "$customer", "spent": {"$sum": "$total"}}}, {"$sort": {"spent": -1}}]))
    lines = aggregate(orders, [{"$unwind": "$items"}])
    flat = [{"sku": d["items"]["sku"], "qty": d["items"]["qty"]} for d in lines]
    print(aggregate(flat, [{"$group": {"_id": "$sku", "units": {"$sum": "$qty"}}}, {"$sort": {"units": -1, "_id": 1}}, {"$limit": 2}]))
    ```

    MongoDB's query language is a set of predicates over documents and arrays, and aggregation is a pipeline of stages each transforming the document list. This toy engine mirrors the semantics, not the implementation. Verified by running the solution.

### Why shard-key choice matters

Implement shard_for for hashed and ranged strategies and compare how historical and new monotonically increasing keys distribute across three shards. Observe the hotspot with ranged keys and the even spread with hashed keys.

Starter code:

```
import hashlib
from collections import Counter

def shard_for(key, shards, strategy, ranges=None):
    # "hashed": md5 of str(key) as an int, modulo shards; "ranged": index of the first upper bound
    # greater than key (the last shard when above every bound); ValueError for unknown strategies
    return 0

def distribution(keys, shards, strategy, ranges=None):
    c = Counter(shard_for(k, shards, strategy, ranges) for k in keys)
    return [c.get(i, 0) for i in range(shards)]

def imbalance(counts):
    return round(max(counts) / (sum(counts) / len(counts)), 2)

N = 3000
monotonic = list(range(N))                      # an ever-increasing key, e.g. a timestamp or counter
ranges = [1000, 2000]                           # 3 chunks created from the initial data
print("ranged, historical data:", distribution(monotonic, 3, "ranged", ranges))
new_inserts = list(range(N, N + 900))           # new writes are all above the highest range
print("ranged, new inserts:    ", distribution(new_inserts, 3, "ranged", ranges), "-> hotspot")
print("hashed, new inserts:    ", distribution(new_inserts, 3, "hashed"))
print("imbalance ranged/hashed:", imbalance(distribution(new_inserts, 3, "ranged", ranges)), imbalance(distribution(new_inserts, 3, "hashed")))
status = ["paid"] * 2900 + ["refunded"] * 100   # a low-cardinality key can't spread
print("low cardinality key:", sorted(set(status)), "-> at most", len(set(status)), "chunks")
```

Expected output:

```text
ranged, historical data: [1000, 1000, 1000]
ranged, new inserts:     [0, 0, 900] -> hotspot
hashed, new inserts:     [285, 293, 322]
imbalance ranged/hashed: 3.0 1.07
low cardinality key: ['paid', 'refunded'] -> at most 2 chunks
```

??? success "Show a solution"

    ```
    import hashlib
    from collections import Counter

    def shard_for(key, shards, strategy, ranges=None):
        if strategy == "hashed":
            h = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
            return h % shards
        if strategy == "ranged":
            for i, upper in enumerate(ranges):
                if key < upper:
                    return i
            return len(ranges)
        raise ValueError(strategy)

    def distribution(keys, shards, strategy, ranges=None):
        c = Counter(shard_for(k, shards, strategy, ranges) for k in keys)
        return [c.get(i, 0) for i in range(shards)]

    def imbalance(counts):
        return round(max(counts) / (sum(counts) / len(counts)), 2)

    N = 3000
    monotonic = list(range(N))                      # an ever-increasing key, e.g. a timestamp or counter
    ranges = [1000, 2000]                           # 3 chunks created from the initial data
    print("ranged, historical data:", distribution(monotonic, 3, "ranged", ranges))
    new_inserts = list(range(N, N + 900))           # new writes are all above the highest range
    print("ranged, new inserts:    ", distribution(new_inserts, 3, "ranged", ranges), "-> hotspot")
    print("hashed, new inserts:    ", distribution(new_inserts, 3, "hashed"))
    print("imbalance ranged/hashed:", imbalance(distribution(new_inserts, 3, "ranged", ranges)), imbalance(distribution(new_inserts, 3, "hashed")))
    status = ["paid"] * 2900 + ["refunded"] * 100   # a low-cardinality key can't spread
    print("low cardinality key:", sorted(set(status)), "-> at most", len(set(status)), "chunks")
    ```

    With a ranged, ever-increasing key every new insert lands in the last chunk and one shard takes all the writes, while a hashed key spreads them evenly (at the cost of efficient range queries). A low-cardinality key can't be split into more chunks than it has values. Verified by running the solution.

---

More free practice: the [MongoDB Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
