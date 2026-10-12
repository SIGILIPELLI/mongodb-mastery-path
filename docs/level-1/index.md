---
description: "MongoDB Level 1: a local lab, mongosh, BSON types, CRUD with query and update operators, arrays, bulk writes, PyMongo and Node drivers, and the data tools."
---

# Level 1 · MongoDB Foundations

Level 1 gets you from "I've heard MongoDB stores JSON" to writing correct application code against it.
That takes more than learning `find()` and `insertOne()`. It means knowing what a document really is
(BSON, with types JSON doesn't have), why `{price: 0.1}` adds up wrong, why a filter on an array matched
a student who failed the test, why a read-then-write counter lost 80% of its increments, and why a
`mongorestore` "succeeded" without restoring anything.

Every lesson is built from commands run against a real server — MongoDB 9.0.2 — and every output shown
is what that server returned, errors included.

**Prerequisites:** basic programming in Python or JavaScript, and comfort at a terminal. No database
experience is required, though if you know SQL you'll see where the two models differ.

## Modules

1. **[How MongoDB Is Built: mongod, Storage & a Local Lab](01-architecture-and-setup.md)** — install
   from the official tarball, start a throwaway server, and look inside its data directory, log and
   cache.
2. **[mongosh Like a Pro](02-mongosh-like-a-pro.md)** — connection strings, cursors and batches,
   scripts, JSON output modes, numbers, errors and `.mongoshrc.js`.
3. **[Documents, BSON & Data Types](03-documents-bson-types.md)** — ObjectId anatomy, Decimal128 for
   money, real dates, null vs missing, mixed-type comparisons and the 16 MB limit.
4. **[Inserting & Querying Documents](04-insert-and-query.md)** — filters, comparison and logical
   operators, projection, sort/skip/limit, counts and `$expr`.
5. **[Querying Arrays & Embedded Documents](05-arrays-embedded-queries.md)** — "contains" semantics,
   `$all`, `$size`, why `$elemMatch` matters, and trimming arrays in projections.
6. **[Update Operators, Upserts & Array Updates](06-update-operators.md)** — atomic `$inc` vs lost
   updates, `$push` with `$slice`, positional and filtered updates, upserts, `findOneAndUpdate`.
7. **[Deletes, Replaces & Bulk Writes](07-deletes-replaces-bulk.md)** — safe deletes, `drop()` vs
   `deleteMany({})`, ordered vs unordered batches and `bulkWrite`.
8. **[Drivers: PyMongo & the Node.js Driver](08-drivers.md)** — one shared client, connection pools,
   type mapping, timeouts, errors and retryable writes.
9. **[mongoimport, mongoexport, mongodump & mongorestore](09-import-export-dump.md)** — typed CSV
   imports, Extended JSON, archives and namespace remapping.
10. **[Project — Recipe Box Data Layer](10-project-recipe-box.md)** — a tested Python data layer with
    unique slugs, atomic ratings, array updates and keyset paging.

## After Level 1

You can store and change data correctly. Level 2 is about doing it *well*: deciding what to embed and
what to reference, enforcing a schema in a "schemaless" database, and designing the indexes that turn
the collection scans you've been running into fast, targeted reads — with `explain()` to prove it.
