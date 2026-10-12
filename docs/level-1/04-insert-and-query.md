---
description: "Insert documents and query them with find(): comparison and logical operators, regex, projection rules, sort/skip/limit, counts, distinct and $expr."
---

# 04 · Inserting & Querying Documents

This lesson covers the everyday half of MongoDB: putting documents in and getting the right ones
back out. If you know SQL (the [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
covers it), most of the ideas map directly — `find()` is `SELECT`, the filter is `WHERE`, the
projection is the column list — but the syntax is a document, and a few behaviours differ in ways
that matter.

## A sample collection

Save this as `books.js` and run `mongosh --quiet books.js`. The rest of the lesson queries it.

```javascript
const lib = db.getSiblingDB("library");
lib.books.drop();
lib.books.insertMany([
  { _id: 1, title: "The Pragmatic Programmer", author: "Hunt & Thomas", year: 1999, pages: 352, genres: ["software", "career"], rating: 4.6, inStock: true },
  { _id: 2, title: "Designing Data-Intensive Applications", author: "Kleppmann", year: 2017, pages: 616, genres: ["software", "databases"], rating: 4.8, inStock: true },
  { _id: 3, title: "Dune", author: "Herbert", year: 1965, pages: 688, genres: ["sci-fi"], rating: 4.5, inStock: false },
  { _id: 4, title: "Project Hail Mary", author: "Weir", year: 2021, pages: 496, genres: ["sci-fi"], rating: 4.7, inStock: true },
  { _id: 5, title: "The Martian", author: "Weir", year: 2011, pages: 384, genres: ["sci-fi"], rating: 4.6, inStock: true },
  { _id: 6, title: "Clean Code", author: "Martin", year: 2008, pages: 464, genres: ["software"], rating: 4.2, inStock: false },
  { _id: 7, title: "Database Internals", author: "Petrov", year: 2019, pages: 370, genres: ["databases"], rating: 4.5, inStock: true },
  { _id: 8, title: "Foundation", author: "Asimov", year: 1951, pages: 255, genres: ["sci-fi"], rating: 4.3, inStock: true },
  { _id: 9, title: "Refactoring", author: "Fowler", year: 2018, pages: 448, genres: ["software"], rating: 4.4 },
  { _id: 10, title: "Thinking in Systems", author: "Meadows", year: 2008, pages: 240, genres: ["systems"], rating: 4.3, inStock: true }
]);
```

The ratings and page counts are illustrative values for practice, not authoritative data. Note that
book 9 has no `inStock` field at all — real collections are like that, and it will matter below.

## Inserting

```javascript
const r = db.books.insertOne({ title: "Snow Crash", author: "Stephenson", year: 1992, genres: ["sci-fi"] });
r
```

```text
{ acknowledged: true, insertedId: ObjectId('6acbc6dc40661aefb428093d') }
```

Every document has an `_id`, unique within its collection. If you don't supply one, the **driver**
generates an ObjectId before sending the insert — which is why `insertedId` comes back immediately.
Supply your own `_id` when the data has a natural unique key (an ISBN, an order number, a username):
it saves an index and makes lookups by that key free.

`insertMany()` sends a batch in one round trip. What happens when one document in the batch fails is
covered in lesson 07.

## Filters: equality and comparison

A filter is a document. Each field is a condition; several fields are combined with AND.

```javascript
db.books.find({ year: { $gte: 2015 } }, { _id: 0, title: 1, year: 1 }).sort({ year: -1 })
```

```text
[
  { title: 'Project Hail Mary', year: 2021 },
  { title: 'Database Internals', year: 2019 },
  { title: 'Refactoring', year: 2018 },
  { title: 'Designing Data-Intensive Applications', year: 2017 }
]
```

| Operator | Meaning | Example |
|----------|---------|---------|
| (none) | equals | `{ author: "Weir" }` |
| `$eq`, `$ne` | equals / not equals | `{ inStock: { $ne: true } }` |
| `$gt`, `$gte`, `$lt`, `$lte` | ranges | `{ pages: { $lt: 300 } }` |
| `$in`, `$nin` | in / not in a list | `{ author: { $in: ["Weir", "Asimov"] } }` |
| `$exists` | field present or not | `{ inStock: { $exists: false } }` |
| `$type` | BSON type | `{ rating: { $type: "double" } }` |
| `$regex` | pattern match | `{ title: /^the /i }` |

Two conditions on different fields:

```javascript
db.books.find({ author: { $in: ["Weir", "Asimov"] }, pages: { $lt: 450 } }, { _id: 0, title: 1 })
```

```text
[ { title: 'The Martian' }, { title: 'Foundation' } ]
```

Two conditions on the **same** field go in one operator document:

```javascript
db.books.find({ year: { $gte: 2000, $lt: 2010 } }, { _id: 0, title: 1, year: 1 })
```

```text
[ { title: 'Clean Code', year: 2008 }, { title: 'Thinking in Systems', year: 2008 } ]
```

## The duplicate-key trap

This looks like "between 2000 and 1960" (which should match nothing) but it is not:

```javascript
db.books.find({ year: { $gte: 2000 }, year: { $lt: 1960 } }, { _id: 0, title: 1, year: 1 })
```

```text
[ { title: 'Foundation', year: 1951 } ]
```

A JavaScript (or Python, or JSON) object can't have two `year` keys — the second silently replaces
the first, and the server only ever receives `{ year: { $lt: 1960 } }`. Merge conditions on one field
into one operator document, or use an explicit `$and` array.

## Logical operators

```javascript
db.books.find(
  { $or: [{ rating: { $gte: 4.7 } }, { pages: { $lt: 250 } }] },
  { _id: 0, title: 1, rating: 1, pages: 1 }
)
```

```text
[
  { title: 'Designing Data-Intensive Applications', pages: 616, rating: 4.8 },
  { title: 'Project Hail Mary', pages: 496, rating: 4.7 },
  { title: 'Thinking in Systems', pages: 240, rating: 4.3 }
]
```

`$and`, `$or` and `$nor` take arrays of filters; `$not` negates one operator expression. When every
branch of an `$or` tests the *same* field for equality, use `$in` instead — it is shorter and easier
for the planner.

## Negation includes missing fields

```javascript
db.books.find({ inStock: { $ne: true } }, { _id: 0, title: 1, inStock: 1 })
```

```text
[
  { title: 'Dune', inStock: false },
  { title: 'Clean Code', inStock: false },
  { title: 'Refactoring' }
]
```

`$ne` and `$nin` match documents where the field is **missing**, too. Refactoring has no `inStock`
field, so it is "not equal to true". Sometimes that is what you want; often it is a bug in a report.
Write `{ inStock: false }` when you mean "explicitly false".

## Regular expressions

```javascript
db.books.find({ title: /^the /i }, { _id: 0, title: 1 })
```

```text
[ { title: 'The Pragmatic Programmer' }, { title: 'The Martian' } ]
```

Only a case-sensitive regex anchored with `^` can use an index efficiently (as a prefix range).
Case-insensitive or unanchored patterns must examine every candidate value. For real search, see
Level 3 · 09.

## Comparing fields with each other: $expr

Ordinary filters compare a field to a constant. To compare two fields of the same document, use
`$expr` with aggregation expressions:

```javascript
db.books.find({ $expr: { $gt: ["$pages", { $multiply: ["$rating", 100] }] } }, { _id: 0, title: 1 })
```

```text
[
  { title: 'Designing Data-Intensive Applications' },
  { title: 'Dune' },
  { title: 'Project Hail Mary' },
  { title: 'Clean Code' },
  { title: 'Refactoring' }
]
```

`$expr` is powerful but usually can't use an index for the comparison itself. Use it for filters that
genuinely need it, not as your default syntax.

## Projection

The second argument chooses which fields come back. Either **include** fields (`{title: 1}`) or
**exclude** them (`{reviews: 0}`) — not both, with one exception: `_id` is included by default and
may always be excluded.

```javascript
db.books.find({}, { title: 1, year: 0 })
```

```text
31254 Cannot do exclusion on field year in inclusion projection
```

Projection reduces what crosses the network and what your application has to deserialize. It does
**not** make the server read less from disk unless an index alone can answer the query (a *covered*
query, Level 2 · 07).

## Sort, skip and limit

```javascript
db.books.find({}, { _id: 0, title: 1 }).sort({ _id: 1 }).skip(3).limit(2)
```

```text
[ { title: 'Project Hail Mary' }, { title: 'The Martian' } ]
```

The server always applies them in the order sort → skip → limit, no matter what order you chain
them. Without `sort()`, the order of results is **not defined** — it often looks like insertion order
on a small test collection and then changes after updates, on a different server, or when the plan
changes. If order matters, sort, and include a unique field (like `_id`) as the last sort key so ties
are broken deterministically.

`skip()` still walks over every skipped document, so `skip(100000)` for page 5,001 of a list is slow.
For deep paging use a **range** on the sort key instead ("give me 20 books with `_id` greater than the
last one I saw").

## Counting and distinct values

```javascript
db.books.countDocuments({ inStock: { $ne: true } })   // 3 — runs the query
db.books.estimatedDocumentCount()                      // 10 — reads collection metadata
db.books.distinct("genres")
```

```text
3
10
[ 'career', 'databases', 'sci-fi', 'software', 'systems' ]
```

`countDocuments()` runs a real query and is exact. `estimatedDocumentCount()` returns the count kept
in the collection's metadata without scanning — instant on a billion documents, but it takes no
filter and can be slightly off after an unclean shutdown. `distinct()` unwinds arrays, so each genre
appears once even though `genres` is an array.

`findOne()` returns the first matching document (or `null`) instead of a cursor:

```javascript
db.books.findOne({ author: "Weir" }, { title: 1 })
```

```text
{ _id: 4, title: 'Project Hail Mary' }
```

## How It Actually Works

When a `find` command arrives, the server **parses and normalizes** the filter into a tree of match
expressions — `{year: {$gte: 2000, $lt: 2010}}` becomes an AND node with two leaf comparisons. The
**query planner** then looks for indexes that could satisfy some of those leaves. Here there are none
except `_id`, so the only possible plan is a **collection scan** (`COLLSCAN`): read every document,
evaluate the tree against it, and keep the ones that pass.

Matching documents flow through a pipeline of execution stages: `COLLSCAN` → `SORT` (which must see
all matches before it can return the first one, and has a memory limit) → `SKIP` → `LIMIT` →
`PROJECTION`. Results are packed into a batch of up to 16 MiB (or 101 documents for the first batch
by default) and returned with a cursor id if there are more.

Note what this means for `sort` + `limit`: without an index on the sort field, finding the top 10
still requires examining every matching document — the server keeps only the best 10 in memory as it
goes (a "top-k" sort), but it cannot avoid reading them all. Level 2 is about the indexes that turn
these full scans into small, targeted reads, and `explain()` (Level 2 · 07) shows you which stages a
query actually used.

## Common mistakes

- **Repeating a key in a filter object** — the last one silently wins.
- **Forgetting that `$ne`/`$nin` match missing fields.**
- **Relying on natural order.** Always `sort()` when order matters, with a unique tie-breaker.
- **Mixing inclusion and exclusion in a projection.**
- **Deep pagination with `skip()`.** Use range-based ("keyset") pagination.
- **Using `countDocuments({})` for a dashboard total on a huge collection** when an estimate would do.
- **Unanchored or case-insensitive regex for search boxes.** They scan; use a text or search index.

## Exercise

Using the `library.books` collection:

1. Find the titles of in-stock books with more than 400 pages, sorted by pages descending.
2. Find books in the `sci-fi` genre published before 1970 *or* rated at least 4.7 (hint: an AND of
   one condition and an `$or`).
3. Count books where `inStock` is explicitly `false`, then count books where it is "not true". Explain
   the difference in the numbers.
4. Implement keyset pagination: write a loop that prints the books two at a time ordered by `_id`,
   using `{_id: {$gt: lastId}}` instead of `skip()`.
5. Use `$expr` to find books whose title is longer than 15 characters (`$strLenCP`).
