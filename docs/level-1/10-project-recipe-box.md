---
description: "Level 1 project: build and test a Python data layer for a recipe box — unique slugs, ingredient search, atomic ratings, array updates and keyset paging."
---

# 10 · Project — Recipe Box Data Layer

Time to put Level 1 together. You'll build the data layer for a small recipe-box application: the
module that the web routes or CLI would call. It uses only what you've learned so far — documents with
embedded arrays, query operators, update operators, a unique index, PyMongo, and keyset pagination —
and it comes with a test suite that runs against a real server.

The point of a data layer is that the rest of the application never builds MongoDB filters itself.
Routes call `box.rate("dal-tadka", 5)`; only this module knows that a rating is two counters updated
with `$inc`.

## Requirements

1. Each recipe has a URL-friendly **slug** derived from its title, and slugs must be unique — even when
   two people add "Masala Omelette" at the same moment.
2. Find recipes containing **all** of a list of ingredients, case-insensitively.
3. Find quick recipes: a tag plus a maximum cooking time, fastest first.
4. Rate a recipe 1–5 stars and get the new average back. Concurrent ratings must never be lost.
5. Add tags (no duplicates, kept sorted) and change one ingredient's quantity in place.
6. Page through all recipes in a stable order without `skip()`.
7. Delete a recipe, reporting whether it existed.

## The document shape

```javascript
{
  _id: ObjectId("..."),
  slug: "masala-omelette",
  title: "Masala Omelette",
  ingredients: [ { name: "egg", qty: "2" }, { name: "onion", qty: null } ],
  steps: ["Beat eggs", "Fry onion", "Cook"],
  tags: ["breakfast", "quick"],
  minutes: 10,
  rating: { count: 0, total: 0 },
  createdAt: ISODate("..."),
  updatedAt: ISODate("...")
}
```

Design decisions worth noting:

- **Ingredients and steps are embedded.** They are always read with the recipe and never shared
  between recipes, and a recipe has a bounded number of them. (Level 2 · 01 formalises when to embed.)
- **Ingredient names and tags are stored lower-case.** Then equality queries are case-insensitive
  without regexes, and they can use indexes later.
- **The rating is stored as a count and a total, not as an average.** An average can't be updated
  atomically with `$inc`; a count and a total can, and the average is one division away.
- **`_id` is an ObjectId, `slug` has a unique index.** The slug is what users see and may in future be
  editable; `_id` never changes. Keyset pagination uses `_id` because it is unique and roughly ordered by
  creation time.

## The code

Save as `recipes.py`:

```python
"""Data layer for a small recipe-box app, using only Level 1 features."""
import re
from datetime import datetime, timezone

from bson import ObjectId
from pymongo import ASCENDING, MongoClient, ReturnDocument
from pymongo.errors import DuplicateKeyError


class SlugTaken(Exception):
    pass


def slugify(title: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", title.lower()).strip("-")


class RecipeBox:
    def __init__(self, client: MongoClient, db_name: str = "recipebox"):
        self.recipes = client[db_name].recipes
        self.recipes.create_index([("slug", ASCENDING)], unique=True)

    def add(self, title, ingredients, steps, tags=(), minutes=None):
        now = datetime.now(timezone.utc)
        doc = {
            "slug": slugify(title),
            "title": title,
            "ingredients": [{"name": i["name"].lower(), "qty": i.get("qty")} for i in ingredients],
            "steps": list(steps),
            "tags": sorted(set(t.lower() for t in tags)),
            "minutes": minutes,
            "rating": {"count": 0, "total": 0},
            "createdAt": now,
            "updatedAt": now,
        }
        try:
            return self.recipes.insert_one(doc).inserted_id
        except DuplicateKeyError:
            raise SlugTaken(doc["slug"]) from None

    def get(self, slug):
        return self.recipes.find_one({"slug": slug})

    def with_all_ingredients(self, names):
        names = [n.lower() for n in names]
        return list(self.recipes.find({"ingredients.name": {"$all": names}},
                                      {"_id": 0, "slug": 1}).sort("slug", ASCENDING))

    def quick(self, tag, max_minutes):
        return list(self.recipes.find({"tags": tag, "minutes": {"$lte": max_minutes}},
                                      {"_id": 0, "slug": 1, "minutes": 1}).sort([("minutes", 1), ("slug", 1)]))

    def rate(self, slug, stars):
        if not 1 <= stars <= 5:
            raise ValueError("stars must be 1-5")
        doc = self.recipes.find_one_and_update(
            {"slug": slug},
            {"$inc": {"rating.count": 1, "rating.total": stars}, "$currentDate": {"updatedAt": True}},
            projection={"_id": 0, "rating": 1},
            return_document=ReturnDocument.AFTER,
        )
        if doc is None:
            raise KeyError(slug)
        r = doc["rating"]
        return round(r["total"] / r["count"], 2)

    def add_tag(self, slug, tag):
        tag = tag.lower()
        # $addToSet would append at the end; this keeps the array sorted and duplicate-free.
        res = self.recipes.update_one(
            {"slug": slug, "tags": {"$ne": tag}},
            {"$push": {"tags": {"$each": [tag], "$sort": 1}}},
        )
        return res.modified_count == 1 or self.recipes.count_documents({"slug": slug}, limit=1) == 1

    def set_qty(self, slug, ingredient, qty):
        res = self.recipes.update_one({"slug": slug, "ingredients.name": ingredient.lower()},
                                      {"$set": {"ingredients.$.qty": qty}})
        return res.modified_count == 1

    def page(self, after_id=None, size=2):
        flt = {"_id": {"$gt": ObjectId(after_id)}} if after_id else {}
        docs = list(self.recipes.find(flt, {"slug": 1}).sort("_id", ASCENDING).limit(size))
        next_cursor = str(docs[-1]["_id"]) if len(docs) == size else None
        return [d["slug"] for d in docs], next_cursor

    def delete(self, slug):
        return self.recipes.delete_one({"slug": slug}).deleted_count == 1
```

