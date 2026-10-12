---
description: "The core MongoDB modeling decision — embed related data or reference it — measured: read cost, document growth, the 16 MB wall, duplication and atomicity."
---

# 01 · Thinking in Documents: Embed or Reference?

In a relational database, the schema follows the data: you normalise into tables, and queries join
them back together. In MongoDB the schema follows the **access patterns**: you shape documents around
how the application reads and writes them. The central decision, made over and over, is whether related
data lives **inside** a document (embedding) or in **another** document that you point to (referencing).

There is no universal answer, but there is a method — and the trade-offs can be measured. That's what
this lesson does.

## The question to ask first

Before drawing any schema, list the application's operations and how often each runs:

| Operation | Frequency | Data needed |
|-----------|-----------|-------------|
| Show a blog post page | very high | post + its comments |
| Show the homepage list | very high | titles of recent posts |
| Add a comment | medium | one post |
| Show a user's comment history | low | comments by one user across posts |
| Rename a user | rare | every place the name appears |

The schema that wins is the one that makes the **frequent** operations cheap, while keeping the rare
ones possible.

## Experiment: the post page, two ways

A Python script built the same data twice — 2,000 posts with 20 comments each:

- `posts_embedded`: each post holds its comments in an array.
- `posts` + `comments`: comments are separate documents with a `postId` field and an index on
  `{postId: 1, at: 1}`.

```python
embedded.append({"_id": p, "title": f"Post {p}", "body": "Lorem " * 200, "comments": cs})
posts.append({"_id": p, "title": f"Post {p}", "body": "Lorem " * 200})
comments.extend({"postId": p, **c} for c in cs)
...
e = timeit(lambda i: db.posts_embedded.find_one({"_id": i}))
r = timeit(lambda i: (db.posts.find_one({"_id": i}),
                      list(db.comments.find({"postId": i}).sort("at", 1))))
```

3,000 random page views each, on the local lab:

```text
embedded:   0.127 ms per page view (1 query)
referenced: 0.281 ms per page view (2 queries)
```

Embedding was a little over twice as fast here, and that is with the database on the same machine. Over
a network each extra query adds a round trip, so the gap grows with latency. When data is **always read
together**, embedding wins.

But the sizes tell the other side of the story:

```text
posts_embedded  docs=  2000  avgObjSize= 3275 B  size=6.6 MB
posts           docs=  2000  avgObjSize= 1245 B  size=2.5 MB
comments        docs= 40000  avgObjSize=  126 B  size=5.1 MB
```

Every read of an embedded post — even one that only wants the title — makes the server fetch a 3.2 KB
document into memory instead of a 1.2 KB one. Projection reduces what is *sent*, not what is *read*:

```text
title only from embedded (projection): 0.120 ms
```

Almost exactly the cost of fetching the whole thing. For a homepage listing 50 titles, you're pulling
the comments of 50 posts into the cache for nothing. When some frequent operation needs the parent
**without** the child data, embedding costs memory.

## Experiment: what happens when a list never stops growing

One post goes viral. Push comments to it in batches of 10,000:

```javascript
db.viral.insertOne({ _id: 1, title: "Viral post", comments: [] });
const batch = Array.from({ length: 10000 }, (_, i) =>
  ({ author: "user" + (i % 500), text: "Great point, thanks for sharing this!", at: new Date() }));
let n = 0;
try { while (true) { db.viral.updateOne({ _id: 1 }, { $push: { comments: { $each: batch } } }); n += 10000; } }
catch (e) { print("failed after", n, "comments:", e.code, e.message) }
```

```text
failed after 180000 comments: 10334 Plan executor error during update :: caused by :: Resulting document after update is larger than 16777216
```

The 16 MB limit stopped it at 180,000 comments — and from that moment no one can comment on the post.
Long before the wall, the document was already a problem. At 16.5 MB:

```text
20 small updates on the huge doc ms: 305
20 small updates on a small doc ms: 103
20 title reads on the huge doc ms: 129
20 title reads on a small doc ms: 7
```

Reading the title of the huge post was about 18 times slower than reading a normal one, because the
whole document is loaded; every change to it rewrites a large document and, on a replica set, can
produce large oplog entries. **Unbounded arrays are the most common MongoDB modeling mistake.**

