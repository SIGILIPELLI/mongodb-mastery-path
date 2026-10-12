---
description: "Move data with mongoimport, mongoexport, mongodump and mongorestore: typed CSV imports, Extended JSON, archives and namespace remapping."
---

# 09 · mongoimport, mongoexport, mongodump & mongorestore

Sooner or later you need to load a spreadsheet, hand data to an analyst, copy a database to your
laptop, or take a quick backup before a risky migration. The MongoDB Database Tools cover all four.
They come in two pairs that are easy to confuse:

| Pair | Format | Preserves BSON types? | Use for |
|------|--------|------------------------|---------|
| `mongoimport` / `mongoexport` | JSON, CSV, TSV (text) | Only with Extended JSON; CSV loses them unless you declare types | Exchanging data with other systems and people |
| `mongodump` / `mongorestore` | BSON (binary) + metadata | Yes, exactly, plus indexes and collection options | Copies and logical backups between MongoDB deployments |

The tools are a separate download from the server and are versioned on their own (100.19.1 here).
Every command below ran against the lab replica set with `U="mongodb://127.0.0.1:27017/?replicaSet=rs0"`.

## Importing a CSV: what goes wrong by default

```text
sku,name,price,qty,added
P-100,Steel kettle,29.99,3,2026-09-01
P-101,Toaster,45.00,0,2026-09-14
P-102,"Mug, large",6.50,40,2026-10-02
P-103,007 Coffee filter,3.25,120,2026-10-05
```

```bash
mongoimport --uri "$U" --db io --collection naive --type csv --headerline --drop products.csv
```

```text
2026-10-11T23:03:00.644+0530	dropping: `io.naive`
2026-10-11T23:03:00.676+0530	4 document(s) imported successfully. 0 document(s) failed to import.
```

It worked — but look at the types:

```text
[ 'P-100', 'Steel kettle', 29.99, 'number', '2026-09-01' ]
[ 'P-101', 'Toaster', 45, 'number', '2026-09-14' ]
...
```

`mongoimport` guesses: anything that looks numeric becomes a number (a **double** for prices — lesson
03 explained why that is bad for money), and dates stay **strings**. The guessing also destroys data
that merely looks numeric:

```text
name,zip
Holtsville,00501
```

```text
{ name: 'Holtsville', zip: 501 }
```

The leading zeros are gone. Postal codes, phone numbers, account numbers and product codes are
identifiers, not numbers.

## Importing with declared types

Put the type of each column in the header and pass `--columnsHaveTypes`:

```text
sku.string(),name.string(),price.decimal(),qty.int32(),added.date(2006-01-02)
P-100,Steel kettle,29.99,3,2026-09-01
...
```

```bash
mongoimport --uri "$U" --db io --collection products --type csv --headerline --columnsHaveTypes --drop products2.csv
```

```text
{
  sku: 'P-102',
  name: 'Mug, large',
  price: Decimal128('6.50'),
  qty: 40,
  added: ISODate('2026-10-02T00:00:00.000Z')
}
```

Exact decimals, real dates, and the quoted comma in `"Mug, large"` handled correctly. The date format
uses Go's reference-time layout: `2006-01-02` means "year-month-day" (other layouts:
`date_go(...)`, `date_ms(...)` and `date_oracle(...)`; check `mongoimport --help` for your version).

For JSON input, `mongoimport` expects one document per line by default, or a single array with
`--jsonArray`. Extended JSON (`{"$date": ...}`, `{"$numberDecimal": ...}`) in the input is converted to
the proper BSON types.

Useful flags: `--mode upsert --upsertFields sku` to update existing documents instead of failing on
duplicates, `--numInsertionWorkers` for parallelism on big files, and `--stopOnError` if you'd rather
fail fast than skip bad lines.

## Exporting

```bash
mongoexport --uri "$U" -d io -c products --sort '{sku:1}' --limit 2 --quiet
```

