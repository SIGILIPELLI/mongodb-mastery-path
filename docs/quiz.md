---
title: "Final Quiz"
description: "Test what you have learned across all four levels of MongoDB Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A document model fits when related data is read together and bounded. What does MongoDB recommend?",
    "options": [
      "Always reference",
      "Always embed",
      "Embed it in one document; reference when data is unbounded or shared",
      "Always normalize"
    ],
    "correctIndex": 2,
    "explanation": "Model for access patterns."
  },
  {
    "question": "An array in a document grows forever with every event. What's the risk and fix?",
    "options": [
      "None",
      "The 16 MB document limit and slow updates; use the bucket pattern or references",
      "Add RAM",
      "Index it"
    ],
    "correctIndex": 1,
    "explanation": "Unbounded arrays."
  },
  {
    "question": "How do you query a field inside an embedded document?",
    "options": [
      "Dot notation, e.g. find({\"address.city\": \"Oslo\"})",
      "address:city",
      "address[city]",
      "address->city"
    ],
    "correctIndex": 0,
    "explanation": "Quotes are required."
  },
  {
    "question": "You need an element that satisfies two conditions at once ( sku = pen AND qty > 5). Which operator?",
    "options": [
      "$in",
      "$elemMatch",
      "$and across fields",
      "$all"
    ],
    "correctIndex": 1,
    "explanation": "Conditions apply to the same element."
  },
  {
    "question": "What happens if `updateOne` is used with upsert: true and nothing matches?",
    "options": [
      "Update all",
      "Error",
      "A new document is inserted",
      "Nothing"
    ],
    "correctIndex": 2,
    "explanation": "Update-or-insert."
  },
  {
    "question": "How do you atomically increment a counter?",
    "options": [
      "$push",
      "$inc",
      "$max",
      "$set after read"
    ],
    "correctIndex": 1,
    "explanation": "No read-modify-write race."
  },
  {
    "question": "Why use Decimal128 for prices?",
    "options": [
      "Law",
      "Smaller",
      "Faster",
      "Binary floating point can't represent decimal fractions exactly"
    ],
    "correctIndex": 3,
    "explanation": "Avoid rounding errors."
  },
  {
    "question": "A query is slow and explain shows COLLSCAN with docsExamined 1,000,000 and nReturned 10. What now?",
    "options": [
      "More RAM",
      "Limit 10",
      "Shard",
      "Add an index matching the query (equality, sort, range)"
    ],
    "correctIndex": 3,
    "explanation": "Selectivity."
  },
  {
    "question": "A compound index {status:1, createdAt:-1} serves which query best?",
    "options": [
      "Text search",
      "Filter status = x sorted by createdAt descending",
      "Filter createdAt only",
      "Regex on status"
    ],
    "correctIndex": 1,
    "explanation": "Equality then sort."
  },
  {
    "question": "Which index automatically removes old documents?",
    "options": [
      "Hashed",
      "Partial",
      "Unique",
      "A TTL index"
    ],
    "correctIndex": 3,
    "explanation": "expireAfterSeconds."
  },
  {
    "question": "A pipeline filters with $match after a $group. How can it be faster?",
    "options": [
      "Add $sort",
      "Move $match first so it can use an index and reduce documents",
      "Add $project",
      "Remove $group"
    ],
    "correctIndex": 1,
    "explanation": "Filter early."
  },
  {
    "question": "What does $lookup do and what should be indexed?",
    "options": [
      "Inner join; nothing",
      "Union; _id",
      "Merge; none",
      "A left outer join; index the foreign field to avoid repeated scans"
    ],
    "correctIndex": 3,
    "explanation": "Or embed to avoid joins."
  },
  {
    "question": "When do you need a multi-document transaction?",
    "options": [
      "For reads",
      "When several documents/collections must change atomically (single-document updates are already atomic)",
      "Always",
      "Never"
    ],
    "correctIndex": 1,
    "explanation": "Embedding often avoids them."
  },
  {
    "question": "A service must react to changes in a collection in real time. What fits?",
    "options": [
      "Triggers in SQL",
      "Polling",
      "Cron",
      "Change streams (with resume tokens)"
    ],
    "correctIndex": 3,
    "explanation": "Requires a replica set."
  },
  {
    "question": "Which write concern avoids losing acknowledged writes after a failover?",
    "options": [
      "w: 1",
      "w: \"majority\"",
      "j: false",
      "w: 0"
    ],
    "correctIndex": 1,
    "explanation": "Durability trade-off."
  },
  {
    "question": "Why avoid a monotonically increasing shard key?",
    "options": [
      "Costly",
      "Too slow",
      "All new inserts hit one shard (a hotspot)",
      "Too random"
    ],
    "correctIndex": 2,
    "explanation": "Hashed keys spread writes."
  },
  {
    "question": "Which shard key trait is essential?",
    "options": [
      "High cardinality and even distribution aligned with queries",
      "Low cardinality",
      "Always growing",
      "Boolean"
    ],
    "correctIndex": 0,
    "explanation": "Avoid jumbo chunks."
  },
  {
    "question": "A replica set has 3 voting members. How many failures can it tolerate and still elect a primary?",
    "options": [
      "Three",
      "One (a majority of two must remain)",
      "Zero",
      "Two"
    ],
    "correctIndex": 1,
    "explanation": "Use odd numbers."
  },
  {
    "question": "How do you know a backup works?",
    "options": [
      "Check file size",
      "Trust",
      "Restore it regularly in a drill",
      "Copy it"
    ],
    "correctIndex": 2,
    "explanation": "Unrestored backups are unproven."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A mini query and aggregation engine

Implement matches (dot paths, comparison operators, $in, $elemMatch and array containment) and aggregate with $match, $unwind, $group ($sum, $max), $sort and $limit. The script exercises them on four orders.

**Starter code**

```javascript
import operator

OPS = {"$gt": operator.gt, "$gte": operator.ge, "$lt": operator.lt, "$lte": operator.le,
       "$eq": operator.eq, "$ne": operator.ne}

def get(doc, path):
    for part in path.split("."):
        if isinstance(doc, dict) and part in doc:
            doc = doc[part]
        else:
            return None
    return doc

def matches(doc, query):
    # supports equality, dot paths (via get), comparison operators, $in, $elemMatch, and array containment
    # ({tags: "gift"} matches when "gift" is in the array). Return True/False.
    return False

def aggregate(docs, pipeline):
    # support stages $match, $unwind, $group ($sum of 1 or a field, $max), $sort and $limit
    return docs

orders = [
    {"_id": 1, "customer": "ada", "status": "paid", "items": [{"sku": "pen", "qty": 3}, {"sku": "book", "qty": 1}], "total": 24.5, "tags": ["gift"]},
    {"_id": 2, "customer": "bo", "status": "paid", "items": [{"sku": "pen", "qty": 10}], "total": 15.0, "tags": []},
    {"_id": 3, "customer": "ada", "status": "refunded", "items": [{"sku": "book", "qty": 2}], "total": 40.0, "tags": ["gift", "late"]},
    {"_id": 4, "customer": "cy", "status": "paid", "items": [{"sku": "lamp", "qty": 1}], "total": 60.0, "tags": []},
]
print([d["_id"] for d in orders if matches(d, {"status": "paid", "total": {"$gte": 20}})])
print([d["_id"] for d in orders if matches(d, {"tags": "gift"})])
print([d["_id"] for d in orders if matches(d, {"items": {"$elemMatch": {"sku": "pen", "qty": {"$gt": 5}}}})])
print(aggregate(orders, [{"$match": {"status": "paid"}}, {"$group": {"_id": "$customer", "spent": {"$sum": "$total"}}}, {"$sort": {"spent": -1}}]))
lines = aggregate(orders, [{"$unwind": "$items"}])
flat = [{"sku": d["items"]["sku"], "qty": d["items"]["qty"]} for d in lines]
print(aggregate(flat, [{"$group": {"_id": "$sku", "units": {"$sum": "$qty"}}}, {"$sort": {"units": -1, "_id": 1}}, {"$limit": 2}]))
```

**Expected output**

```text
[1, 4]
[1, 3]
[2]
[{'_id': 'cy', 'spent': 60.0}, {'_id': 'ada', 'spent': 24.5}, {'_id': 'bo', 'spent': 15.0}]
[{'_id': 'pen', 'units': 13}, {'_id': 'book', 'units': 3}]
```

<details><summary>Show solution and explanation</summary>

<pre><code>import operator

OPS = {&quot;$gt&quot;: operator.gt, &quot;$gte&quot;: operator.ge, &quot;$lt&quot;: operator.lt, &quot;$lte&quot;: operator.le,
       &quot;$eq&quot;: operator.eq, &quot;$ne&quot;: operator.ne}

def get(doc, path):
    for part in path.split(&quot;.&quot;):
        if isinstance(doc, dict) and part in doc:
            doc = doc[part]
        else:
            return None
    return doc

def matches(doc, query):
    for field, cond in query.items():
        value = get(doc, field)
        if isinstance(cond, dict):
            for op, arg in cond.items():
                if op == &quot;$in&quot;:
                    ok = value in arg
                elif op == &quot;$elemMatch&quot;:
                    ok = isinstance(value, list) and any(matches(e, arg) for e in value)
                else:
                    ok = value is not None and OPS[op](value, arg)
                if not ok:
                    return False
        elif isinstance(value, list):
            if cond not in value:                      # array containment, like {tags: &quot;x&quot;}
                return False
        elif value != cond:
            return False
    return True

def aggregate(docs, pipeline):
    for stage in pipeline:
        (name, spec), = stage.items()
        if name == &quot;$match&quot;:
            docs = [d for d in docs if matches(d, spec)]
        elif name == &quot;$unwind&quot;:
            field = spec.lstrip(&quot;$&quot;)
            docs = [{**d, field: v} for d in docs for v in d.get(field, [])]
        elif name == &quot;$group&quot;:
            groups = {}
            for d in docs:
                key = d[spec[&quot;_id&quot;].lstrip(&quot;$&quot;)]
                g = groups.setdefault(key, {&quot;_id&quot;: key})
                for out, acc in spec.items():
                    if out == &quot;_id&quot;:
                        continue
                    (op, arg), = acc.items()
                    if op == &quot;$sum&quot;:
                        g[out] = g.get(out, 0) + (1 if arg == 1 else d[arg.lstrip(&quot;$&quot;)])
                    elif op == &quot;$max&quot;:
                        v = d[arg.lstrip(&quot;$&quot;)]; g[out] = v if out not in g else max(g[out], v)
            docs = list(groups.values())
        elif name == &quot;$sort&quot;:
            for field, direction in reversed(list(spec.items())):
                docs = sorted(docs, key=lambda d: d[field], reverse=direction &lt; 0)
        elif name == &quot;$limit&quot;:
            docs = docs[:spec]
    return docs

orders = [
    {&quot;_id&quot;: 1, &quot;customer&quot;: &quot;ada&quot;, &quot;status&quot;: &quot;paid&quot;, &quot;items&quot;: [{&quot;sku&quot;: &quot;pen&quot;, &quot;qty&quot;: 3}, {&quot;sku&quot;: &quot;book&quot;, &quot;qty&quot;: 1}], &quot;total&quot;: 24.5, &quot;tags&quot;: [&quot;gift&quot;]},
    {&quot;_id&quot;: 2, &quot;customer&quot;: &quot;bo&quot;, &quot;status&quot;: &quot;paid&quot;, &quot;items&quot;: [{&quot;sku&quot;: &quot;pen&quot;, &quot;qty&quot;: 10}], &quot;total&quot;: 15.0, &quot;tags&quot;: []},
    {&quot;_id&quot;: 3, &quot;customer&quot;: &quot;ada&quot;, &quot;status&quot;: &quot;refunded&quot;, &quot;items&quot;: [{&quot;sku&quot;: &quot;book&quot;, &quot;qty&quot;: 2}], &quot;total&quot;: 40.0, &quot;tags&quot;: [&quot;gift&quot;, &quot;late&quot;]},
    {&quot;_id&quot;: 4, &quot;customer&quot;: &quot;cy&quot;, &quot;status&quot;: &quot;paid&quot;, &quot;items&quot;: [{&quot;sku&quot;: &quot;lamp&quot;, &quot;qty&quot;: 1}], &quot;total&quot;: 60.0, &quot;tags&quot;: []},
]
print([d[&quot;_id&quot;] for d in orders if matches(d, {&quot;status&quot;: &quot;paid&quot;, &quot;total&quot;: {&quot;$gte&quot;: 20}})])
print([d[&quot;_id&quot;] for d in orders if matches(d, {&quot;tags&quot;: &quot;gift&quot;})])
print([d[&quot;_id&quot;] for d in orders if matches(d, {&quot;items&quot;: {&quot;$elemMatch&quot;: {&quot;sku&quot;: &quot;pen&quot;, &quot;qty&quot;: {&quot;$gt&quot;: 5}}}})])
print(aggregate(orders, [{&quot;$match&quot;: {&quot;status&quot;: &quot;paid&quot;}}, {&quot;$group&quot;: {&quot;_id&quot;: &quot;$customer&quot;, &quot;spent&quot;: {&quot;$sum&quot;: &quot;$total&quot;}}}, {&quot;$sort&quot;: {&quot;spent&quot;: -1}}]))
lines = aggregate(orders, [{&quot;$unwind&quot;: &quot;$items&quot;}])
flat = [{&quot;sku&quot;: d[&quot;items&quot;][&quot;sku&quot;], &quot;qty&quot;: d[&quot;items&quot;][&quot;qty&quot;]} for d in lines]
print(aggregate(flat, [{&quot;$group&quot;: {&quot;_id&quot;: &quot;$sku&quot;, &quot;units&quot;: {&quot;$sum&quot;: &quot;$qty&quot;}}}, {&quot;$sort&quot;: {&quot;units&quot;: -1, &quot;_id&quot;: 1}}, {&quot;$limit&quot;: 2}]))
</code></pre>

<p>MongoDB&#x27;s query language is a set of predicates over documents and arrays, and aggregation is a pipeline of stages each transforming the document list. This toy engine mirrors the semantics, not the implementation. Verified by running the solution.</p>
</details>

### Why shard-key choice matters

Implement shard_for for hashed and ranged strategies and compare how historical and new monotonically increasing keys distribute across three shards. Observe the hotspot with ranged keys and the even spread with hashed keys.

**Starter code**

```javascript
import hashlib
from collections import Counter

def shard_for(key, shards, strategy, ranges=None):
    # "hashed": md5 of str(key) as an int, modulo shards; "ranged": index of the first upper bound
    # greater than key (the last shard when above every bound); ValueError for unknown strategies
    return 0

def distribution(keys, shards, strategy, ranges=None):
    c = Counter(shard_for(k, shards, strategy, ranges) for k in keys)
    return [c.get(i, 0) for i in range(shards)]

def imbalance(counts):
    return round(max(counts) / (sum(counts) / len(counts)), 2)

N = 3000
monotonic = list(range(N))                      # an ever-increasing key, e.g. a timestamp or counter
ranges = [1000, 2000]                           # 3 chunks created from the initial data
print("ranged, historical data:", distribution(monotonic, 3, "ranged", ranges))
new_inserts = list(range(N, N + 900))           # new writes are all above the highest range
print("ranged, new inserts:    ", distribution(new_inserts, 3, "ranged", ranges), "-> hotspot")
print("hashed, new inserts:    ", distribution(new_inserts, 3, "hashed"))
print("imbalance ranged/hashed:", imbalance(distribution(new_inserts, 3, "ranged", ranges)), imbalance(distribution(new_inserts, 3, "hashed")))
status = ["paid"] * 2900 + ["refunded"] * 100   # a low-cardinality key can't spread
print("low cardinality key:", sorted(set(status)), "-> at most", len(set(status)), "chunks")
```

**Expected output**

```text
ranged, historical data: [1000, 1000, 1000]
ranged, new inserts:     [0, 0, 900] -> hotspot
hashed, new inserts:     [285, 293, 322]
imbalance ranged/hashed: 3.0 1.07
low cardinality key: ['paid', 'refunded'] -> at most 2 chunks
```

<details><summary>Show solution and explanation</summary>

<pre><code>import hashlib
from collections import Counter

def shard_for(key, shards, strategy, ranges=None):
    if strategy == &quot;hashed&quot;:
        h = int(hashlib.md5(str(key).encode()).hexdigest(), 16)
        return h % shards
    if strategy == &quot;ranged&quot;:
        for i, upper in enumerate(ranges):
            if key &lt; upper:
                return i
        return len(ranges)
    raise ValueError(strategy)

def distribution(keys, shards, strategy, ranges=None):
    c = Counter(shard_for(k, shards, strategy, ranges) for k in keys)
    return [c.get(i, 0) for i in range(shards)]

def imbalance(counts):
    return round(max(counts) / (sum(counts) / len(counts)), 2)

N = 3000
monotonic = list(range(N))                      # an ever-increasing key, e.g. a timestamp or counter
ranges = [1000, 2000]                           # 3 chunks created from the initial data
print(&quot;ranged, historical data:&quot;, distribution(monotonic, 3, &quot;ranged&quot;, ranges))
new_inserts = list(range(N, N + 900))           # new writes are all above the highest range
print(&quot;ranged, new inserts:    &quot;, distribution(new_inserts, 3, &quot;ranged&quot;, ranges), &quot;-&gt; hotspot&quot;)
print(&quot;hashed, new inserts:    &quot;, distribution(new_inserts, 3, &quot;hashed&quot;))
print(&quot;imbalance ranged/hashed:&quot;, imbalance(distribution(new_inserts, 3, &quot;ranged&quot;, ranges)), imbalance(distribution(new_inserts, 3, &quot;hashed&quot;)))
status = [&quot;paid&quot;] * 2900 + [&quot;refunded&quot;] * 100   # a low-cardinality key can&#x27;t spread
print(&quot;low cardinality key:&quot;, sorted(set(status)), &quot;-&gt; at most&quot;, len(set(status)), &quot;chunks&quot;)
</code></pre>

<p>With a ranged, ever-increasing key every new insert lands in the last chunk and one shard takes all the writes, while a hashed key spreads them evenly (at the cost of efficient range queries). A low-cardinality key can&#x27;t be split into more chunks than it has values. Verified by running the solution.</p>
</details>
