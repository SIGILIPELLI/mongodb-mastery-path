---
description: "Proven MongoDB schema patterns with measurements: computed, subset, extended reference, bucket, attribute, outlier, polymorphic and schema versioning."
---

# 03 · Schema Design Patterns

Experienced MongoDB designers reach for a small set of recurring shapes. They have names — computed,
subset, bucket, attribute and so on — and knowing them speeds up design discussions the way knowing
"observer" or "factory" speeds up code reviews. Each pattern is an answer to a specific pressure:
too many reads, too many documents, too many fields, too much variation. This lesson shows each one,
when it applies, and — where it can be measured — what it buys.

## Computed pattern: do the math on write

**Pressure:** a value is read far more often than the data it's derived from changes.

The recipe box in Level 1 stored `rating: {count, total}` instead of recomputing the average from every
rating on every page view. Same idea for a product's review count, a forum's post count, an account's
balance. Compute it **when the inputs change**, with an atomic `$inc`, and reads become free.

```javascript
db.posts.updateOne({ _id: postId }, { $inc: { commentCount: 1 }, $set: { lastCommentAt: new Date() } })
```

The price: the derived value is a second source of truth. If you also insert the comment document, the
two writes can disagree after a crash in between — either accept a rare off-by-one that a nightly job
corrects, or use a transaction (Level 3 · 06).

## Subset pattern: keep the hot part embedded

**Pressure:** a document is big because of a list, but readers only need the first few items.

A product page shows the 10 most recent reviews; the "all reviews" page is rare. Embed the latest 10
(bounded with `$push` + `$slice`, Level 1 · 06) and keep every review in a `reviews` collection.

```javascript
// on new review: two writes
db.reviews.insertOne(review);
db.products.updateOne({ _id: review.productId }, {
  $push: { recentReviews: { $each: [review], $sort: { at: -1 }, $slice: 10 } },
  $inc: { reviewCount: 1 }
});
```

Lesson 01 measured why: reading a title from a 3.2 KB post with embedded comments cost the same as
reading the whole post. The subset keeps the hot document small and the hot read single-query.

## Extended reference: copy the fields you display

**Pressure:** a frequent read needs a few fields from a related document.

An order displays the customer's name and shipping address. Instead of joining to `customers` on every
order view, copy those fields into the order: `customer: {_id, name, shippingAddress}`. Often the copy is
not just faster but *more correct* — the order should show the address it was shipped to, even if the
customer moves later. Copy fields that rarely change or that should be frozen as history; never copy
things like passwords or balances.

## Bucket pattern: group many small documents into fewer, larger ones

**Pressure:** huge numbers of tiny, similar documents — readings, events, metrics — each paying
per-document and per-index-entry overhead.

An experiment: 10 sensors reporting one temperature per minute for a week, stored two ways.

- `readings`: one document per reading, index on `{sensor: 1, at: 1}`.
- `buckets`: one document per sensor per hour, readings appended to an array, with running aggregates.

The bucket write is an upsert that creates the hour's document on its first reading:

```javascript
const hour = new Date(Math.floor(r.at.getTime() / 3600000) * 3600000);
db.buckets.updateOne(
  { sensor: r.sensor, hour },
  { $push: { m: { at: r.at, temp: r.temp } },
    $inc: { count: 1, sum: r.temp }, $min: { min: r.temp }, $max: { max: r.temp } },
  { upsert: true });
// with: db.buckets.createIndex({ sensor: 1, hour: 1 }, { unique: true })
```

The resulting collections:

```text
readings: docs=100800 dataMB=6.3 storageMB=2.3 indexMB=3.79
buckets: docs=1680 dataMB=3.6 storageMB=1.3 indexMB=0.13
```

The same data in 60× fewer documents, with a little over half the data size and **29× less index**. Indexes must
fit in RAM to be fast, so that last number is the one that matters at scale.

Then the query "hourly average temperature for sensor s3 on 3 October":

```text
24 24 22.450 22.450 flat ms 14 bucket ms 3
flat docsExamined 1440 bucket docsExamined 24
```

Both designs give the same 24 hourly averages (22.450 for the sixth hour in both), but the flat design
read 1,440 documents and grouped them, while the bucket design read 24 documents and divided two numbers
already stored in each. The buckets' pre-computed `count`, `sum`, `min` and `max` are the computed
pattern applied inside the bucket pattern.

The costs: writes are updates to a growing document rather than plain inserts; queries for an individual
reading must reach into the array; and you must choose the bucket size up front (an hour of minute data =
60 elements, a sensible size; a year in one bucket would be an unbounded array). MongoDB's **time series
collections** (lesson 09) implement this pattern for you inside the server, and are usually the better
choice for pure measurement data. The manual pattern is still useful when buckets need custom contents —
pre-aggregates, per-bucket metadata, or grouping that isn't by time.

## Attribute pattern: many optional fields become key–value pairs

**Pressure:** documents have many different, sparse fields that you need to query — product specs, where
laptops have `ram_gb` and kettles have `watts`.

One index per possible field doesn't scale. Instead, move them into an array of `{k, v}` pairs and index
the pair once:

```javascript
db.products.insertMany([
  { _id: 1, name: "Laptop A", specs: [{ k: "ram_gb", v: 16 }, { k: "screen_in", v: 14 }, { k: "cpu", v: "M-class" }] },
  { _id: 2, name: "Laptop B", specs: [{ k: "ram_gb", v: 32 }, { k: "screen_in", v: 16 }] },
  { _id: 3, name: "Kettle",   specs: [{ k: "litres", v: 1.7 }, { k: "watts", v: 2200 }] },
  { _id: 4, name: "Monitor",  specs: [{ k: "screen_in", v: 27 }, { k: "hz", v: 144 }] }]);
db.products.createIndex({ "specs.k": 1, "specs.v": 1 });
db.products.find({ specs: { $elemMatch: { k: "screen_in", v: { $gte: 15 } } } })
```

