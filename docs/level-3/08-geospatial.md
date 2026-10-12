---
description: "Location queries in MongoDB: GeoJSON, 2dsphere indexes, $near and $geoNear distances, $geoWithin zones, $geoIntersects, and the longitude-first trap."
---

# 08 · Geospatial Queries

"Stores near me", "drivers within 2 km", "which delivery zone is this address in", "restaurants inside this
map view" — location queries show up in almost every consumer app. MongoDB stores locations as **GeoJSON**,
indexes them with a **2dsphere** index that understands the Earth is round, and answers proximity and
containment questions with a handful of operators.

## Storing locations

A location is a GeoJSON object with a `type` and `coordinates`:

```javascript
{ name: "Store 1", type: "pharmacy",
  loc: { type: "Point", coordinates: [73.7035, 18.4336] } }   // [longitude, latitude]
```

**Longitude first, then latitude** — x before y. Maps, GPS apps and most people say "lat, long", so this is the
most common geospatial bug in existence. Other GeoJSON types — `LineString`, `Polygon`, `MultiPolygon` — store
routes and areas.

The lab dataset: 2,000 stores with seeded random positions across a roughly 30 × 24 km box around central Pune
(longitude 73.70–74.00, latitude 18.42–18.64). The positions are synthetic, not real businesses.

```javascript
db.stores.insertMany(docs);
db.stores.createIndex({ loc: "2dsphere" });
const me = { type: "Point", coordinates: [73.8567, 18.5204] };   // a point in central Pune
```

## Nearest first: $near

```javascript
db.stores.find({ type: "pharmacy", loc: { $near: { $geometry: me, $maxDistance: 2000 } } }, { name: 1, _id: 0 }).limit(3)
```

```text
{"name":"Store 1738"}
{"name":"Store 661"}
{"name":"Store 88"}
```

`$near` returns documents **sorted by distance**, nearest first, optionally limited by `$maxDistance` (and
`$minDistance`) in **metres**. It requires a geospatial index. Its explain:

```text
near: stages "stage":"FETCH","stage":"IXSCAN" keys 123 docs 59 n 42
```

Within 2 km there were 42 stores; the index search examined 123 keys and 59 documents to find them — it searches
outward in rings of index cells and fetches only candidates in those cells.

One subtlety: adding `.sort()` to a `$near` query **replaces** the distance ordering. A `$near` query sorted by
name returned "Store 1" — alphabetical order, not proximity. If you need "nearest, then by name", sort in the
application or use `$geoNear`.

## Distances: $geoNear

To show "407 m away", use the `$geoNear` aggregation stage, which adds the distance to each document. It must be
the first stage of the pipeline:

```javascript
db.stores.aggregate([
  { $geoNear: { near: me, distanceField: "dist", maxDistance: 1500, query: { type: "cafe" }, spherical: true } },
  { $limit: 3 },
  { $project: { _id: 0, name: 1, metres: { $round: ["$dist", 0] } } }
])
```

```text
{"name":"Store 173","metres":407}
{"name":"Store 1154","metres":621}
{"name":"Store 1646","metres":640}
```

Put non-spatial filters in `$geoNear`'s `query` option rather than a later `$match`; that way they're applied during
the search and `$limit` gets the nearest *cafés*, not the nearest stores filtered down afterwards. Distances are
computed on a sphere — accurate to well under 1% for app-scale distances, which is plenty for "how far is the shop",
but not a surveying tool.

## Inside an area: $geoWithin

