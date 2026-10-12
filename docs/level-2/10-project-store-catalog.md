---
description: "Level 2 project: model a 100,000-product catalogue with patterns and a validator, design ESR and partial indexes, and prove every query with explain()."
---

# 10 · Project — Model & Index a Store Catalog

This project takes everything in Level 2 and applies it to one realistic problem: the product catalogue of
an online store. You'll model it from its access patterns, enforce its shape with a validator, design one
index per hot query, and — the part most teams skip — write a script that **proves** every query uses its
index well, so a future change that breaks a query fails loudly instead of slowly.

## The access patterns

| # | Operation | Frequency | Shape |
|---|-----------|-----------|-------|
| Q1 | Category page: active products in a category, cheapest first, 24 per page | very high | equality, equality, sort |
| Q2 | Brand page: a brand's active products, best rated first | high | equality, equality, sort |
| Q3 | Spec filter: e.g. laptops with at least 32 GB RAM | high | equality + attribute range |
| Q4 | Product page by SKU | very high | unique equality |
| Q5 | Low-stock report: active products with fewer than 5 in stock | a few times a day | equality + range + sort on the range |

## The model

```javascript
{
  sku: "SKU-000123",
  name: "Zen Audio headphones 123",
  category: "headphones",
  brand: { _id: "zen", name: "Zen Audio" },          // extended reference (lesson 03)
  price: Decimal128("129.99"),                       // exact money (Level 1 · 03)
  status: "active",
  stock: 42,
  attrs: [ { k: "wireless", v: true }, { k: "anc", v: false } ],   // attribute pattern (lesson 03)
  rating: { count: 311, avg: 4.12 },                 // computed pattern (lesson 03)
  recentReviews: [ { stars: 5, at: ISODate("...") }, ... ]           // subset pattern, max 5 (lesson 03)
}
```

Each choice comes from Level 2:

- **Brand is an extended reference.** Every product listing shows the brand name, brands are renamed almost
  never, and `brand._id` still links to a full `brands` document for the brand page header.
- **Specs use the attribute pattern.** Four categories with different specs (and many more in a real store)
  share one `attrs_kv` index instead of one index per spec.