How each requirement maps to Level 1 features:

| Requirement | Method | Feature |
|-------------|--------|---------|
| Unique slugs | `__init__`, `add` | unique index + catching `DuplicateKeyError` (code 11000) — lesson 08 |
| All ingredients | `with_all_ingredients` | dot path into an array of documents + `$all` — lesson 05 |
| Quick recipes | `quick` | array "contains" + range + compound sort with tie-breaker — lessons 04, 05 |
| Ratings | `rate` | `find_one_and_update` with `$inc` and `ReturnDocument.AFTER` — lesson 06 |
| Tags | `add_tag` | conditional `$push` with `$each` + `$sort` — lesson 06 |
| Quantities | `set_qty` | positional `$` — lesson 06 |
| Paging | `page` | keyset pagination on `_id` — lesson 04 |
| Delete | `delete` | `deleted_count` — lesson 07 |

`create_index` is idempotent: calling it when the index already exists with the same definition does
nothing, so it is safe in the constructor for a project this size. Larger applications create indexes in
a migration or deployment step instead (Level 4 · 09).

## The tests

Save as `test_recipes.py`. The fixture creates a throwaway database for each test and drops it
afterwards, so tests can't affect each other or your real data.

```python
import pytest
from pymongo import MongoClient
from recipes import RecipeBox, SlugTaken

URI = "mongodb://127.0.0.1:27017/?replicaSet=rs0"


@pytest.fixture
def box():
    client = MongoClient(URI, tz_aware=True)
    client.drop_database("recipebox_test")
    b = RecipeBox(client, "recipebox_test")
    b.add("Masala Omelette", [{"name": "Egg", "qty": "2"}, {"name": "Onion"}, {"name": "Chilli"}],
          ["Beat eggs", "Fry onion", "Cook"], tags=["Breakfast", "quick"], minutes=10)
    b.add("Egg Fried Rice", [{"name": "egg", "qty": "2"}, {"name": "rice", "qty": "2 cups"}, {"name": "onion"}],
          ["Scramble", "Add rice"], tags=["dinner", "quick"], minutes=20)
    b.add("Dal Tadka", [{"name": "lentils", "qty": "1 cup"}, {"name": "onion"}, {"name": "ghee"}],
          ["Boil", "Temper"], tags=["dinner"], minutes=40)
    yield b
    client.drop_database("recipebox_test")
    client.close()


def test_slug_is_unique(box):
    with pytest.raises(SlugTaken):
        box.add("masala omelette!", [{"name": "egg"}], ["x"])


def test_ingredient_search_needs_all(box):
    assert box.with_all_ingredients(["EGG", "onion"]) == [{"slug": "egg-fried-rice"}, {"slug": "masala-omelette"}]
    assert box.with_all_ingredients(["egg", "ghee"]) == []


def test_quick_recipes_sorted_by_time(box):
    assert [r["slug"] for r in box.quick("quick", 15)] == ["masala-omelette"]
    assert [r["slug"] for r in box.quick("dinner", 60)] == ["egg-fried-rice", "dal-tadka"]


def test_rating_average(box):
    assert box.rate("dal-tadka", 5) == 5.0
    assert box.rate("dal-tadka", 4) == 4.5
    with pytest.raises(KeyError):
        box.rate("nope", 3)


def test_tags_and_quantities(box):
    assert box.add_tag("dal-tadka", "Comfort")
    assert box.add_tag("dal-tadka", "comfort")
    assert box.get("dal-tadka")["tags"] == ["comfort", "dinner"]
    assert box.set_qty("dal-tadka", "Ghee", "1 tbsp")
    assert not box.set_qty("dal-tadka", "saffron", "a pinch")
    ghee = [i for i in box.get("dal-tadka")["ingredients"] if i["name"] == "ghee"][0]
    assert ghee["qty"] == "1 tbsp"


def test_keyset_pagination(box):
    first, cur = box.page()
    second, cur2 = box.page(cur)
    assert first == ["masala-omelette", "egg-fried-rice"]
    assert second == ["dal-tadka"] and cur2 is None


def test_delete(box):
    assert box.delete("dal-tadka")
    assert not box.delete("dal-tadka")
```

