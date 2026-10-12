---
description: "Level 3 project: an order-analytics backend with a $merge rollup synced by a change-stream worker, a $facet dashboard, $lookup best-sellers and tests."
---

# 10 · Project — Order Analytics Backend

The Level 3 project builds the backend of a sales dashboard over the `shop` database from lesson 01. It pulls
together most of the level: aggregation stages and expressions, a `$lookup`, a `$facet` dashboard with a window
function, a `$merge` rollup, and a change-stream worker that keeps the rollup correct as orders change — with a
test suite that proves the rollup always agrees with the raw data.

## Requirements

1. **A dashboard query** for a date range (and optional city): total revenue and units, revenue by category, and
   daily revenue with a 7-day moving average. It runs every few seconds per open dashboard, so it must not scan raw
   orders.
2. **Best-selling products** for a range, with product names.
3. **Days are Indian Standard Time days**, everywhere.
4. **The dashboard must reflect changes within seconds**: new deliveries, cancellations, returns, deletions, and
   late corrections to old orders.
5. **The numbers must be exactly right** — the rollup must equal an aggregation over raw orders.

## Design

- `daily_sales` is a rollup keyed by `{day, city, category}` with `units`, `revenue` and distinct `orders` — the
  lesson 05 rollup plus an order count. The dashboard reads only this collection (requirement 1).
- `refresh_days(days)` recomputes whole IST days. It **deletes** the days' rollup documents first, then re-runs the
  rollup pipeline for them with `$merge`. Deleting first fixes the "group became empty" edge case from lesson 05; working
  in whole days makes refreshes idempotent.
