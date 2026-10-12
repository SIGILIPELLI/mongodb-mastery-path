---
description: "Model one-to-one, one-to-few, one-to-many, one-to-squillions, many-to-many and tree relationships in MongoDB, with the queries and indexes each needs."
---

# 02 · Modeling Relationships

Lesson 01 gave you the embed-or-reference trade-off. This lesson turns it into concrete shapes for
every kind of relationship you will meet, with the query that reads each shape and the index it needs.
The cardinality labels are borrowed from relational modeling, but MongoDB adds a distinction that matters
more than "one" versus "many": **how many is many?** "A few", "hundreds" and "unbounded" lead to three
different designs.

## One-to-one

A user and their profile settings. Almost always: **embed**.

```javascript
{ _id: "u42", email: "ana@example.com",
  settings: { theme: "dark", locale: "en-IN", emailDigest: "weekly" } }
```

Split it out only when the embedded part is large and rarely read (a 200 KB avatar thumbnail, a
verbose audit blob) — then a separate collection keyed by the same `_id` keeps the hot document small.

## One-to-few: embed an array

A person's addresses, a product's handful of variants, an order's line items.

```javascript
{ _id: "u42", addresses: [
    { label: "home", city: "Pune", zip: "411001" },
    { label: "office", city: "Pune", zip: "411014" } ] }
```

The list is bounded by reality, read with the parent, and changing it atomically with the parent is
useful. Query into it with dot notation and `$elemMatch` (Level 1 · 05); update it with positional
operators (Level 1 · 06).

## One-to-many: reference from the child

A customer's orders, a post's comments, a course's enrolments: potentially thousands, each queried on its
own, growing over time. Store the parent's id **in the child**:

```javascript
// orders
{ _id: ObjectId("..."), customerId: "u42", placedAt: ISODate("2026-10-01"), total: NumberDecimal("59.90") }
```

```javascript
db.orders.createIndex({ customerId: 1, placedAt: -1 })
db.orders.find({ customerId: "u42" }).sort({ placedAt: -1 }).limit(20)
```

The compound index serves "this customer's latest orders" directly. Do **not** also store an array of
order ids in the customer: it's an unbounded array (lesson 01), and keeping it in sync is a second write
that can fail.

## One-to-squillions

Log lines from a host, sensor readings from a device, events in an analytics stream: millions per parent.
The design is the same as one-to-many — child references parent — but now the child collection's volume
dominates everything. You will want time series collections or the bucket pattern (lesson 03 and lesson
09), a TTL or archival policy (lesson 06), and, at scale, sharding (Level 4).

## Many-to-many: arrays of references

Books and authors. Most books have one to three authors (bounded), while a prolific author may have
dozens of books. So store the references on the **bounded** side:

```javascript
db.authors.insertMany([{ _id: "kl", name: "Martin Kleppmann" }, { _id: "pe", name: "Alex Petrov" },
                       { _id: "co", name: "Co Author" }]);
db.books.insertMany([
  { _id: "ddia", title: "Designing Data-Intensive Applications", authorIds: ["kl"] },
  { _id: "dbi", title: "Database Internals", authorIds: ["pe"] },
  { _id: "joint", title: "A Joint Work", authorIds: ["kl", "co"] }]);
db.books.createIndex({ authorIds: 1 });
```

Both directions are now a single indexed query. Books by an author ("array contains"):

```javascript
db.books.find({ authorIds: "kl" }, { _id: 0, title: 1 })
```

```text
[ { title: 'Designing Data-Intensive Applications' }, { title: 'A Joint Work' } ]
```

Authors of a book — fetch the book, then `$in` its ids:

```javascript
const b = db.books.findOne({ _id: "joint" });
db.authors.find({ _id: { $in: b.authorIds } }, { _id: 0, name: 1 })
```

```text
[ { name: 'Co Author' }, { name: 'Martin Kleppmann' } ]
```

