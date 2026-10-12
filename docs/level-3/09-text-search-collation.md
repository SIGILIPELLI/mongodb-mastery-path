---
description: "MongoDB text indexes (stemming, weights, phrases, scores, limits) and collations for case- and accent-insensitive queries and sorting."
---

# 09 · Text Search & Collation

Two features this lesson covers deal with the fact that human text doesn't compare like bytes. **Text
indexes** let you search for words — "running" should find "runs" — and rank results by relevance.
**Collations** make equality and sorting follow language rules — "zoe" equals "Zoe", "émile" sorts next to
"Emile", "item9" comes before "item10". Both are built into the server; both have limits you should know
before you promise a product manager "search".

## Text indexes

Five articles, one text index over title and body, with the title weighted five times higher:

```javascript
db.articles.insertMany([
  { _id: 1, title: "Running a database in production", body: "Backups, monitoring and running upgrades without downtime." },
  { _id: 2, title: "Indexes explained", body: "How a B-tree index runs range scans. The runner examines keys." },
  { _id: 3, title: "Cooking with cast iron", body: "Seasoning a pan; why food runs off a well-seasoned surface." },
  { _id: 4, title: "Database backups", body: "Logical and physical backup strategies for a production database." },
  { _id: 5, title: "Running shoes reviewed", body: "Road and trail running shoes tested over 500 km." }]);
db.articles.createIndex({ title: "text", body: "text" },
  { weights: { title: 5, body: 1 }, name: "articles_text", default_language: "english" });
```

Searching, with each result's relevance score:

```javascript
const S = q => db.articles.find({ $text: { $search: q } }, { score: { $meta: "textScore" }, title: 1 })
  .sort({ score: { $meta: "textScore" } }).toArray().map(d => d._id + ":" + d.score.toFixed(2)).join("  ");
```

```text
run         -> 1:3.92  5:3.90  3:0.57  2:0.56
database    -> 4:4.33  1:3.33
```

"run" matched "Running", "running" and "runs" — the index stores **stems**, so word forms match. Articles 1 and 5
score highest because "Running" is in their (weighted) titles; 2 and 3 only have "runs" in the body. Note what
*didn't* match: "runner" in article 2's body is a different stem, so article 2 matched only through "runs".

```text
"production database" -> 4:4.92
running -shoes        -> 1:3.92  3:0.57  2:0.56
datab (partial)       ->
the (stop word)       ->
```

- A **quoted phrase** must appear exactly (article 1 has both words, but not as the phrase "production database").
- A **minus** excludes a term.
- **No partial words**: "datab" matches nothing. Text indexes are not autocomplete.
- **Stop words** ("the", "a", "and") aren't indexed, so searching for them returns nothing.

Space-separated terms are OR'ed (any term matches), with documents matching more terms scoring higher.

The plan is an index scan per term, OR'ed together, then fetched and matched:

```text
text explain: "stage":"TEXT_MATCH","stage":"FETCH","stage":"OR","stage":"IXSCAN"
```

### Limits worth knowing

- **One text index per collection.** Creating a second fails — every text index has the same internal key
  pattern, so the server reports it as a conflict:

  ```text
  85 IndexOptionsConflict An equivalent index already exists with a different name and options. Requested index: { v: 2, key: { _fts: "text", _fts ...
  ```

  Include every field you'll want to search in that one index (or use `{"$**": "text"}` for all string fields).
- **No fuzzy matching, typo tolerance, synonyms or autocomplete.**
- **Scoring is simple** term-frequency weighting, not a modern relevance model.
- **Stemming is per language** (`default_language`, or a `language` field per document); `"none"` disables stemming
  and stop words.

For a search box that users will judge against web search engines, MongoDB offers a separate full-text engine
built on Apache Lucene: **Atlas Search** on MongoDB Atlas, and in recent releases a self-managed search process
(`mongot`) that brings the `$search` aggregation stage to Community and Enterprise deployments. It adds fuzzy matching,
autocomplete, synonyms, facets and better relevance. It wasn't part of this course's lab setup, so it isn't
demonstrated here; check MongoDB's current documentation for availability on your deployment type. Text indexes
remain a good fit for simple keyword search inside an application: admin tools, internal search, tagging.

## Collation: language-aware comparison

By default, strings compare by their bytes (UTF-8 binary order). That's fast and predictable, and wrong for people:

```javascript
db.people.insertMany([{ n: "émile" }, { n: "Zoe" }, { n: "adam" }, { n: "Émile" }, { n: "zoe" }, { n: "Adam" },
                      { n: "item10" }, { n: "item9" }]);
db.people.find().sort({ n: 1 })                                  // binary
db.people.find().sort({ n: 1 }).collation({ locale: "en" })      // English rules
```

