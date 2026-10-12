---
description: "Reshape documents in pipelines: $set/$project, $map, $filter, $reduce, $cond, $switch, $ifNull, date and string operators, and safe $convert."
---

# 02 · Reshaping Documents with Expressions

Lesson 01 used stages. This lesson is about **expressions** — the small functional language you write
*inside* stages to compute values: totals from arrays, labels from numbers, local dates from UTC, clean
decimals from messy strings. The same expressions work in `$project`, `$set`, `$group` accumulators,
`$match` with `$expr`, pipeline updates (Level 1 · 06) and validators (Level 2 · 04), so learning them once
pays off everywhere.

Examples use the `shop` database from lesson 01.

## Syntax in one paragraph

An expression is either a **field path** (`"$city"`, `"$items.qty"`), a **variable** (`"$$i"`, `"$$NOW"`,
`"$$ROOT"`), a **literal**, or an **operator document** `{ $operator: [args] }`. Operators nest freely:
`{ $multiply: ["$items.qty", "$items.price"] }`. Inside array operators you name a variable with `as` and
refer to it with two dollar signs. To use a literal string that starts with `$`, wrap it:
`{ $literal: "$5 off" }`.

## $set, $unset and $project

- **`$set`** (alias `$addFields`) adds or replaces fields and keeps everything else.
- **`$unset`** removes fields: `{ $unset: ["items", "city"] }`.
- **`$project`** builds the output explicitly: include (`field: 1`), exclude (`field: 0`), or compute
  (`field: <expression>`).

```javascript
db.orders.aggregate([{ $match: { _id: 1 } }, { $unset: ["items", "city"] }])
```

```text
{"_id":1,"customerId":"C0110","status":"shipped","placedAt":"2026-02-21T11:48:00.000Z"}
```

Use `$set` when you're adding to documents, `$project` when you're defining a final output shape.

## Working on arrays without $unwind

Lesson 01 unwound items to sum them. Often you don't need to: array expressions compute **per document**,
which keeps the number of documents (and the meaning of every count) unchanged.

**`$map`** transforms every element:

```javascript
db.orders.aggregate([
  { $match: { _id: { $in: [1, 2] } } },
  { $set: { lineTotals: { $map: { input: "$items", as: "i", in: { $multiply: ["$$i.qty", "$$i.price"] } } } } },
  { $set: { total: { $sum: "$lineTotals" },
            itemCount: { $reduce: { input: "$items", initialValue: 0, in: { $add: ["$$value", "$$this.qty"] } } } } },
  { $project: { _id: 1, lineTotals: 1, total: 1, itemCount: 1 } }
])
```

```text
{"_id":1,"lineTotals":[{"$numberDecimal":"254"},{"$numberDecimal":"1183"}],"total":{"$numberDecimal":"1437"},"itemCount":2}
{"_id":2,"lineTotals":[{"$numberDecimal":"2549"}],"total":{"$numberDecimal":"2549"},"itemCount":1}
```

`$sum` applied to an array (outside `$group`) adds its elements. **`$reduce`** folds an array into one value,
with `$$value` as the accumulator and `$$this` as the current element — here the total quantity.

**`$filter`** keeps matching elements. Orders (among the first eight) containing electronics, with just those
SKUs:

```javascript
db.orders.aggregate([
  { $match: { _id: { $in: [1, 2, 3, 4, 5, 6, 7, 8] } } },
  { $project: { elec: { $filter: { input: "$items", as: "i", cond: { $eq: ["$$i.category", "electronics"] } } } } },
  { $match: { "elec.0": { $exists: true } } },
  { $project: { skus: "$elec.sku" } }
])
```

```text
{"_id":6,"skus":["P080"]}
{"_id":8,"skus":["P123"]}
```

This is how you get *all* matching elements of an array — Level 1 · 05's positional projection only returned
the first. Note the path `"$elec.sku"` on an array of documents yields an array of the `sku` values.