Notice the order: the book lists `["kl", "co"]` but the authors came back as Co, then Martin. `$in` does
**not** preserve the order of the list you give it. If author order matters (it does on a book cover),
reorder the results in your code using the original array.

Or do the join on the server with `$lookup` (Level 3 · 03 covers it fully):

```javascript
db.books.aggregate([
  { $match: { _id: "joint" } },
  { $lookup: { from: "authors", localField: "authorIds", foreignField: "_id", as: "authors" } },
  { $project: { _id: 0, title: 1, "authors.name": 1 } }])
```

```text
[ { title: 'A Joint Work', authors: [ { name: 'Co Author' }, { name: 'Martin Kleppmann' } ] } ]
```

Same caveat: the joined array isn't in `authorIds` order either.

When **both** sides can be large — students and courses on a platform with thousands of each — don't
store arrays on either side. Use a separate **link collection** (`enrolments: {studentId, courseId,
enrolledAt, progress}`) with an index for each direction. That is also where data *about* the
relationship (enrolment date, progress, grade) naturally lives.

## Denormalising a little

If book listings always show author names, a second query per page is a cost you pay millions of times
to avoid a cost (renaming an author) you pay almost never. Embed a small **extended reference**:

```javascript
{ _id: "joint", title: "A Joint Work",
  authors: [ { _id: "kl", name: "Martin Kleppmann" }, { _id: "co", name: "Co Author" } ] }
```

The `_id` still links to the full author document; the name is a copy for display. When an author is
renamed, an `updateMany({"authors._id": "kl"}, {$set: {"authors.$[a].name": ...}}, {arrayFilters: ...})`
fixes the copies — rarely, and in the background.

## Trees

Categories, org charts, folder hierarchies, threaded comments. Four standard shapes, each optimising a
different question:

| Shape | Stores | Cheap | Expensive |
|-------|--------|-------|-----------|
| Parent reference | `parent` | children of a node; moving a node | all descendants; the full path |
| Child references | `children: [...]` | children of a node | parents; unbounded for wide trees |
| Array of ancestors | `parent`, `ancestors: [...]` | all descendants, breadcrumbs | moving a subtree |
| Materialized path | `path: ",a,b,c,"` | descendants via prefix, sorting by path | moving a subtree |

The ancestors array covers the questions most applications ask. Six categories:

```javascript
db.cats.insertMany([
  { _id: "electronics", parent: null, ancestors: [], path: ",electronics," },
  { _id: "computers", parent: "electronics", ancestors: ["electronics"], path: ",electronics,computers," },
  { _id: "laptops", parent: "computers", ancestors: ["electronics", "computers"], path: ",electronics,computers,laptops," },
  { _id: "gaming-laptops", parent: "laptops", ancestors: ["electronics", "computers", "laptops"], path: ",electronics,computers,laptops,gaming-laptops," },
  { _id: "phones", parent: "electronics", ancestors: ["electronics"], path: ",electronics,phones," },
  { _id: "kitchen", parent: null, ancestors: [], path: ",kitchen," }]);
db.cats.createIndex({ ancestors: 1 });
db.cats.createIndex({ path: 1 });
```

```text
children of electronics: computers,phones
all under computers: laptops,gaming-laptops
breadcrumb: electronics > computers > laptops
```

Those are `find({parent: "electronics"})`, `find({ancestors: "computers"})` and reading one document's
`ancestors`. "Everything under computers" is a single indexed lookup at any depth.

The materialized path answers the same descendant question with an anchored regex:

```javascript
db.cats.find({ path: /^,electronics,computers,/ })
```

```text
path prefix: computers,laptops,gaming-laptops
```

Its `explain()` showed an `IXSCAN` with 4 keys examined and 3 documents fetched — a case-sensitive regex
anchored with `^` becomes a range scan on the index. The leading and trailing commas stop `,phone`
from matching `,phones,`.

### Moving a subtree