```text
Laptop B, Monitor
```

The plan uses the one index with tight bounds on both parts:

```text
"indexBounds":{"specs.k":["[\"screen_in\", \"screen_in\"]"],"specs.v":["[15, inf]"]}
2 2 2
```

Two keys examined, two documents fetched, two returned. The `$elemMatch` is essential: without it, `k` and
`v` could come from different pairs (Level 1 · 05), and the index bounds can't be combined. A wildcard
index (lesson 06) is an alternative when you'd rather keep specs as ordinary fields.

## Outlier pattern: design for the normal case, flag the exception

**Pressure:** 99.9% of documents have a small list, a few have an enormous one — the celebrity with ten
million followers, the bestseller with 500,000 buyers.

Don't distort the schema for everyone. Embed normally, and when a document's list passes a threshold,
move the overflow elsewhere and set a flag:

```javascript
{ _id: "book-123", title: "...", buyers: [ /* first 1000 */ ], hasOverflow: true }
// overflow documents: { bookId: "book-123", page: 1, buyers: [ ... next 1000 ... ] }
```

Application code checks `hasOverflow` and only then fetches the extra pages. Most reads stay one query.

## Polymorphic pattern: different shapes in one collection

**Pressure:** entities that are mostly alike but differ in details — payment methods (card, UPI, bank
transfer), notifications (email, SMS, push), catalogue items of different kinds.

Keep them in **one collection** with a discriminator field, because they're queried together ("all payment
methods of this user"):

```javascript
{ _id: 1, userId: "u42", type: "card", last4: "4242", expiry: "2028-04" }
{ _id: 2, userId: "u42", type: "upi",  vpa: "ana@bank" }
```

Index the shared fields (`userId`, `type`). Validate the variant-specific fields with a `oneOf` schema
(lesson 04).

## Schema versioning: change shape without downtime

**Pressure:** the document shape must change, but you can't stop the application to rewrite a billion
documents.

Add a `schemaVersion` field, teach the code to read every version, and upgrade documents **lazily** as
they're touched:

```python
def upgrade(doc):
    """Bring any stored version of a user up to version 2."""
    if doc.get("schemaVersion", 1) == 1:
        first, _, last = doc.pop("name").partition(" ")
        doc.update(firstName=first, lastName=last, schemaVersion=2)
    return doc

def get_user(uid):
    doc = users.find_one({"_id": uid})
    if doc and doc.get("schemaVersion", 1) < 2:
        doc = upgrade(doc)
        # Write back only if nobody else migrated it in the meantime.
        users.replace_one({"_id": uid, "schemaVersion": {"$exists": False}}, doc)
    return doc
```

With one version-1 user (`{_id: 1, name: "Ana Rao"}`) and one version-2 user:

```text
{'_id': 1, 'firstName': 'Ana', 'lastName': 'Rao', 'schemaVersion': 2}
{'_id': 2, 'schemaVersion': 2, 'firstName': 'Ben', 'lastName': 'Ito'}
0 documents still on version 1
```

A background job then migrates the long tail of documents nobody reads, after which the version-1 branch
can be deleted. Level 4 · 09 develops this into a full zero-downtime migration procedure.

## How It Actually Works

Most of these patterns trade **write-time work** for **read-time work**, and they pay off because of how
costs scale inside the server.

Every document carries fixed overhead: its BSON header and field names, a record in the collection's
B-tree, and one entry in **every** index (more for multikey indexes). In the bucket experiment, the flat
design had 100,800 records and 100,800 entries in the `{sensor, at}` index; the bucket design had 1,680 of
each. Index size is what decides whether lookups stay in memory, and fewer, larger documents also
compress better on disk, since similar values sit next to each other in one block.

Reads are dominated by the number of documents fetched, because each fetch is a separate B-tree lookup and
document load. The bucket query fetched 24 documents instead of 1,440; the attribute query fetched exactly
the matching documents because the index bounds were tight. Pre-computed values (the computed pattern)
move arithmetic from every read to the much rarer writes.

The limit is the document itself: everything in it is read and rewritten together, and it can't exceed
16 MB. Buckets must be bounded, subsets must be capped, outliers must overflow. Every pattern here is a
way of making documents **as large as the access pattern wants and no larger**.

## Common mistakes

- **Applying patterns before measuring a need.** Start simple; reach for a pattern when a specific
  pressure appears.
- **Unbounded buckets** — a "bucket per sensor" with no time boundary is just an unbounded array.
- **Attribute pattern without `$elemMatch`** in queries, so `k` and `v` match different pairs.
- **Computed values updated outside the write that changes their inputs**, drifting silently.
- **Extended references to fields that change often**, creating constant fan-out updates.
- **Schema versions without a plan to finish the migration**, leaving three code paths forever.

## Exercise

1. Build the flat and bucket designs for 50 sensors × 30 days of minute readings. Compare document count,
   data size and index size, and time "daily min/max per sensor for a month" on both.
2. Change the bucket size from one hour to one day. What happens to index size, document size and the
   hourly-average query?
3. Model a product catalogue with books, laptops and kettles using the polymorphic and attribute patterns
   together. Write a query for "laptops with at least 16 GB RAM and a screen under 15 inches" that uses one
   index.
4. Implement the outlier pattern for a `followers` array with a threshold of 1,000. Write `follow()` and
   `followersPage(userId, page)`.
5. Write the background migration job for the schema versioning example: process version-1 users in
   batches of 500 with `bulkWrite`, safely alongside the lazy migration.