Other array operators you'll use: `$size`, `$arrayElemAt` (negative indexes count from the end), `$first`
and `$last` (as expressions), `$slice`, `$concatArrays`, `$in`, `$indexOfArray`, `$sortArray`, `$zip`, and
the set operators `$setUnion`, `$setIntersection`, `$setDifference`.

```javascript
db.orders.aggregate([{ $match: { _id: 1 } }, { $project: {
  first: { $arrayElemAt: ["$items.sku", 0] }, last: { $last: "$items.sku" }, n: { $size: "$items" },
  label: { $let: { vars: { n: { $size: "$items" } },
                   in: { $concat: [{ $toString: "$$n" }, " item(s) for ", "$city"] } } } } }])
```

```text
{"_id":1,"first":"P169","last":"P055","n":2,"label":"2 item(s) for Hyderabad"}
```

`$let` names an intermediate value so you don't repeat a sub-expression.

## Conditions

`$cond` is if-then-else; `$switch` is a chain of cases; `$ifNull` supplies a default for null or missing.
Banding orders by total:

```javascript
db.orders.aggregate([
  { $set: { total: { $sum: { $map: { input: "$items", as: "i", in: { $multiply: ["$$i.qty", "$$i.price"] } } } } } },
  { $set: { band: { $switch: {
      branches: [ { case: { $lt: ["$total", 1000] }, then: "small" },
                  { case: { $lt: ["$total", 10000] }, then: "medium" } ],
      default: "large" } } } },
  { $sortByCount: "$band" }
])
```

```text
{"_id":"large","count":13737}
{"_id":"medium","count":13664}
{"_id":"small","count":2599}
```

Branches are evaluated in order and the first true one wins, so ranges can be written as simple upper bounds.
Always give `$switch` a `default`; without one, a document matching no case is an error.

## Dates and time zones

BSON dates are UTC. Convert at the edges — for display and for grouping by *local* day:

```javascript
db.orders.aggregate([{ $match: { _id: { $in: [1, 3] } } }, { $project: {
  cancelled: { $cond: [{ $eq: ["$status", "cancelled"] }, true, false] },
  coupon: { $ifNull: ["$coupon", "none"] },
  dow: { $dayOfWeek: "$placedAt" },
  local: { $dateToString: { date: "$placedAt", format: "%Y-%m-%d %H:%M", timezone: "Asia/Kolkata" } },
  utc: { $dateToString: { date: "$placedAt", format: "%Y-%m-%d %H:%M" } } } }])
```

```text
{"_id":1,"cancelled":false,"coupon":"none","dow":7,"local":"2026-02-21 17:18","utc":"2026-02-21 11:48"}
{"_id":3,"cancelled":false,"coupon":"none","dow":5,"local":"2026-01-15 05:56","utc":"2026-01-15 00:26"}
```

`$dayOfWeek` returns 1 for Sunday through 7 for Saturday (21 February 2026 is a Saturday). Order 3 was placed
at 00:26 UTC — already 05:56 in India. A "daily sales" report grouped by UTC day would put orders placed
between midnight and 05:30 IST on the previous day. Pass `timezone` to `$dateTrunc`, `$dateToString`,
`$dayOfWeek` and friends whenever the business day isn't UTC. `$dateAdd`, `$dateDiff` and `$dateSubtract`
handle calendar arithmetic (month lengths, leap years) correctly; adding milliseconds by hand does not.

## Reshaping whole documents

`$replaceWith` (alias of `$replaceRoot: {newRoot: ...}`) makes an expression the whole document. Combined with
`$mergeObjects`, it's the cleanest way to flatten a line item with fields from its order:

```javascript
db.orders.aggregate([{ $match: { _id: 1 } }, { $unwind: "$items" },
  { $replaceWith: { $mergeObjects: [{ orderId: "$_id", city: "$city" }, "$items"] } }])
```

```text
{"orderId":1,"city":"Hyderabad","sku":"P169","category":"toys","qty":1,"price":{"$numberDecimal":"254"}}
{"orderId":1,"city":"Hyderabad","sku":"P055","category":"kitchen","qty":1,"price":{"$numberDecimal":"1183"}}
```