```text
binary sort:      Adam,Zoe,adam,item10,item9,zoe,Émile,émile
en sort:          adam,Adam,émile,Émile,item10,item9,zoe,Zoe
```

Binary order puts all capitals before all lower-case letters and accented letters after `z`. The English collation
sorts the way a person expects. Add `numericOrdering` and digit sequences compare as numbers:

```text
numericOrdering:  item9,item10
```

### Strength: what counts as "equal"

```javascript
db.people.countDocuments({ n: "zoe" })                                                    // 1
db.people.countDocuments({ n: "zoe" }, { collation: { locale: "en", strength: 2 } })      // 2
db.people.countDocuments({ n: "emile" }, { collation: { locale: "en", strength: 1 } })    // 2
```

`strength: 2` ignores case (zoe = Zoe); `strength: 1` also ignores accents (emile = émile = Émile). The default is 3,
which distinguishes both.

### Indexes must have the same collation

A case-insensitive lookup is only fast if an index with the **same collation** exists and the query asks for it:

```javascript
db.people.createIndex({ n: 1 }, { collation: { locale: "en", strength: 2 }, name: "n_ci" });
```

```text
query with matching collation: "stage":"FETCH","stage":"IXSCAN"
query without collation:       "stage":"COLLSCAN"
regex /^zoe$/i plan:           "stage":"COLLSCAN"
```

The query without a collation uses binary comparison, which the `n_ci` index can't answer. And the classic workaround,
a case-insensitive regex, can't use an index efficiently at all.

The simplest way to make collation automatic is to set it on the **collection**; every index and query then defaults to
it. Case-insensitive unique emails:

```javascript
db.createCollection("users", { collation: { locale: "en", strength: 2 } });
db.users.createIndex({ email: 1 }, { unique: true });
db.users.insertOne({ email: "Ana@Example.com" });
db.users.insertOne({ email: "ana@example.com" })              // 11000 duplicate key
db.users.countDocuments({ email: "ANA@EXAMPLE.COM" })          // 1
```

The alternative — storing a normalised `emailLower` field alongside the original — works everywhere and is easier to
reason about across languages and tools; a collection collation is neater when the whole collection is about
human-facing text.

## How It Actually Works

A text index is a B-tree whose keys are `(term, weight)` pairs: when a document is indexed, each indexed string field is
tokenised, lower-cased, stop words are dropped, each token is reduced to its stem with a Snowball stemmer for the
document's language, and one key per distinct stem is written with a weight derived from the term's frequency and the
field's weight. A `$text` query applies the same processing to the search string, does one index scan per stem (the `OR`
of `IXSCAN`s), and accumulates each document's score; phrases and negations are checked afterwards against the fetched
document (`TEXT_MATCH`). Because the index stores whole stems, prefixes and typos can't be matched.

A collation is implemented with ICU, the Unicode library: each string is converted into a **sort key** — a byte string
whose binary order is the collation order for that locale and strength. An index with a collation stores sort keys
instead of raw strings, which is why equality and range scans on it are fast, and why a query must use the same collation
to use the index: its comparison values have to be converted the same way. Sort keys at strength 2 don't encode case,
so "Zoe" and "zoe" produce the same key — that's what makes the unique index above case-insensitive.

## Common mistakes

- **Promising "search" with a text index** and then being asked for autocomplete and typo tolerance.
- **Trying to add a second text index** instead of including all fields in one.
- **Searching for partial words** (`$text` matches whole stems only).
- **Case-insensitive regex for lookups** on large collections — use a collation index or a normalised field.
- **Index and query collations that differ**, silently falling back to a collection scan.
- **Sorting user-visible names in binary order.**

## Exercise

1. Build a text index over a `recipes` collection (title weighted 10, ingredients 3, steps 1). Search for "tomato
   -garlic", for a phrase, and print the scores. Explain each score ordering.
2. Insert some recipes with `language: "spanish"` and search them with `$language: "spanish"`. What changes in the matched
   stems?
3. Create a `products` collection with an English collation at strength 1 and a unique index on `name`. Which of "Café",
   "cafe", "CAFE" can coexist?
4. Sort 1,000 strings like `"file1"` … `"file1000"` with and without `numericOrdering` and print the first 12 of each.
5. Measure a case-insensitive lookup on 100,000 users three ways: `/^…$/i` regex, a collation index, and a normalised
   `emailLower` field with an ordinary index. Compare keys and documents examined.
