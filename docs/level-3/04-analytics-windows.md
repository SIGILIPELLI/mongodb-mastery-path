---
description: "MongoDB analytics: multi-facet results with $facet, histograms with $bucket and $bucketAuto, window functions with $setWindowFields, gap filling."
---

# 04 · Analytics: $facet, $bucket & Window Functions

Lessons 01–03 produced one result per pipeline: a list of groups, a joined document. Real dashboards and search
pages need more: a result list *and* the category counts down the side *and* the price histogram, from one
request; a running total; a moving average; a rank within each city; a chart with no missing days. This lesson
covers the stages built for that kind of analytics.

Examples use the `shop` database. Several pipelines start by computing an order total as a double, so the numbers
are easy to round and print:

```javascript
const tot = { $set: { total: { $toDouble: { $sum: { $map: { input: "$items", as: "i",
                in: { $multiply: ["$$i.qty", "$$i.price"] } } } } } } };
```

(For money you'd keep Decimal128; converting to double here is purely for readable charts.)

## $facet: several pipelines over the same documents

A search results page for "delivered orders in Pune" with a category breakdown, price bands, the newest
results and a total count:

```javascript
db.orders.aggregate([
  { $match: { city: "Pune", status: "delivered" } }, tot,
  { $facet: {
      byCategory: [{ $unwind: "$items" }, { $sortByCount: "$items.category" }],
      priceBands: [{ $bucket: { groupBy: "$total", boundaries: [0, 1000, 5000, 20000, 50000],
                                default: "50000+", output: { n: { $sum: 1 } } } }],
      newest: [{ $sort: { placedAt: -1 } }, { $limit: 2 }, { $project: { _id: 1, total: 1 } }],
      count: [{ $count: "n" }] } }
])
```

```text
{"byCategory":[{"_id":"electronics","count":2181},{"_id":"books","count":2167},{"_id":"kitchen","count":1975},{"_id":"toys","count":1875}],
 "priceBands":[{"_id":0,"n":374},{"_id":1000,"n":1309},{"_id":5000,"n":988},{"_id":20000,"n":547},{"_id":"50000+","n":988}],
 "newest":[{"_id":664,"total":10650},{"_id":20701,"total":2933}],
 "count":[{"n":4206}]}
```

One round trip, one document, four answers. The documents entering `$facet` are read once and fed to each
sub-pipeline. Two limits to respect:

- The **output is one document**, so all facets together must stay under 16 MB. Always `$limit` list facets.
- Sub-pipelines **can't use indexes** — they operate on the documents `$facet` received. Put the selective
  `$match` (and any `$sort` you want indexed) *before* `$facet`.

`byCategory` counts **line items** per category (it unwinds); if the side panel should show "orders containing
a book", group on distinct order ids or use `$addToSet` of categories per order first.

## Histograms: $bucket and $bucketAuto

`$bucket` (used above) takes explicit boundaries: each document goes in the bucket whose lower bound is the
largest boundary ≤ its value; values outside the boundaries go to `default` (or cause an error if there's no
default). It's for ranges that mean something to the business.

`$bucketAuto` picks boundaries itself to spread documents evenly across N buckets:

```javascript
db.orders.aggregate([{ $match: { status: "delivered" } }, tot, { $bucketAuto: { groupBy: "$total", buckets: 4 } }])
```

```text
{"_id":{"min":228,"max":2686},"count":6021}
{"_id":{"min":2686,"max":8231},"count":6020}
{"_id":{"min":8231,"max":46504},"count":6020}
{"_id":{"min":46504,"max":402892},"count":6019}
```

Those are quartiles of order value: a quarter of delivered orders are under ₹2,686, and the top quarter spans
₹46,504 to ₹402,892 — a long tail typical of baskets containing electronics.

## Window functions: $setWindowFields

`$group` collapses documents into groups. Window functions compute values *across* neighbouring documents while
keeping every document — the running total, the moving average, the rank. Daily revenue in Pune for the first nine
days of 2026, with a running total and a 3-day moving average:

```javascript
db.orders.aggregate([
  { $match: { city: "Pune", status: "delivered",
              placedAt: { $gte: new Date("2026-01-01"), $lt: new Date("2026-01-10") } } }, tot,
  { $group: { _id: { $dateTrunc: { date: "$placedAt", unit: "day" } }, revenue: { $sum: "$total" } } },
  { $setWindowFields: { sortBy: { _id: 1 }, output: {
      running: { $sum: "$revenue", window: { documents: ["unbounded", "current"] } },
      ma3: { $avg: "$revenue", window: { documents: [-2, 0] } } } } },
  { $project: { _id: 0, day: { $dateToString: { date: "$_id", format: "%m-%d" } },
                revenue: { $round: ["$revenue", 0] }, running: { $round: ["$running", 0] }, ma3: { $round: ["$ma3", 0] } } }
])
```

```text
{"day":"01-01","revenue":244592,"running":244592,"ma3":244592}
{"day":"01-02","revenue":614289,"running":858881,"ma3":429440}
{"day":"01-03","revenue":119788,"running":978669,"ma3":326223}
{"day":"01-04","revenue":619420,"running":1598089,"ma3":451166}
{"day":"01-05","revenue":102073,"running":1700162,"ma3":280427}
{"day":"01-06","revenue":339622,"running":2039784,"ma3":353705}
{"day":"01-07","revenue":441535,"running":2481319,"ma3":294410}
{"day":"01-08","revenue":156187,"running":2637506,"ma3":312448}
{"day":"01-09","revenue":705777,"running":3343283,"ma3":434500}
```

The window is defined relative to the current document after sorting: `["unbounded", "current"]` is everything up
to here (running total); `[-2, 0]` is the two previous documents and this one. Note the first two moving averages
cover fewer than three days — the window is truncated at the start. You can also define windows by value range
(`range: [-7, 0], unit: "day"`), which is the right choice when days might be missing: a document window of 3
"documents" silently spans more than 3 days if a day has no sales.

Window functions run **per partition**. Rank customers by spend within each city and keep the top two:

```javascript
db.orders.aggregate([
  { $match: { status: "delivered" } }, tot,
  { $group: { _id: { city: "$city", c: "$customerId" }, spend: { $sum: "$total" } } },
  { $setWindowFields: { partitionBy: "$_id.city", sortBy: { spend: -1 }, output: { rank: { $rank: {} } } } },
  { $match: { rank: { $lte: 2 } } }, { $sort: { "_id.city": 1, rank: 1 } }, { $limit: 4 },
  { $project: { _id: 0, city: "$_id.city", customer: "$_id.c", spend: { $round: ["$spend", 0] }, rank: 1 } }
])
```

```text
{"rank":1,"city":"Bengaluru","customer":"C0649","spend":1055327}
{"rank":2,"city":"Bengaluru","customer":"C1779","spend":1034841}
{"rank":1,"city":"Chennai","customer":"C0481","spend":1252339}
{"rank":2,"city":"Chennai","customer":"C0810","spend":909825}
```

Other window operators: `$denseRank`, `$documentNumber`, `$shift` (previous/next value — for "change since
yesterday"), `$derivative`, `$integral`, `$expMovingAvg`, `$covariancePop`, plus every `$group` accumulator applied
over a window.

## Filling gaps: $densify and $fill

Charts lie when days with no data are simply absent. Given counts for 1, 2 and 5 March:

```javascript
db.daily.insertMany([{ d: new Date("2026-03-01"), n: 5 }, { d: new Date("2026-03-02"), n: 7 }, { d: new Date("2026-03-05"), n: 4 }]);
db.daily.aggregate([
  { $densify: { field: "d", range: { step: 1, unit: "day", bounds: "full" } } },
  { $fill: { sortBy: { d: 1 }, output: { n: { value: 0 } } } },
  { $project: { _id: 0, d: { $dateToString: { date: "$d", format: "%m-%d" } }, n: 1 } }
])
```

```text
{"n":5,"d":"03-01"}
{"n":7,"d":"03-02"}
{"n":0,"d":"03-03"}
{"n":0,"d":"03-04"}
{"n":4,"d":"03-05"}
```

`$densify` inserts documents for the missing dates; `$fill` sets their values. Zero is right for counts. For
measurements (a temperature you didn't record), interpolation may be better:

```javascript
{ $fill: { sortBy: { d: 1 }, output: { n: { method: "linear" } } } }
```

```text
{"n":5,"d":"03-01"}
{"n":7,"d":"03-02"}
{"n":6,"d":"03-03"}
{"n":5,"d":"03-04"}
{"n":4,"d":"03-05"}
```

`method: "locf"` (last observation carried forward) is the third option. Choose deliberately — filling sales gaps
with interpolated numbers invents sales.

## How It Actually Works

`$facet` buffers nothing per facet beyond what each sub-pipeline's own stages need: as each input document arrives,
it is passed through every sub-pipeline, and at the end each sub-pipeline's results are collected into an array
field of the single output document. That's why the inputs aren't re-read, why sub-pipelines can't use indexes
(they don't read collections — they read the stream), and why the 16 MB output limit applies to all facets
combined.

`$setWindowFields` first sorts by `partitionBy` and `sortBy` (a blocking sort, or an index-provided order if the
preceding stages allow it), then walks each partition maintaining the state of each window as it slides: for
`["unbounded", "current"]` it just keeps adding; for a bounded window it adds the entering document and removes the
leaving one, so a moving average over N documents costs O(1) per document rather than O(N). Window operators whose
state can't be updated incrementally (like `$median` over a sliding window) cost more, and windows that must hold many
documents in memory are subject to the 100 MB per-stage limit, spilling where supported.

`$densify` works on the sorted stream too: it remembers the last value of the field (per partition) and generates
the missing steps between it and the next value, respecting time-zone-aware units for dates.

## Common mistakes

- **Unlimited arrays inside `$facet`**, pushing the single output document over 16 MB.
- **Expecting `$facet` sub-pipelines to use indexes.** Filter before the facet.
- **Document-count windows over data with gaps**, so "last 7 days" silently spans weeks. Use range windows or
  `$densify` first.
- **Ranking with `$rank` when you need unique positions** — ties share a rank; use `$documentNumber`.
- **Interpolating values that should be zero.**
- **Running analytics pipelines against the primary of a busy OLTP system** without thinking about load —
  consider a secondary with an analytics read preference (Level 4 · 02) or a precomputed rollup (lesson 05).

## Exercise

1. Build a product search facet for the `products` collection: matching products (top 10 by price), counts per
   category, and a price histogram with `$bucketAuto` into 5 buckets.
2. For each city, compute monthly delivered revenue and the month-over-month change in percent using `$shift`.
3. Compute, for every delivered order, the customer's cumulative number of orders up to and including it (their
   "order number"), using `$setWindowFields` partitioned by customer. Find how many customers placed a 10th order.
4. Produce daily order counts for Hyderabad for all of February 2026 with no missing days, then a 7-day moving
   average defined by a **range** window. Compare it with a 7-**document** window.
5. Rewrite the facet example so `byCategory` counts orders rather than line items.
