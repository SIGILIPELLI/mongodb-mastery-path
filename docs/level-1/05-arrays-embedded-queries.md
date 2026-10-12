---
description: "How MongoDB matches arrays and embedded documents: contains vs exact match, $all, $size, why $elemMatch matters, dot paths and $slice projection."
---

# 05 · Querying Arrays & Embedded Documents

Arrays and embedded documents are why you chose a document database: a student's test results live
inside the student, an order's line items inside the order. But array queries follow rules that
surprise nearly everyone the first time — a filter that "obviously" means one thing matches documents
you didn't expect. This lesson reproduces each surprise so you recognise it in your own data.

## Sample data

```javascript
const d = db.getSiblingDB("arrays");
d.students.insertMany([
  { _id: "ana", tags: ["math", "chess"], scores: [72, 95],
    tests: [{ subject: "math", score: 95, year: 2025 }, { subject: "art", score: 60, year: 2026 }],
    address: { city: "Pune", zip: "411001" } },
  { _id: "ben", tags: ["chess"], scores: [88, 81],
    tests: [{ subject: "math", score: 58, year: 2026 }, { subject: "art", score: 91, year: 2025 }],
    address: { city: "Hyderabad", zip: "500001" } },
  { _id: "cy", tags: ["math", "chess", "go"], scores: [55, 62, 99],
    tests: [{ subject: "math", score: 91, year: 2026 }], address: { city: "Pune" } },
  { _id: "dee", tags: [], scores: [], tests: [] },
  { _id: "eve", tags: ["go"], scores: [50, 95], tests: [], address: { city: "Chennai" } }
]);
const ids = q => d.students.find(q, { _id: 1 }).toArray().map(x => x._id).join(",");
```

The helper `ids()` prints just the matching `_id`s so results fit on one line.

## A scalar condition on an array means "contains"

```javascript
ids({ tags: "chess" })
```

```text
ana,ben,cy
```

When a field holds an array, `{field: value}` matches if **any element** equals the value. You never
write a loop; the server does it.

## An array condition means "exactly this array"

```javascript
ids({ tags: ["math", "chess"] })   // → ana
ids({ tags: ["chess", "math"] })   // → (nothing)
```

Giving an array as the value asks for an exact match: same elements, same order, same length. Cy has
math and chess too, but also go, so it doesn't match. To ask "contains all of these, in any order",
use `$all`:

```javascript
ids({ tags: { $all: ["chess", "math"] } })
```

```text
ana,cy
```

`$size` matches an exact length — useful for finding empty arrays:

```javascript
ids({ tags: { $size: 0 } })
```

```text
dee
```

`$size` only takes an exact number. For "at least 3 tags" use `{"tags.2": {$exists: true}}` (an
element at index 2 exists) or `$expr` with `$size`.

## The surprise: several conditions on one array

Which students have a score greater than 90 *and* less than 60?

```javascript
ids({ scores: { $gt: 90, $lt: 60 } })
```

```text
cy
```

No number is both above 90 and below 60, yet Cy matched. Each condition is checked against the array
**independently**: some element is above 90 (99), and some element — possibly a different one — is
below 60 (55). That is sometimes what you want. When you need a **single element** to satisfy all the
conditions, use `$elemMatch`:

```javascript
ids({ scores: { $gt: 60, $lt: 90 } })                  // ana,ben,cy,eve
ids({ scores: { $elemMatch: { $gt: 60, $lt: 90 } } })  // ana,ben,cy
```

Eve's scores are 50 and 95. Neither is between 60 and 90, but 95 is above 60 and 50 is below 90, so
the first query matches her and the second does not.

## The same surprise with arrays of documents

This is where it causes real bugs. "Students who scored at least 90 in math":

```javascript
ids({ "tests.subject": "math", "tests.score": { $gte: 90 } })
```

```text
ana,ben,cy
```

Ben scored **58** in math. He matched because he has *a* test whose subject is math (the 58) and
*a* test with score ≥ 90 (his 91 in art). Adding a third condition doesn't help:

```javascript
ids({ "tests.subject": "math", "tests.score": { $gte: 90 }, "tests.year": 2026 })
```

```text
ana,ben,cy
```

Each dotted condition can be satisfied by a different element. `$elemMatch` ties them to one element:

```javascript
ids({ tests: { $elemMatch: { subject: "math", score: { $gte: 90 }, year: 2026 } } })
```

```text
cy
```

Only Cy has one test that is math, ≥ 90 **and** in 2026. Rule of thumb: **if you put two or more
conditions on fields of the same array of documents, you almost certainly want `$elemMatch`**.