Within a circle (radius in **radians**: distance divided by the Earth's radius, 6,378.1 km):

```javascript
db.stores.countDocuments({ loc: { $geoWithin: { $centerSphere: [me.coordinates, 1 / 6378.1] } } })   // 10 within 1 km
```

Within a polygon — say a delivery zone drawn on a map:

```javascript
const zone = { type: "Polygon", coordinates: [[
  [73.80, 18.50], [73.88, 18.50], [73.88, 18.56], [73.80, 18.56], [73.80, 18.50] ]] };
db.stores.countDocuments({ loc: { $geoWithin: { $geometry: zone } } })   // 156
```

A polygon's ring must be **closed** (first point repeated at the end), and its edges are geodesic — great-circle
arcs, not straight lines on a flat map. For small zones the difference is negligible; for a polygon spanning hundreds
of kilometres it isn't. `$geoWithin` doesn't sort, and it works without an index (slowly) as well as with one.

## Which area contains this point: $geoIntersects

Flip the question: store the zones, query with a point.

```javascript
db.zones.insertOne({ _id: "central", area: zone });
db.zones.insertOne({ _id: "east", area: { type: "Polygon", coordinates: [[
  [73.88, 18.50], [73.95, 18.50], [73.95, 18.56], [73.88, 18.56], [73.88, 18.50] ]] } });
db.zones.createIndex({ area: "2dsphere" });
db.zones.find({ area: { $geoIntersects: { $geometry: me } } }, { _id: 1 })
```

```text
[{"_id":"central"}]
```

That's how "is this address deliverable, and by which hub?" is answered in one indexed query.

## The longitude-first trap, measured

A store inserted with its coordinates swapped — `[18.5204, 73.8567]` — is still a valid point: longitude 18.5°E,
latitude 73.9°N, somewhere in the Arctic north of Norway. Nothing fails:

```text
swapped lat/lng distance km: 7005
```

The "local" store is 7,005 km away and simply never appears in nearby results. Only swaps that produce an
impossible latitude are rejected:

```javascript
db.stores.insertOne({ name: "bad", loc: { type: "Point", coordinates: [73.8, 98.5] } })
```

```text
510 Could not extract geo keys at path 'loc' for document with _id ObjectId('...'): Longitude/latitude is out of bounds, ...
```

Validate coordinates at the edge of your system (a `$jsonSchema` that restricts longitude to [-180, 180] and latitude
to [-90, 90] catches only the impossible ones; for a regional app, restrict to your region's bounding box) and name
variables `lng`/`lat` explicitly in code.

## How It Actually Works

A 2dsphere index maps the sphere onto a hierarchy of cells using Google's **S2** geometry library: the Earth is
projected onto the six faces of a cube, and each face is recursively divided into four, giving cells at 30 levels of
precision, each with a 64-bit id. Nearby cells have nearby ids along a space-filling curve. A point is indexed by the
id of its cell at a fine level; a polygon by a *covering* — a set of cells, at mixed levels, that together cover the
shape.

That turns geometry into B-tree range scans. `$geoWithin` with a polygon computes a covering of the query polygon and
scans the index ranges for those cells; the fetched candidates are then checked exactly against the geometry (cells
near the edge can contain points just outside it — the extra documents examined). `$near` starts with the cells
around the query point and expands in rings, emitting documents in distance order from each ring once it knows nothing
closer can appear in an unexplored ring. That's why `$near` can stop early after a `limit`, and why it examined 123
keys to return 42 stores.

## Common mistakes

- **Latitude first.** GeoJSON is `[longitude, latitude]`.
- **Radians vs metres**: `$centerSphere` takes radians; `$near`/`$geoNear` with GeoJSON take metres.
- **Unclosed polygon rings**, which are rejected.
- **Filtering after `$geoNear`** instead of using its `query` option, so `$limit` gives too few results.
- **Adding `sort()` to `$near`** and losing distance order.
- **Using the legacy `2d` index and coordinate pairs** for real-world locations — it treats the Earth as flat. It's
  only for genuinely flat coordinate systems (a game map, a floor plan).

## Exercise

1. Add a `rating` to each store. Find the 5 nearest stores with a rating of at least 4, with their distances in
   metres, within 3 km of a point of your choice.
2. Draw a polygon around a neighbourhood using a map tool that exports GeoJSON, and count the stores inside it. Make
   sure the coordinate order matches.
3. Create three overlapping delivery zones with a `priority` field. For a given address, return the highest-priority zone
   that contains it.
4. Write a `$jsonSchema` validator for `stores.loc` that only accepts points inside the bounding box of your city.
5. Simulate "drivers": 500 documents whose locations update every second (a script with `bulkWrite`), and a query for
   the 3 nearest available drivers. Measure query time while the updates run.