```text
{"_id":{"$oid":"6acbc84d1c4469c7f143e675"},"sku":"P-100","name":"Steel kettle","price":{"$numberDecimal":"29.99"},"qty":3,"added":{"$date":"2026-09-01T00:00:00Z"}}
{"_id":{"$oid":"6acbc84d1c4469c7f143e677"},"sku":"P-101","name":"Toaster","price":{"$numberDecimal":"45.00"},"qty":0,"added":{"$date":"2026-09-14T00:00:00Z"}}
```

This is **relaxed Extended JSON**: plain numbers where nothing is lost, `$oid`/`$numberDecimal`/`$date`
wrappers where a type would otherwise be lost. Feed it back into `mongoimport` and the types survive.
`--jsonFormat canonical` wraps every value, including ints and dates as epoch milliseconds:

```text
{"_id":{"$oid":"6acbc84d1c4469c7f143e675"},"sku":"P-100","name":"Steel kettle","price":{"$numberDecimal":"29.99"},"qty":{"$numberInt":"3"},"added":{"$date":{"$numberLong":"1788220800000"}}}
```

CSV export needs an explicit field list, because documents have no fixed columns:

```bash
mongoexport --uri "$U" -d io -c products --type csv --fields sku,price,added --sort '{sku:1}' --quiet
```

```text
sku,price,added
P-100,29.99,2026-09-01T00:00:00.000Z
P-101,45.00,2026-09-14T00:00:00.000Z
P-102,6.50,2026-10-02T00:00:00.000Z
P-103,3.25,2026-10-05T00:00:00.000Z
```

Arrays and embedded documents end up as JSON text inside a CSV cell — usually a sign you should
reshape with an aggregation (Level 3) and export the result instead.

## mongodump: a logical copy

After adding a unique index on `sku`:

```bash
mongodump --uri "$U" --db io --out dump
find dump | sort
```

```text
dump
dump/io
dump/io/naive.bson
dump/io/naive.metadata.json
dump/io/prelude.json
dump/io/products.bson
dump/io/products.metadata.json
dump/io/zips.bson
dump/io/zips.metadata.json
```

For each collection: a `.bson` file with the documents in raw BSON, and a `.metadata.json` with its
indexes and options. `prelude.json` records what produced the dump:

```text
{"ServerVersion":"9.0.2","ToolVersion":"100.19.1"}
```

The metadata contains the index definitions, not the index data — `sku_1` will be **rebuilt** on
restore:

```text
{"indexes":[{"v":{"$numberInt":"2"},"key":{"_id":{"$numberInt":"1"}},"name":"_id_",...},{"v":{"$numberInt":"2"},"key":{"sku":{"$numberInt":"1"}},"name":"sku_1","unique":true,...}],...,"collectionName":"products","type":"collection"}
```

`bsondump` turns a `.bson` file into readable JSON when you need to peek inside:

```bash
bsondump --quiet dump/io/products.bson | head -1
```

```text
{"_id":{"$oid":"6acbc84d1c4469c7f143e675"},"sku":"P-100","name":"Steel kettle","price":{"$numberDecimal":"29.99"},"qty":{"$numberInt":"3"},"added":{"$date":{"$numberLong":"1788220800000"}}}
```

For a single file that is easy to move around, use an archive, optionally compressed:

```bash
mongodump --uri "$U" --db io --archive=io.archive.gz --gzip --quiet
```

The whole database became an 845-byte file. `--archive` without a file name writes to standard output,
so you can pipe a dump straight into `mongorestore` on another server.

## mongorestore

Restore into a **different** database name with namespace remapping — the safe way to inspect a backup
without touching the original:

```bash
mongorestore --uri "$U" --nsFrom 'io.*' --nsTo 'io_copy.*' dump
```

```text
2026-10-11T23:03:14.865+0530	no indexes to restore for collection `io_copy.zips`
2026-10-11T23:03:14.989+0530	9 document(s) restored successfully. 0 document(s) failed to restore.
```

