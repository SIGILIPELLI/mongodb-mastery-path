---
description: "What a MongoDB document really is: BSON types, ObjectId anatomy, int vs double vs Decimal128, dates, null vs missing, mixed sorting, the 16 MB limit."
---

# 03 · Documents, BSON & Data Types

A MongoDB document looks like JSON, and most of the time you can think of it that way. But it is
stored and sent as **BSON** — a binary format with more types than JSON, a defined comparison order
across those types, and a hard size limit. The differences are small and the bugs they cause are
large: totals that are off by a fraction of a cent, dates that sort wrong, queries that silently
match nothing.

Everything below was run in `mongosh` against the lab server, in a database called `bson`.

## The types you will actually use

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

There are others (`timestamp`, `regex`, `minKey`, `maxKey`, `javascript`) that you will rarely store
yourself. The `timestamp` type in particular is internal — it is what the oplog uses — so store
points in time as `date`.

## Size on disk, byte by byte

`bsonsize()` in the shell shows how many bytes a document takes:

```javascript
print(bsonsize({}), bsonsize({a:1}), bsonsize({a:1.5}), bsonsize({a:"hi"}),
      bsonsize({a:NumberLong("1")}), bsonsize({a:NumberDecimal("1.5")}), bsonsize({_id:new ObjectId()}))
```

```text
5 12 16 15 16 24 22
```

An empty document is 5 bytes: a 4-byte length and a terminating zero. `{a:1}` adds a type byte, the
field name `a` plus its null terminator, and a 4-byte int: 12. The same value as a double needs 8
bytes, so 16. Decimal128 needs 16 bytes for the value. Note that **the field name is stored in every
document** — a field called `customer_shipping_address_line_1` costs 33 bytes in every document that
has it, before any value. On a billion documents, naming matters. (Compression on disk reduces this,
but the cache and the wire still carry the full names.)

## ObjectId: a timestamp in disguise

When you insert a document without `_id`, the driver generates an ObjectId. It is 12 bytes: a 4-byte
timestamp (seconds since the epoch), 5 bytes of random value fixed per process, and a 3-byte
counter.

```javascript
const id = new ObjectId("6703a1f2c9e77a1b2c3d4e5f");
id.getTimestamp().toISOString()
```

```text
2024-10-07T08:55:14.000Z
```

Two ObjectIds generated back to back in one process:

```text
6acbc68a750685768b1b44d3
6acbc68a750685768b1b44d4
```

Same first 8 hex digits (the second they were made in), same random middle, counter incremented by
one. Consequences:

- ObjectIds are roughly increasing over time, so sorting by `_id` approximates insertion order across
  processes (only to the second; within a second, across machines, it is not exact).
- You can recover a document's creation time from its `_id` without a separate field.
- They are **not secret**: anyone holding one knows when it was created and can guess neighbours.
  Do not use them as unguessable tokens in URLs for password resets or private links.

## Numbers: why money needs Decimal128

```javascript
db.money.insertMany([
  { k: "double", v: 0.1 }, { k: "double", v: 0.2 },
  { k: "dec", v: NumberDecimal("0.1") }, { k: "dec", v: NumberDecimal("0.2") }
]);
db.money.aggregate([{ $group: { _id: "$k", total: { $sum: "$v" } } }, { $sort: { _id: 1 } }])
```

```text
[
  { _id: 'dec', total: Decimal128('0.3') },
  { _id: 'double', total: 0.30000000000000004 }
]
```

The double total is the classic binary floating-point result: 0.1 has no exact representation in
base 2. For currency either store **Decimal128** or store integer minor units (cents as `long`).
Whichever you choose, choose once — a collection where some prices are doubles and some are decimals
sorts and compares correctly (all numeric types compare by value) but adds up inconsistently.

## Dates belong in Date fields

```javascript
db.dates.insertMany([
  { s: "2026-9-5",   t: new Date("2026-09-05") },
  { s: "2026-10-01", t: new Date("2026-10-01") }
]);
db.dates.find({}, { _id: 0, s: 1 }).sort({ s: -1 })   // "newest first" by the string
db.dates.find({}, { _id: 0, s: 1 }).sort({ t: -1 })   // newest first by the date
```

```text
[ { s: '2026-9-5' }, { s: '2026-10-01' } ]
[ { s: '2026-10-01' }, { s: '2026-9-5' } ]
```

The string sort is wrong because strings compare character by character and `"9"` is greater than
`"1"`. A real `Date` also gives you date operators in aggregation (`$dateTrunc`, `$dateDiff`,
time zones), TTL indexes (Level 2 · 06) and 8 bytes instead of 10–30. BSON dates are always UTC;
convert to local time when displaying, not when storing.

## Null versus missing

```javascript
db.mixed.insertMany([{_id:1,v:"10"},{_id:2,v:9},{_id:3,v:null},{_id:4},{_id:5,v:true},
  {_id:6,v:new Date(0)},{_id:7,v:[3]},{_id:8,v:{x:1}},{_id:9,v:ObjectId("000000000000000000000000")}]);
print(db.mixed.countDocuments({v:null}),
      db.mixed.countDocuments({v:{$exists:false}}),
      db.mixed.countDocuments({v:{$type:"null"}}));
```

```text
2 1 1
```

`{v: null}` matches **both** the document where `v` is null and the one where `v` does not exist.
When you need to tell them apart, use `$exists` or `$type`.

## Comparing values of different types

A field is allowed to hold different types in different documents. Sorting such a field follows a
fixed order of type classes: null and missing, then numbers, then strings, then objects, arrays,
binary data, ObjectId, booleans, dates, and so on:

```javascript
db.mixed.find({}, { _id: 1 }).sort({ v: 1 }).toArray().map(x => x._id)
```

```text
[ 3, 4, 7, 2, 1, 8, 9, 5, 6 ]
```

Null (3) and missing (4) first; then the numbers — `[3]` (7) sorts by its smallest element when
ascending, so it lands among the numbers before 9 (2); then the string `"10"` (1); the object (8);
the ObjectId (9); `true` (5); and the date (6).

**Queries**, on the other hand, only compare within the same type class ("type bracketing"):

```javascript
print(db.mixed.countDocuments({v:{$gt:5}}), db.mixed.countDocuments({v:{$gt:"1"}}));
```

```text
1 1
```

`$gt: 5` matched only the number 9 — not the string `"10"`, not the date. `$gt: "1"` matched only
the string `"10"`. This is why a field that is a number in some documents and a numeric *string* in
others makes queries look broken: half the data is invisible to any one query.

## Embedded documents: field order matters for whole-document equality

```javascript
db.emb.insertOne({ p: { a: 1, b: 2 } });
print(db.emb.countDocuments({ p: { a: 1, b: 2 } }),
      db.emb.countDocuments({ p: { b: 2, a: 1 } }),
      db.emb.countDocuments({ "p.a": 1, "p.b": 2 }));
```

```text
1 0 1
```

Matching an entire embedded document compares it byte for byte, including field order — so
`{b:2, a:1}` does not match. Dot notation (`"p.a"`) matches individual fields and does not care about
order. Prefer dot notation unless you really mean "exactly this sub-document".

## Field names with dots and dollars

Modern servers accept them, but they are awkward:

```javascript
db.odd.insertOne({ "a.b": 1, "$x": 2 });
db.odd.countDocuments({ "a.b": 1 })                                    // 0 — dot means "path"
db.odd.countDocuments({ $expr: { $eq: [{ $getField: "a.b" }, 1] } })   // 1
```

```text
0
1
```

The ordinary query treats `"a.b"` as the path `a` → `b`, which doesn't exist. Avoid dots and leading
dollars in field names; they are the source of endless escaping problems. This matters most when you
use user input (email addresses, URLs) as *keys* — store them as values instead.

## The 16 MB limit

```javascript
for (const mb of [15.9, 16.0]) {
  try { db.big.insertOne({ blob: "x".repeat(Math.floor(mb * 1024 * 1024)) }); print(mb, "ok") }
  catch (e) { print(mb, e.name, e.code, e.message) }
}
```

```text
15.9 ok
16 MongoServerError 10334 object to insert too large. size in bytes: 16777249, max size: 16777216
```

A single document can be at most 16 MiB (`db.hello().maxBsonObjectSize` returns `16777216`). The
limit is deliberate: a document is read and written as a unit, so a huge one makes every operation on
it expensive. If you approach it you almost always have an *unbounded array* — comments, events,
log lines appended forever — and the fix is a modeling change (Level 2 · 03), not a bigger limit.
For genuinely large binary files there is GridFS, which splits a file across many documents.

## How It Actually Works

A BSON document is a length-prefixed sequence of elements. Each element is: one type byte, the field
name as a null-terminated C string, then the value encoded according to its type (4 bytes for an int,
8 for a double or date, 12 for an ObjectId, a length plus bytes for strings and binary, and a nested
BSON document for objects and arrays). An array is literally an embedded document whose keys are the
strings `"0"`, `"1"`, `"2"`…

The length prefixes make BSON fast to *skip*: to read field 30 the server can hop over fields 1–29
using their lengths without parsing their contents. The cost is that values are stored in
document order, so finding a field is a linear scan of the document — another reason to keep
documents reasonably sized.

Comparison is defined in two layers. First the **type class** order (MinKey, null, numbers, symbols
and strings, objects, arrays, binary, ObjectId, booleans, dates, timestamps, regular expressions,
MaxKey). Then, within a class, values compare naturally: numbers by value regardless of int, long,
double or decimal; strings by simple binary comparison unless a **collation** says otherwise (Level 3
· 09). Sort uses the full order so every document has a place; query operators like `$gt` stay inside
the type class of the value you gave them, which is why indexes can answer range queries with one
contiguous scan.

## Common mistakes

- **Storing dates as strings.** They sort wrong, can't use date operators or TTL, and take more space.
- **Storing money as doubles.** Use Decimal128 or integer minor units, consistently.
- **Mixing types in one field** (`"42"` here, `42` there). Queries see only one type at a time.
- **Using ObjectIds as secrets.** They are timestamped and sequential.
- **Matching embedded documents whole** when you meant to match some of their fields — use dot notation.
- **Long field names on high-volume collections.** They're stored in every document.
- **Letting arrays grow forever** until a write hits the 16 MB limit in production.

## Exercise

1. Generate three ObjectIds a second apart and decode their timestamps with `getTimestamp()`.
   Which bytes changed between them?
2. Store ten prices as doubles and the same ten as Decimal128, then sum both with `$group`. Find a set
   of prices where the double total is visibly wrong.
3. In one collection, store a `qty` field as `5`, `"5"`, `5.0`, `NumberLong("5")` and
   `NumberDecimal("5")`. Which of them does `{qty: 5}` match? Which does `{qty: "5"}` match? Explain.
4. Use `bsonsize()` to compare a document with the field name `customerShippingAddressLine1` against
   one with `addr1`. Multiply the difference by 50 million documents.
5. Write a query that finds documents where a field is missing but **not** where it is null.