## Dot notation for embedded documents and array positions

```javascript
ids({ "address.city": "Pune" })     // ana,cy — field inside an embedded document
ids({ "scores.0": { $gt: 70 } })    // ana,ben — the first element of the array
ids({ "tests.0": { $exists: false } })  // dee,eve — empty (or missing) tests array
```

A numeric path component means "the element at this index". Use it sparingly: position-based logic
breaks as soon as someone inserts at the front of the array.

Remember from lesson 03 that `{address: {city: "Pune"}}` (no dot) would only match documents whose
*entire* address is exactly `{city: "Pune"}` — that matches Cy, but not Ana, whose address also has a
zip.

## Returning only part of an array

By default you get the whole array back. Three projections trim it.

**Positional `$`** returns the first element that matched the query:

```javascript
d.students.find({ "tests.subject": "art" }, { _id: 1, "tests.$": 1 })
```

```text
[
  { _id: 'ana', tests: [ { subject: 'art', score: 60, year: 2026 } ] },
  { _id: 'ben', tests: [ { subject: 'art', score: 91, year: 2025 } ] }
]
```

**`$elemMatch` in a projection** returns the first element matching its own condition, independent of
the query:

```javascript
d.students.find({ _id: "ben" }, { _id: 0, tests: { $elemMatch: { score: { $gte: 90 } } } })
```

```text
[ { tests: [ { subject: 'art', score: 91, year: 2025 } ] } ]
```

**`$slice`** returns a range — here the last two scores:

```javascript
d.students.find({ _id: "cy" }, { scores: { $slice: -2 }, _id: 0, tags: 0, tests: 0, address: 0 })
```

```text
[ { scores: [ 62, 99 ] } ]
```

And a dotted inclusion keeps the array but only that field of each element:

```javascript
d.students.find({}, { _id: 1, "tests.score": 1 }).limit(2)
```

```text
[
  { _id: 'ana', tests: [ { score: 95 }, { score: 60 } ] },
  { _id: 'ben', tests: [ { score: 58 }, { score: 91 } ] }
]
```

Both `$` and projection-`$elemMatch` return **only the first** matching element. If you need every
matching element (all of a student's tests above 90), use the aggregation `$filter` expression —
Level 3 · 02.

## How It Actually Works

The matcher evaluates each leaf of the filter against a **path**. When it walks a path like
`tests.score` and meets an array along the way, it doesn't stop — it fans out and collects every value
reachable through that path: for Ben, `tests.score` yields `{58, 91}`. A leaf such as
`{$gte: 90}` is true if **any** collected value satisfies it. The leaves of an implicit AND are each
evaluated separately, with their own fan-out, which is exactly why conditions "jump" between
elements.

`$elemMatch` changes the unit of evaluation: it iterates the array's elements and applies its whole
sub-filter to one element at a time, returning true as soon as one element satisfies everything. That
is also why it is the only way to express "same element" constraints in an index-friendly way: a
multikey index (Level 2 · 06) stores one entry per array element, and the planner can only combine
bounds on `tests.subject` and `tests.score` tightly when it knows they must come from the same element
— which `$elemMatch` tells it.

When the value in an equality condition is itself an array (`{tags: ["math","chess"]}`), the matcher
first tries to match the whole field as one value (exact array equality) and then tries each element
(so a nested array element `[["math","chess"]]` would also match). That two-step rule is why the
array-literal form means "exactly equal" rather than "contains all".

## Common mistakes

- **Multiple dotted conditions on one array of documents without `$elemMatch`** — the Ben bug. It
  passes tests with simple data and fails with real data.
- **Using an array literal when you mean `$all`.** Order and length must match exactly.
- **Expecting `$size` to accept ranges.** It doesn't; use an index-position `$exists` or `$expr`.
- **Expecting positional `$` to return all matches.** It returns the first only.
- **Index-based paths (`scores.0`) for meaning** like "latest score". Store "latest" explicitly if you
  query it.

## Exercise

Using the `arrays.students` collection:

1. Find students who have *both* the `math` and `go` tags, in any order.
2. Find students with at least one score from 80 to 89 inclusive. Then write the version *without*
   `$elemMatch` and find a document (insert one if needed) that the two queries disagree on.
3. Find students who took an art test in 2025 with a score above 90. Prove that your query is correct
   by inserting a student who would wrongly match a version without `$elemMatch`.
4. Return each student's `_id` and only their highest-indexed (last) test using `$slice`.
5. Find students who have an `address` but no `zip` inside it.