- **Ratings are pre-computed** (count + average) and the latest reviews are a bounded subset; the full review
  history would live in its own `reviews` collection (lesson 01's referencing rules).
- **SKU has a unique index**, and is what URLs use.

## The code

`catalog.py` creates the collection with its validator and indexes, loads 100,000 products with a seeded
random generator, and runs the query budget check. Run it with `--no-indexes` to see the "before" picture.

```python
"""Level 2 project: model, validate, index and verify a store catalogue."""
import random
from datetime import datetime, timedelta, timezone
from decimal import Decimal

from bson import Decimal128
from pymongo import ASCENDING, DESCENDING, MongoClient

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0", tz_aware=True)
db = client.catalog

SCHEMA = {"$jsonSchema": {
    "bsonType": "object",
    "required": ["sku", "name", "category", "brand", "price", "status", "attrs"],
    "properties": {
        "sku": {"bsonType": "string", "pattern": "^[A-Z]{3}-[0-9]{6}$"},
        "name": {"bsonType": "string", "minLength": 2},
        "category": {"enum": ["laptop", "phone", "kettle", "headphones"]},
        "brand": {"bsonType": "object", "required": ["_id", "name"],
                  "properties": {"_id": {"bsonType": "string"}, "name": {"bsonType": "string"}}},
        "price": {"bsonType": "decimal", "minimum": 0},
        "status": {"enum": ["active", "discontinued"]},
        "stock": {"bsonType": "int", "minimum": 0},
        "attrs": {"bsonType": "array", "maxItems": 20,
                  "items": {"bsonType": "object", "required": ["k", "v"]}},
        "rating": {"bsonType": "object", "properties": {"count": {"bsonType": "int"}, "avg": {"bsonType": "double"}}},
        "recentReviews": {"bsonType": "array", "maxItems": 5},
    }}}

INDEXES = [
    # Q1 category page: active products in a category, cheapest first
    ([("status", ASCENDING), ("category", ASCENDING), ("price", ASCENDING)], "status_category_price"),
    # Q2 brand page: a brand's active products, best rated first
    ([("brand._id", ASCENDING), ("status", ASCENDING), ("rating.avg", DESCENDING)], "brand_status_rating"),
    # Q3 spec filter: attribute pattern, one index for every spec
    ([("attrs.k", ASCENDING), ("attrs.v", ASCENDING)], "attrs_kv"),
    # Q5 low-stock report: partial index only over the few active products with low stock
    ([("stock", ASCENDING)], "low_stock"),
]

BRANDS = [("acme", "Acme"), ("zen", "Zen Audio"), ("nova", "Nova"), ("kettleco", "KettleCo"), ("orbit", "Orbit")]
ATTRS = {
    "laptop": lambda r: [{"k": "ram_gb", "v": r.choice([8, 16, 32, 64])}, {"k": "screen_in", "v": r.choice([13, 14, 15, 16])},
                         {"k": "weight_kg", "v": round(r.uniform(0.9, 2.5), 2)}],
    "phone": lambda r: [{"k": "storage_gb", "v": r.choice([128, 256, 512])}, {"k": "screen_in", "v": r.choice([6.1, 6.5, 6.7])}],
    "kettle": lambda r: [{"k": "litres", "v": r.choice([1.0, 1.5, 1.7])}, {"k": "watts", "v": r.choice([1500, 2200, 3000])}],
    "headphones": lambda r: [{"k": "wireless", "v": r.random() < 0.7}, {"k": "anc", "v": r.random() < 0.4}],
}


def setup(with_indexes=True):
    client.drop_database("catalog")
    db.create_collection("products", validator=SCHEMA)
    db.products.create_index("sku", unique=True, name="sku_unique")
    if not with_indexes:
        return
    for keys, name in INDEXES[:3]:
        db.products.create_index(keys, name=name)
    db.products.create_index(*[INDEXES[3][0]], name=INDEXES[3][1],
                             partialFilterExpression={"status": "active", "stock": {"$lt": 5}})


def load(n=100_000, seed=11):
    r = random.Random(seed)
    batch = []
    base = datetime(2026, 1, 1, tzinfo=timezone.utc)
    for i in range(n):
        cat = r.choice(list(ATTRS))
        bid, bname = r.choice(BRANDS)
        count = r.randint(0, 400)
        batch.append({
            "sku": f"SKU-{i:06d}", "name": f"{bname} {cat} {i}", "category": cat,
            "brand": {"_id": bid, "name": bname},
            "price": Decimal128(Decimal(r.randint(999, 249999)) / 100),
            "status": "active" if r.random() < 0.9 else "discontinued",
            "stock": r.randint(0, 200), "attrs": ATTRS[cat](r),
            "rating": {"count": count, "avg": round(r.uniform(2.5, 5.0), 2) if count else 0.0},
            "recentReviews": [{"stars": r.randint(1, 5), "at": base + timedelta(days=r.randint(0, 280))}
                              for _ in range(min(count, 5))],
        })
        if len(batch) == 5000:
            db.products.insert_many(batch, ordered=False)
            batch = []


QUERIES = {
    "Q1 category page": dict(filter={"status": "active", "category": "laptop"}, sort=[("price", 1)], limit=24),
    "Q2 brand page": dict(filter={"brand._id": "zen", "status": "active"}, sort=[("rating.avg", -1)], limit=24),
    "Q3 laptops 32GB+ RAM": dict(filter={"category": "laptop",
                                         "attrs": {"$elemMatch": {"k": "ram_gb", "v": {"$gte": 32}}}}, limit=24),
    "Q4 product page": dict(filter={"sku": "SKU-000123"}, limit=1),
    "Q5 low stock": dict(filter={"status": "active", "stock": {"$lt": 5}}, sort=[("stock", 1)], limit=50),
}


def stages(plan):
    out = []
    while plan:
        out.append(plan.get("stage"))
        plan = plan.get("inputStage")
    return out


def check():
    ok = True
    for label, q in QUERIES.items():
        cur = db.products.find(q["filter"]).limit(q["limit"])
        if "sort" in q:
            cur = cur.sort(q["sort"])
        ex = cur.explain()
        es = ex["executionStats"]
        st = stages(ex["queryPlanner"]["winningPlan"])
        idx = ex["queryPlanner"]["winningPlan"]
        while idx and "indexName" not in idx:
            idx = idx.get("inputStage")
        idx = idx["indexName"] if idx else "-"
        n, keys, docs = es["nReturned"], es["totalKeysExamined"], es["totalDocsExamined"]
        problems = []
        if "COLLSCAN" in st:
            problems.append("collection scan")
        if "SORT" in st:
            problems.append("blocking sort")
        if docs > max(4 * n, 10):
            problems.append(f"docs examined {docs} > 4x returned")
        ok &= not problems
        print(f"{label:21} {idx:22} {'<-'.join(st):28} n={n:3} keys={keys:6} docs={docs:6} {'OK' if not problems else 'FAIL: ' + ', '.join(problems)}")
    return ok


if __name__ == "__main__":
    import sys
    setup(with_indexes="--no-indexes" not in sys.argv)
    load()
    print("products:", db.products.estimated_document_count())
    print("all queries within budget:", check())
```

## The indexes, query by query

- **Q1** `{status: 1, category: 1, price: 1}` — two equalities then the sort field: pure ESR (lesson 05). The
  scan starts at "active laptops", walks in price order and stops after 24.
- **Q2** `{"brand._id": 1, status: 1, "rating.avg": -1}` — same shape. The sort direction matches the query's
  descending sort.
- **Q3** `{"attrs.k": 1, "attrs.v": 1}` — the attribute index, used with `$elemMatch` so `k` and `v` bound the
  scan together (lesson 03).
- **Q4** the unique `sku` index.
- **Q5** a **partial** index on `stock` covering only active products with stock under 5 (lesson 06). The
  report runs a few times a day; a full index on `stock` would be maintained on every one of the thousands of
  stock changes per hour for no benefit. Its query includes `status: "active"` and `stock: {$lt: 5}`, so the
  planner can prove the partial index holds every possible result.

## Before: no secondary indexes

```text
$ python catalog.py --no-indexes
products: 100000
Q1 category page      -                      SORT<-COLLSCAN               n= 24 keys=     0 docs=100000 FAIL: collection scan, blocking sort, docs examined 100000 > 4x returned
Q2 brand page         -                      SORT<-COLLSCAN               n= 24 keys=     0 docs=100000 FAIL: collection scan, blocking sort, docs examined 100000 > 4x returned
Q3 laptops 32GB+ RAM  -                      LIMIT<-COLLSCAN              n= 24 keys=     0 docs=   165 FAIL: collection scan, docs examined 165 > 4x returned
Q4 product page       sku_unique             EXPRESS_IXSCAN               n=  1 keys=     1 docs=     1 OK
Q5 low stock          -                      SORT<-COLLSCAN               n= 50 keys=     0 docs=100000 FAIL: collection scan, blocking sort, docs examined 100000 > 4x returned
all queries within budget: False
```

Three of the queries read the entire collection and sorted it in memory to return 24 or 50 documents. Q3 got
lucky: laptops with 32 GB or more are common, so the scan found 24 within 165 documents and stopped — on a
rarer spec it would have scanned much further. That is exactly the kind of query that looks fine in testing
and degrades in production, which is why the budget check fails it anyway.

## After: the designed indexes

```text
$ python catalog.py
products: 100000
Q1 category page      status_category_price  LIMIT<-FETCH<-IXSCAN         n= 24 keys=    24 docs=    24 OK
Q2 brand page         brand_status_rating    LIMIT<-FETCH<-IXSCAN         n= 24 keys=    24 docs=    24 OK
Q3 laptops 32GB+ RAM  attrs_kv               LIMIT<-FETCH<-IXSCAN         n= 24 keys=    24 docs=    24 OK
Q4 product page       sku_unique             EXPRESS_IXSCAN               n=  1 keys=     1 docs=     1 OK
Q5 low stock          low_stock              LIMIT<-FETCH<-IXSCAN         n= 50 keys=    50 docs=    50 OK
all queries within budget: True
```

Every query examines exactly as many keys and documents as it returns, with no blocking sort. That's the best
possible result short of covered queries.

What it costs:

```text
{'_id_': 1.69, 'sku_unique': 3.58, 'status_category_price': 2.2, 'brand_status_rating': 1.84, 'attrs_kv': 4.58, 'low_stock': 0.07} MB; data 48.4 MB
```

Under 14 MB of indexes for 48 MB of data. The partial `low_stock` index is 70 KB — a full index on `stock` would
have been about the size of the others. `attrs_kv` is the largest because it is multikey: two or three entries
per product.

## The validator at work

```python
db.products.insert_one({"sku": "SKU-1", "name": "Bad", "category": "toaster",
                        "brand": {"_id": "acme", "name": "Acme"}, "price": 9.99,
                        "status": "active", "attrs": []})
```

```text
sku -> regular expression did not match
category -> value was not found in enum
price -> type did not match
```

Three mistakes, three precise messages: a malformed SKU, an unknown category, and a price sent as a double
instead of Decimal128 — the bug that would otherwise have silently produced `0.30000000000000004`-style totals
months later.

## How It Actually Works

The budget check works because `explain("executionStats")` reports **what the server did**, not what it might
do: the winning plan's stages and the exact number of index keys and documents it touched. Checking those
numbers against simple rules — no `COLLSCAN`, no `SORT`, documents examined close to documents returned —
captures the properties that decide how a query scales. A query that examines 24 documents to return 24 will
still examine about 24 when the collection has 100 million products; one that examines 100,000 will examine
100 million.

That makes the check suitable for continuous integration: run it against a seeded dataset whenever the schema,
indexes or query code change. It catches the three regressions that most often reach production — a query
that gains a filter on an unindexed field, an index dropped as "unused", and a sort added without updating the
index. The thresholds should be per query (a report can tolerate more than a product page), and the dataset
must resemble production's distributions, or the planner may legitimately choose differently.

## Common mistakes

- **Designing indexes without listing the queries first.**
- **Testing query performance on tiny collections**, where every plan is fast.
- **One index per field** instead of one compound index per query shape.
- **Full indexes for rare, narrow reports** where a partial index would do.
- **No automated check**, so the next feature quietly reintroduces a collection scan.

## Exercise

Extend the catalogue:

1. Add Q6 "search by name prefix within a category" (e.g. names starting with "Acme lap"). Design the index,
   add the query to `QUERIES`, and make the budget check pass.
2. Add a `brands` collection and a brand page that shows the brand document plus Q2's products. Implement a
   brand rename that updates the extended references in products with one `update_many`.
3. Add a `reviews` collection and an `add_review(sku, stars, text)` function that inserts the review and
   updates `rating` and `recentReviews` on the product (computed + subset patterns). What can go wrong between
   the two writes? Keep your answer for Level 3 · 06.
4. Make the budget check run each query against **every** category or brand value and report the worst case,
   not just one value.
5. Add `"additionalProperties": false` to the schema. What else must change for inserts to keep working?