## The first run failed

The first version of `add_tag` used `$addToSet`:

```python
def add_tag(self, slug, tag):
    res = self.recipes.update_one({"slug": slug}, {"$addToSet": {"tags": tag.lower()}})
    return res.matched_count == 1
```

```text
FAILED test_recipes.py::test_tags_and_quantities - AssertionError: assert ['d...
1 failed, 6 passed in 1.36s
```

```text
E       AssertionError: assert ['dinner', 'comfort'] == ['comfort', 'dinner']
```

`$addToSet` prevents duplicates but always **appends** — it doesn't keep `tags` sorted, which `add`
promised. The fix in the code above is a conditional push: the filter `{"tags": {"$ne": tag}}`
only matches when the tag is absent (so there are no duplicates), and `$push` with `$each` and
`$sort: 1` inserts it in order. Because filter and update are applied atomically to one document, two
concurrent `add_tag` calls with the same tag still can't both add it.

After the fix:

```text
test_recipes.py::test_slug_is_unique PASSED                              [ 14%]
test_recipes.py::test_ingredient_search_needs_all PASSED                 [ 28%]
test_recipes.py::test_quick_recipes_sorted_by_time PASSED                [ 42%]
test_recipes.py::test_rating_average PASSED                              [ 57%]
test_recipes.py::test_tags_and_quantities PASSED                         [ 71%]
test_recipes.py::test_keyset_pagination PASSED                           [ 85%]
test_recipes.py::test_delete PASSED                                      [100%]
```

## Checking ratings under concurrency

Unit tests run one thing at a time. Requirement 4 is about concurrency, so check it directly — 400
ratings from 8 threads, cycling through 1–5 stars (80 of each, total 1,200):

```python
from concurrent.futures import ThreadPoolExecutor
from pymongo import MongoClient
from recipes import RecipeBox

client = MongoClient("mongodb://127.0.0.1:27017/?replicaSet=rs0")
client.drop_database("recipebox_load")
box = RecipeBox(client, "recipebox_load")
box.add("Plain Toast", [{"name": "bread"}], ["Toast it"], minutes=3)

with ThreadPoolExecutor(8) as pool:
    list(pool.map(lambda i: box.rate("plain-toast", i % 5 + 1), range(400)))

print(box.get("plain-toast")["rating"])
client.drop_database("recipebox_load")
```

```text
{'count': 400, 'total': 1200}
```

Every rating counted. Compare that with lesson 06, where read-modify-write lost 80% of increments under
the same kind of load.

## How It Actually Works

Three different mechanisms guarantee correctness here, and it's worth being precise about which one
covers which requirement.

**The unique index** guarantees unique slugs. Checking "does this slug exist?" and then inserting would
race: two requests could both see "no" and both insert. With a unique index, the second insert fails
inside the storage engine at the moment it tries to add the duplicate key to the index, regardless of
timing — the application just translates `DuplicateKeyError` into `SlugTaken`.

**Single-document atomicity** guarantees ratings and tags. `find_one_and_update` finds the document,
applies both `$inc`s and the `$currentDate`, and returns the new version as one atomic operation. No
other write to that document can interleave, so `count` and `total` always move together and the
average returned is the average *including* this rating.

**A sort key with a unique field** guarantees stable paging. Keyset pagination asks for "the next two
`_id`s greater than the last one I saw". Because `_id` is unique and indexed, the server jumps straight
to that position in the `_id` index and reads two entries — page 1,000 costs the same as page 1, and
inserts or deletes between page requests can't cause a recipe to be shown twice or skipped (as they can
with `skip()`).

What the design does **not** guarantee yet: search by ingredient and by tag are collection scans,
because the only indexes are `_id` and `slug`. With three recipes that doesn't matter; with three million
it would. Level 2 adds the indexes and shows how to prove they're used.

## Common mistakes

- **Checking for duplicates in application code** instead of with a unique index.
- **Storing an average** (or any derived value) that can't be maintained atomically.
- **Leaking MongoDB filters into route handlers**, so a schema change touches the whole codebase.
- **Tests that share one database** and pass or fail depending on order. Use a fresh database per test.
- **Assuming `$addToSet` preserves order.**

## Exercise

Extend the recipe box:

1. Add `rename(slug, new_title)` that changes the title **and** the slug, raising `SlugTaken` when the
   new slug is in use. Write tests for both outcomes.
2. Add `favourite(slug, user_id)` and `unfavourite(...)` storing user ids in a `favouritedBy` array plus a
   `favouriteCount` kept in sync in the **same** update. Make sure favouriting twice doesn't double-count
   (hint: filter on `favouritedBy: {$ne: user_id}`).
3. Add `recently_rated(n)` returning the `n` recipes with the most recent `updatedAt`, newest first, using a
   unique tie-breaker.
4. Dump the test database with `mongodump` before the fixture drops it (add a flag), and restore it under a
   new name to inspect what the tests wrote.
5. Write a concurrency check for `add_tag`: 20 threads adding the same new tag to one recipe. Assert it
   appears exactly once.