That flat shape is what you'd export to CSV (Level 1 · 09) or load into a BI tool.

## Converting types safely

Imported data is often dirty. `$convert` converts with fallbacks; the `$toX` shortcuts don't have them:

```javascript
db.raw.insertMany([{ v: "12.50" }, { v: "abc" }, { v: null }, { v: 7 }]);
db.raw.aggregate([{ $project: { _id: 0, v: 1,
  asDec: { $convert: { input: "$v", to: "decimal", onError: "BAD", onNull: "MISSING" } } } }])
```

```text
{"v":"12.50","asDec":{"$numberDecimal":"12.50"}}
{"v":"abc","asDec":"BAD"}
{"v":null,"asDec":"MISSING"}
{"v":7,"asDec":{"$numberDecimal":"7"}}
```

```javascript
db.raw.aggregate([{ $project: { x: { $toDecimal: "$v" } } }])
```

```text
241 Executor error during aggregate command on namespace: shop.raw :: caused by :: Failed to parse number 'abc' in $convert ...
```

One bad value fails the whole pipeline. In cleanup pipelines, use `$convert` with `onError`, then `$match` the
failures into a separate report.

## How It Actually Works

Expressions are parsed once, when the pipeline is built, into a tree of expression objects; for each document,
the stage evaluates the tree against that document's fields and the current variable scope. `$$ROOT` is the
whole input document, `$$CURRENT` the document being processed (usually the same), and `$$NOW`/`$$CLUSTER_TIME`
are fixed for the whole operation, so every document sees the same "now". Array operators like `$map` and
`$filter` create a new scope per element with the variable you named, which is why `$$i` is visible only inside
the `in`/`cond` expression.

Expressions run inside the server on BSON values, so they preserve types: `$multiply` of an int and a Decimal128
yields Decimal128 (as in the line totals above), integer overflow promotes to long or double, and comparisons
follow the BSON type order from Level 1 · 03. The optimiser also performs constant folding — an expression with
no field references is computed once — and can push `$set`/`$project` work past `$match` stages when that's safe.

Per-document expressions are cheap compared to moving documents between stages. Computing an order total with
`$map`/`$sum` keeps one document per order; `$unwind` + `$group` creates one document per line item and then
rebuilds groups in a hash table. Reach for `$unwind` when you need to group **by** something inside the array
(revenue per SKU), not when you only need to aggregate **within** each document.

## Common mistakes

- **`$unwind` + `$group` to compute per-document totals** that `$map`/`$sum` would compute in place.
- **One `$` instead of two** inside `$map`/`$filter` (`"$i.qty"` is a field path, `"$$i.qty"` the variable).
- **Grouping by UTC day** when the business operates in another time zone.
- **`$switch` without a default.**
- **`$toDecimal`/`$toInt` on dirty data** without `$convert`'s `onError`.
- **Expecting `$project` to keep fields you didn't list** — that's `$set`'s job.

## Exercise

Using the `shop` database:

1. Add `total` and `itemCount` to every order with one `$set` using `$map`, `$sum` and `$reduce`, then find the
   10 orders with the highest `itemCount`.
2. For each order, compute `electronicsShare`: the fraction of the order total spent on electronics, rounded to
   two decimals (`$round`). Return orders where it is between 0.4 and 0.6.
3. Produce "orders per local (IST) hour of day" for delivered orders, and compare the shape with the same report
   in UTC.
4. Flatten orders to one row per line item with `orderId`, `customerId`, local date string, `sku`, `qty` and
   `lineTotal`, and export the first 1,000 rows to CSV with `$out` + `mongoexport` (lesson 05 covers `$out`).
5. Write a pipeline that classifies each customer as `new` (first order in the last 90 days of the data), `active`
   or `lapsed` (no order in the last 120 days of the data), using `$switch` and `$dateDiff`.