```text
naive,products,zips 4 ["_id_","sku_1"]
```

All three collections, the four products, and the unique index came back.

Now restore over the **existing** `io.products`:

```bash
mongorestore --uri "$U" --nsInclude 'io.products' dump
```

```text
continuing through error: E11000 duplicate key error collection: io.products index: _id_ dup key: { _id: ObjectId('6acbc84d1c4469c7f143e675') }
...
0 document(s) restored successfully. 4 document(s) failed to restore.
```

`mongorestore` only **inserts**; it never overwrites. Documents whose `_id` already exists are skipped
with an error, and documents that exist only in the target are left alone. To replace the collection
with the backup's contents, add `--drop`:

```bash
mongorestore --uri "$U" --nsInclude 'io.products' --drop dump
```

```text
4 document(s) restored successfully. 0 document(s) failed to restore.
```

`--drop` drops each collection *being restored* first — not the whole database, and not collections
absent from the dump.

## How It Actually Works

`mongoexport` and `mongodump` are ordinary clients. Each runs a `find` over every collection (in
parallel across collections, controlled by `--numParallelCollections`) and writes what it receives:
`mongoexport` converts each document to Extended JSON or CSV text; `mongodump` writes the raw BSON
bytes exactly as received, which is why a dump preserves every type for free and is faster to produce.
Neither takes a lock or a consistent snapshot of the whole database: collection A is read at a slightly
different time from collection B, and documents can change while a large collection is being read. On a
replica set, `mongodump --oplog` (whole-deployment dumps only) also captures the oplog entries written
during the dump so `mongorestore --oplogReplay` can bring everything to a single point in time — Level 4
· 06 covers that and the alternatives.

`mongorestore` reads the metadata first, creates each collection with its options, inserts documents in
batches (with multiple insertion workers), and **then** builds the indexes from the definitions.
Building indexes once at the end is much faster than maintaining them during millions of inserts, but
it means a restore of a large, heavily indexed collection spends most of its time in index builds, and
a unique index whose constraint is violated by the data fails at the end, not the start.

Because dumps contain only the documents and index *definitions*, they are smaller than the data
directory and portable across server versions and storage settings. The trade-off is time: dumping and
restoring hundreds of gigabytes is slow, which is why large production systems rely on physical backups
instead (Level 4 · 06).

## Common mistakes

- **Importing CSV without declared types**, turning money into doubles, dates into strings and postal
  codes into numbers.
- **Using mongoexport as a backup.** It is for data exchange; use mongodump (or better, Level 4's
  methods), which keeps types, indexes and options.
- **Expecting mongorestore to overwrite.** It inserts only; use `--drop` deliberately.
- **Restoring over a live database to "check" a backup.** Restore to a different namespace with
  `--nsFrom/--nsTo`, or to a different server.
- **Treating a mongodump of a busy database as a point-in-time snapshot.** It isn't, unless you used
  `--oplog` on a replica set.
- **Never testing restores.** A backup you haven't restored is a hope, not a backup.

## Exercise

1. Download any public CSV dataset with at least one date column and one identifier column with leading
   zeros (or create one). Import it naively, inspect the types with `$type`, then re-import with a typed
   header and compare.
2. Export a collection with `--jsonFormat canonical`, delete the collection, re-import the file and verify
   with `$type` that every field has its original type.
3. Dump the `library` database from lesson 04 as a gzipped archive, restore it as `library_restore`
   using `--nsFrom/--nsTo`, and compare document counts and index names between the two.
4. Pipe a dump directly into a restore on a second local server: start another `mongod` on port 27100 and
   run `mongodump --archive ... | mongorestore --archive --uri mongodb://127.0.0.1:27100`.
5. Find out what happens when you restore a dump into a collection that has a unique index the dumped
   data violates.