## Duplication: the price of embedding things that change

If each embedded comment stores the commenter's display name, renaming a user means updating every
comment they ever wrote, across every post. With references, you update one user document. This is the
real trade-off behind normalisation, and the question is: **how often does the duplicated data change,
and does it matter if old copies stay stale?**

- A product's price at the time of an order *should* be copied into the order and never updated — it is
  history, not a reference.
- A user's display name next to their comments probably should update, but a few seconds of
  inconsistency during a background fix-up is fine.
- A user's email used for login must exist in exactly one place.

## Atomicity is a reason to embed

A single-document write is atomic (Level 1 · 06). If an order and its line items are one document,
"create the order with its items" can't half-succeed, and "change a line's quantity and the order total"
is one update. Split them across collections and you need a multi-document transaction (Level 3 · 06)
to get the same guarantee. Data that must change **together** is a strong signal to embed.

## The rules of thumb

Embed when:

- the child data is read **with** the parent most of the time;
- the relationship is **one-to-few**, or the number of children is **bounded** (a product has a few
  variants; a user has a few addresses);
- the child doesn't need to be queried or updated independently at high volume;
- the parent and child should change **atomically**.

Reference when:

- the child list is **unbounded** or can grow large (comments on a public post, log events, orders of a
  customer over years);
- the child is **shared** by many parents and changes (users, products, tags with metadata);
- frequent operations need the parent **without** the children, or the children without the parent;
- the child is large and rarely needed (full-resolution file metadata, raw payloads).

And very often the right answer is **both**: reference the full data, embed a small, bounded,
denormalised subset for the hot read path — the post embeds its latest 10 comments and a comment count;
the full history lives in `comments`. Lesson 03 names that the *subset pattern*.

## How It Actually Works

The reason embedding is fast and the reason unbounded embedding is slow are the same: **the document is
the unit of I/O**. WiredTiger stores each document as one value in the collection's B-tree. To read any
field, the server locates the document via an index (usually `_id`), loads the page holding it into the
cache (decompressing it if necessary), and hands the whole BSON value to the query layer, which then
applies the projection. An embedded post with comments is one lookup and one value. A referenced design
is one lookup in `posts` plus an index range scan on `comments.postId` and twenty document fetches —
more work, even though each piece is small.

Writes work the same way. Changing one field of a document creates a new version of the **whole**
document in the cache (WiredTiger is multi-versioned: readers that started earlier still see the old
version), and the journal and checkpoint eventually write it out. MongoDB records small *diffs* in the
oplog for updates where it can, but the in-memory and on-disk cost still scales with the document's
size. A 16 MB document updated 100 times a second is 1.6 GB per second of document versions for the
cache to manage.

The cache is also why size matters for reads that "don't need" most of the document: the cache holds
pages of whole documents, so bloated documents mean fewer documents fit in RAM — the **working set** —
and more reads go to disk.

## Common mistakes

- **Modeling the data instead of the access patterns.** Start from the list of operations.
- **Embedding an unbounded list** (comments, events, followers, audit logs).
- **Referencing everything** out of relational habit, then doing three round trips for every page view.
- **Duplicating frequently changing data** without a plan for keeping copies in sync.
- **Assuming projection makes a big document cheap to read.** It doesn't reduce what the server loads.
- **Treating the 16 MB limit as the design limit.** Documents should usually be kilobytes.

## Exercise

1. Write the access-pattern table (operation, frequency, data needed) for an online course platform with
   courses, lessons, students, enrolments and quiz attempts. Decide embed or reference for each
   relationship and justify each in one sentence.
2. Re-run the post-page experiment with 200 comments per post instead of 20. Does embedding still win?
   At what number of comments per post does the referenced design catch up on your machine?
3. Implement the hybrid: `posts` embed `recentComments` (latest 5, via `$push` with `$slice`) and
   `commentCount`; full comments live in `comments`. Write the "add comment" function that updates both.
   What happens if the second write fails? (Keep the answer; Level 3 · 06 returns to it.)
4. Measure the homepage query — 50 most recent post titles — against both designs.