- `RollupWorker` is a change-stream consumer on `orders` (lesson 07). For each batch of events it collects the IST days
  of the affected orders — from the post-image for inserts and updates and from the **pre-image** for deletes (and for
  updates that moved an order's date) — and refreshes just those days.
- `top_products` aggregates raw orders (it's requested rarely) and uses `$lookup` with the concise
  `localField`/`foreignField` + `pipeline` form to fetch only product names (lesson 03).

## The code

`analytics.py`:

```python
"""Level 3 project: an order-analytics backend over the `shop` database.

- daily_sales: rollup per (IST day, city, category), maintained with $merge
- RollupWorker: change-stream consumer that refreshes only the days that changed
- dashboard(): one $facet query over the rollup for the UI
- top_products(): best sellers joined with product names via $lookup
"""
import threading
from decimal import Decimal
from datetime import datetime, timedelta, timezone

from pymongo import MongoClient

TZ = "Asia/Kolkata"
IST = timezone(timedelta(hours=5, minutes=30))


def ist_day_start(dt):
    """Start of the IST calendar day containing dt, as a UTC datetime."""
    local = dt.astimezone(IST)
    return local.replace(hour=0, minute=0, second=0, microsecond=0).astimezone(timezone.utc)


class Analytics:
    def __init__(self, client: MongoClient, db="shop"):
        self.db = client[db]

    # ---- rollup -------------------------------------------------------------
    def _rollup_pipeline(self, start, end):
        return [
            {"$match": {"status": "delivered", "placedAt": {"$gte": start, "$lt": end}}},
            {"$unwind": "$items"},
            {"$group": {"_id": {"day": {"$dateTrunc": {"date": "$placedAt", "unit": "day", "timezone": TZ}},
                                "city": "$city", "category": "$items.category"},
                        "units": {"$sum": "$items.qty"},
                        "revenue": {"$sum": {"$multiply": ["$items.qty", "$items.price"]}},
                        "orders": {"$addToSet": "$_id"}}},
            {"$set": {"orders": {"$size": "$orders"}}},
            {"$merge": {"into": "daily_sales", "on": "_id", "whenMatched": "replace", "whenNotMatched": "insert"}},
        ]

    def refresh_days(self, days):
        """Recompute the rollup for whole IST days (idempotent)."""
        for day in sorted(set(days)):
            start, end = day, day + timedelta(days=1)
            # remove the window first so groups that became empty disappear
            self.db.daily_sales.delete_many({"_id.day": {"$gte": start, "$lt": end}})
            self.db.orders.aggregate(self._rollup_pipeline(start, end))

    def rebuild(self):
        self.db.daily_sales.drop()
        first = self.db.orders.find_one(sort=[("placedAt", 1)])["placedAt"].replace(tzinfo=timezone.utc)
        last = self.db.orders.find_one(sort=[("placedAt", -1)])["placedAt"].replace(tzinfo=timezone.utc)
        self.db.orders.aggregate(self._rollup_pipeline(ist_day_start(first), ist_day_start(last) + timedelta(days=1)))

    # ---- queries ----------------------------------------------------------------
    def dashboard(self, start, end, city=None):
        match = {"_id.day": {"$gte": start, "$lt": end}}
        if city:
            match["_id.city"] = city
        res = list(self.db.daily_sales.aggregate([
            {"$match": match},
            {"$facet": {
                "totals": [{"$group": {"_id": None, "revenue": {"$sum": "$revenue"}, "units": {"$sum": "$units"}}}],
                "byCategory": [{"$group": {"_id": "$_id.category", "revenue": {"$sum": "$revenue"}}},
                               {"$sort": {"revenue": -1}}],
                "byDay": [{"$group": {"_id": "$_id.day", "revenue": {"$sum": "$revenue"}}}, {"$sort": {"_id": 1}},
                          {"$setWindowFields": {"sortBy": {"_id": 1}, "output": {
                              "ma7": {"$avg": "$revenue", "window": {"range": [-6, 0], "unit": "day"}}}}}],
            }}]))[0]
        dec = lambda v: v.to_decimal() if hasattr(v, "to_decimal") else Decimal(v)   # Decimal128 -> Decimal
        totals = res["totals"][0] if res["totals"] else {"revenue": 0, "units": 0}
        return {"revenue": dec(totals["revenue"]), "units": totals["units"],
                "byCategory": [{"category": c["_id"], "revenue": dec(c["revenue"])} for c in res["byCategory"]],
                "byDay": [{"day": d["_id"], "revenue": dec(d["revenue"]), "ma7": dec(d["ma7"])} for d in res["byDay"]]}

    def top_products(self, start, end, n=5):
        return list(self.db.orders.aggregate([
            {"$match": {"status": "delivered", "placedAt": {"$gte": start, "$lt": end}}},
            {"$unwind": "$items"},
            {"$group": {"_id": "$items.sku", "units": {"$sum": "$items.qty"}}},
            {"$sort": {"units": -1, "_id": 1}}, {"$limit": n},
            {"$lookup": {"from": "products", "localField": "_id", "foreignField": "_id",
                         "pipeline": [{"$project": {"_id": 0, "name": 1}}], "as": "p"}},
            {"$project": {"_id": 0, "sku": "$_id", "units": 1, "name": {"$first": "$p.name"}}},
        ]))


class RollupWorker(threading.Thread):
    """Watches `orders` and refreshes the rollup for the IST days touched by each batch of changes."""

    def __init__(self, analytics: Analytics):
        super().__init__(daemon=True)
        self.a = analytics
        self.stop_flag = threading.Event()
        self.ready = threading.Event()
        self.batches = 0

    def run(self):
        pipeline = [{"$match": {"operationType": {"$in": ["insert", "update", "replace", "delete"]}}}]
        with self.a.db.orders.watch(pipeline, full_document="updateLookup",
                                    full_document_before_change="whenAvailable",
                                    max_await_time_ms=200) as stream:
            self.ready.set()
            while not self.stop_flag.is_set():
                days = set()
                while (ch := stream.try_next()) is not None:
                    for doc in (ch.get("fullDocument"), ch.get("fullDocumentBeforeChange")):
                        if doc:
                            days.add(ist_day_start(doc["placedAt"].replace(tzinfo=timezone.utc)))
                if days:
                    self.a.refresh_days(days)
                    self.batches += 1

    def stop(self):
        self.stop_flag.set()
        self.join()
```

### A failure on the way

The first version of `dashboard()` returned the rollup's values as they came from PyMongo, and one test failed:

```text
E       TypeError: '>' not supported between instances of 'Decimal128' and 'int'
FAILED test_analytics.py::test_dashboard_shape - TypeError: '>' not supported...
```

`bson.Decimal128` is a storage type, not a number: Python can't compare or add it. A backend should hand its callers
real `decimal.Decimal` values (or strings, for JSON), so the final version converts every money value with
`to_decimal()` at the boundary — the `dec` helper above.

## The tests

`test_analytics.py` regenerates nothing itself — run `python shopgen.py` from lesson 01 first. The module fixture enables
pre-images on `orders` and rebuilds the rollup.

```python
import random, time
from datetime import datetime, timezone
from decimal import Decimal

import pytest
from bson import Decimal128
from pymongo import MongoClient

from analytics import Analytics, RollupWorker

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0", tz_aware=True)
A = Analytics(client)
APR, JUL = datetime(2026, 3, 31, 18, 30, tzinfo=timezone.utc), datetime(2026, 6, 30, 18, 30, tzinfo=timezone.utc)


def raw_revenue_by_category(start, end):
    rows = client.shop.orders.aggregate([
        {"$match": {"status": "delivered", "placedAt": {"$gte": start, "$lt": end}}},
        {"$unwind": "$items"},
        {"$group": {"_id": "$items.category", "revenue": {"$sum": {"$multiply": ["$items.qty", "$items.price"]}}}}])
    return {r["_id"]: r["revenue"].to_decimal() for r in rows}


def rollup_revenue_by_category(start, end):
    return {r["category"]: r["revenue"] for r in A.dashboard(start, end)["byCategory"]}


@pytest.fixture(scope="module", autouse=True)
def built():
    client.shop.command("collMod", "orders", changeStreamPreAndPostImages={"enabled": True})
    A.rebuild()


def test_rollup_matches_raw():
    assert rollup_revenue_by_category(APR, JUL) == raw_revenue_by_category(APR, JUL)


def test_dashboard_shape():
    d = A.dashboard(APR, JUL, city="Pune")
    assert d["revenue"] > 0 and len(d["byCategory"]) == 4
    assert len(d["byDay"]) == 91                       # every IST day of April-June had Pune sales
    assert all("ma7" in x for x in d["byDay"])


def test_empty_range_returns_zero():
    d = A.dashboard(datetime(2030, 1, 1, tzinfo=timezone.utc), datetime(2030, 2, 1, tzinfo=timezone.utc))
    assert d["revenue"] == 0 and d["byCategory"] == []


def test_top_products_have_names():
    top = A.top_products(APR, JUL, n=3)
    assert len(top) == 3 and all(t["name"] for t in top)
    assert top[0]["units"] >= top[1]["units"] >= top[2]["units"]


def test_worker_keeps_rollup_in_sync():
    w = RollupWorker(A); w.start(); w.ready.wait(5)
    r = random.Random(5)
    orders = client.shop.orders
    ids = [d["_id"] for d in orders.find({"placedAt": {"$gte": APR, "$lt": JUL}}, {"_id": 1})]
    for oid in r.sample(ids, 40):                      # status flips in both directions
        new = r.choice(["delivered", "cancelled", "returned"])
        orders.update_one({"_id": oid}, {"$set": {"status": new}})
    for oid in r.sample(ids, 5):
        orders.delete_one({"_id": oid})
    orders.insert_one({"_id": 999999, "customerId": "C0001", "city": "Pune", "status": "delivered",
                       "placedAt": datetime(2026, 5, 15, 12, 0, tzinfo=timezone.utc),
                       "items": [{"sku": "P001", "category": "books", "qty": 2, "price": Decimal128(Decimal("500"))}]})
    deadline = time.time() + 10
    while time.time() < deadline:
        if rollup_revenue_by_category(APR, JUL) == raw_revenue_by_category(APR, JUL):
            break
        time.sleep(0.2)
    w.stop()
    assert rollup_revenue_by_category(APR, JUL) == raw_revenue_by_category(APR, JUL)
    assert w.batches >= 1
```

The important test is the last one. It starts the worker, then makes 40 random status changes, 5 deletions and one new
delivered order, and waits (up to 10 seconds) for the rollup to agree with a fresh aggregation over raw orders, to the
rupee, for every category.

```text
test_analytics.py::test_rollup_matches_raw PASSED                        [ 20%]
test_analytics.py::test_dashboard_shape PASSED                           [ 40%]
test_analytics.py::test_empty_range_returns_zero PASSED                  [ 60%]
test_analytics.py::test_top_products_have_names PASSED                   [ 80%]
test_analytics.py::test_worker_keeps_rollup_in_sync PASSED               [100%]
============================== 5 passed in 1.24s ===============================
```

## Using it

```python
a = Analytics(MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0", tz_aware=True))
APR, JUL = datetime(2026, 3, 31, 18, 30, tzinfo=timezone.utc), datetime(2026, 6, 30, 18, 30, tzinfo=timezone.utc)
d = a.dashboard(APR, JUL, city="Pune")
```

On a freshly generated dataset:

```text
revenue: 34645411 units: 3431
byCategory: [('electronics', '30280520'), ('kitchen', '2316538'), ('toys', '1268685'), ('books', '779668')]
first days: [('2026-04-01', '490473'), ('2026-04-02', '783530'), ('2026-04-03', '668140')]
top: [{'units': 135, 'sku': 'P162', 'name': 'Books item 162'}, {'units': 128, 'sku': 'P140', 'name': 'Kitchen item 140'}, {'units': 127, 'sku': 'P047', 'name': 'Electronics item 47'}]
```

`APR` and `JUL` are midnight IST on 1 April and 1 July expressed in UTC (18:30 the previous day) — requirement 3 in
practice. The first days' values are the 7-day moving averages, which are short windows at the start of the range.

## How It Actually Works

Correctness comes from three properties working together.

**Idempotent refresh.** `refresh_days` doesn't add or subtract deltas; it recomputes a day from the source of truth and
replaces whatever was there. Running it once, twice, or after a crash half-way through always converges on the right
answer. That's what makes it safe to drive from an at-least-once event source.

**Complete change capture.** The worker must learn about *every* day whose numbers could have changed. Inserts and
updates carry the new document (`updateLookup`); deletes carry nothing about the document except its `_id` — which is
why the collection has pre-images enabled and the worker reads `fullDocumentBeforeChange`. An update that changes an
order's `placedAt` affects two days, the old and the new, and pre- plus post-images give both.

**Batching.** The worker drains all available events, collects the distinct days, and refreshes each once. A burst of
1,000 changes to today's orders becomes one refresh of one day, not 1,000.

What this design doesn't yet do: persist its resume token (a restarted worker would miss changes made while it was down
— lesson 07 shows the fix; a nightly full `rebuild()` is a cheap safety net), or survive running two workers at once
(both would refresh the same days — harmless because refreshes are idempotent, but wasteful).

## Common mistakes

- **Applying deltas** (`$inc` revenue on each event) instead of recomputing — any missed or duplicated event corrupts the
  rollup permanently.
- **Ignoring deletes** because the change event has no document.
- **Mixing UTC and local days** between the rollup, the refresh and the queries.
- **Returning `Decimal128` to application code** that expects numbers.
- **Testing a rollup only right after a rebuild**, never after changes.

## Exercise

1. Persist the worker's resume token in a `worker_state` collection, updated after each batch, and resume from it on start.
   Test it by stopping the worker, making changes, and restarting.
2. Add an `orders` count and average order value to `dashboard()`'s totals (the rollup's `orders` field counts distinct
   orders per day/city/category — think about why summing it across categories over-counts, and fix it).
3. Add a `customer_monthly` rollup (customer, IST month → orders, revenue) maintained by the same worker, and a
   `top_customers(month)` query.
4. Make `top_products` fast for wide date ranges by maintaining a `daily_product_sales` rollup instead of aggregating raw
   orders.
5. Write a load test: a thread that updates 50 random orders per second for 30 seconds while the worker runs. Measure the
   maximum time between a change and the rollup reflecting it.