The price of both shapes is moving a node: every descendant's `ancestors` and `path` change. Move
`laptops` from Computers to a new Portables category — one update for the node, one pipeline update for
its descendants:

```javascript
const node = db.cats.findOne({ _id: "laptops" }), newParent = db.cats.findOne({ _id: "portables" });
const newAnc = [...newParent.ancestors, newParent._id];
const oldPrefix = node.path, newPrefix = newParent.path + node._id + ",";
db.cats.updateOne({ _id: "laptops" }, { $set: { parent: "portables", ancestors: newAnc, path: newPrefix } });
db.cats.updateMany({ ancestors: "laptops" }, [{ $set: {
  ancestors: { $concatArrays: [newAnc, ["laptops"],
    { $slice: ["$ancestors", { $add: [{ $indexOfArray: ["$ancestors", "laptops"] }, 1] }, 100] }] },
  path: { $concat: [newPrefix, { $substrCP: ["$path", oldPrefix.length, 1000] }] } } }]);
```

```text
laptops | electronics>portables | ,electronics,portables,laptops,
gaming-laptops | electronics>portables>laptops | ,electronics,portables,laptops,gaming-laptops,
```

For a category tree edited a few times a year that's fine. For a tree reorganised constantly (a
drag-and-drop folder UI with millions of files), parent references plus `$graphLookup` (Level 3 · 03)
may be the better trade. And note that the node update and the descendants update are two separate
writes: a reader between them sees an inconsistent tree. Wrap them in a transaction if that matters.

## How It Actually Works

Every relationship query above is fast for one reason: it is an **equality or prefix lookup on an
index**. `{customerId: "u42"}`, `{authorIds: "kl"}`, `{ancestors: "computers"}` and the anchored path
regex all translate into a bounded range of keys in a B-tree index, so the cost depends on the number of
matching documents, not on the size of the collection.

Array fields get special treatment in indexes: an index on `authorIds` or `ancestors` is **multikey** —
it stores one index entry per array element, all pointing to the same document. That is what makes
"which books contain author kl?" a direct lookup rather than a scan, and it's why the
ancestors-array tree works at any depth: `gaming-laptops` has three index entries, one for each ancestor.
The cost is proportional: a document with 1,000 array elements has 1,000 index entries to maintain on
every write — another reason references belong on the *bounded* side.

`$lookup` is a join executed inside the server: for each input document it runs a query against the
foreign collection (using an index on the `foreignField` when one exists). It saves round trips, not
work. A schema that needs `$lookup` on every request of its hottest path is usually a sign that some
data should be embedded or denormalised.

## Common mistakes

- **Storing child ids in the parent for unbounded relationships** (`customer.orderIds`).
- **Storing arrays of references on the unbounded side** of a many-to-many.
- **Forgetting the index on the reference field** — every "find children" becomes a collection scan.
- **Assuming `$in` or `$lookup` return results in your array's order.**
- **Unanchored or case-insensitive regex on a materialized path** — it can't use the index as a range.
- **Moving tree nodes without updating descendants**, or doing it in two steps that readers can observe
  half-done.

## Exercise

1. Model a Q&A site: questions, answers, comments on both, votes, tags and users. For each relationship
   state its cardinality ("few", "many", "squillions") and your design. Create the indexes for: a
   question page, a user's answer history, and the newest questions with a tag.
2. Implement students/courses with an `enrolments` link collection. Write the queries for "courses of
   student X with progress" and "students of course Y, newest first" and create one index for each.
3. Build a folder tree with the ancestors-array shape, 4 levels deep. Write `moveFolder(id, newParentId)`
   and `deleteFolder(id)` (deleting the subtree). Run both inside a transaction once you've reached Level 3.
4. Fix the author-order problem: write a function that returns a book's authors in `authorIds` order using
   `$in` plus client-side reordering, and an aggregation version using `$lookup` followed by sorting on
   `$indexOfArray`.
