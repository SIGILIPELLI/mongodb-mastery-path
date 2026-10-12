---
description: "Use mongosh well: connection strings, cursors and batches, scripts with --eval and --file, JSON output modes, .mongoshrc.js helpers and error handling."
---

# 02 · mongosh Like a Pro

`mongosh` is a full Node.js-based JavaScript REPL with a MongoDB connection attached. That one fact
explains most of its behaviour: you can declare variables, write loops and functions, `require`
Node modules, and every `db.collection.method()` call is the Node.js driver underneath. Knowing the
shell well saves you from writing throwaway scripts for every question you want to ask a database.

All examples run against the lab replica set from this course's setup (`rs0` on port 27017).

## Connecting

```bash
mongosh                                         # mongodb://127.0.0.1:27017, database "test"
mongosh --port 27100                            # another local server
mongosh "mongodb://127.0.0.1:27017/shell101"    # connection string with a default database
mongosh "mongodb://app:secret@db1:27017,db2:27017/shop?replicaSet=rs0&authSource=admin"
```

The connection string (URI) is the same format every driver accepts, so a URI that works in the
shell will work in your application. The path part (`/shell101`) selects the starting database:

```text
$ mongosh "mongodb://127.0.0.1:27017/shell101" --quiet --eval 'db.getName()'
shell101
```

Never put real passwords on the command line on a shared machine — they show up in `ps` output and
shell history. Use `mongosh --username app` and let it prompt, or a config file readable only by you.

## The prompt tells you where you are

```text
rs0 [direct: primary] shell101>
```

That is: replica set `rs0`, connected directly to the member that is currently **primary**, using
database `shell101`. If it says `secondary`, writes will fail — useful to notice before you paste a
migration.

## Moving between databases

```javascript
use shell101                 // REPL-only helper: switches the shell's `db`
db.getSiblingDB("shell101")  // works everywhere, including scripts and --eval
show dbs                     // REPL-only
show collections             // REPL-only
db.getCollectionNames()      // works everywhere
```

`use` and `show` are shell commands, not JavaScript, so they are not available inside a function or
a `.js` file you load. Make a habit of `getSiblingDB` in anything you might save.

## Cursors and batches

`find()` does not return documents. It returns a **cursor**, and the shell prints the first batch of
results (20 by default) when the cursor is the last expression in the line. With 45 documents in a
collection:

```text
rs0 [direct: primary] shell101> db.nums.find({}, {_id:0})
[
  { n: 1, sq: 1, even: false },
  ...
  { n: 20, sq: 400, even: true }
]
Type "it" for more
rs0 [direct: primary] shell101> it
[ ... n: 21 to 40 ... ]
Type "it" for more
rs0 [direct: primary] shell101> it
[
  { n: 41, sq: 1681, even: false },
  { n: 42, sq: 1764, even: true },
  { n: 43, sq: 1849, even: false },
  { n: 44, sq: 1936, even: true },
  { n: 45, sq: 2025, even: false }
]
```

You can change the page size, and the setting persists between sessions:

```text
rs0 [direct: primary] shell101> config.set("displayBatchSize", 3)
Setting "displayBatchSize" has been changed
rs0 [direct: primary] shell101> db.nums.find({},{_id:0,n:1}).sort({n:1})
[ { n: 1 }, { n: 2 }, { n: 3 } ]
Type "it" for more
```

Other ways to consume a cursor:

```javascript
db.nums.find({ even: true }).sort({ n: 1 }).toArray()   // everything, as an array
db.nums.find().itcount()                                // iterate and count client-side

const c = db.nums.find({ even: true }).sort({ n: 1 }).limit(3);
while (c.hasNext()) { const d = c.next(); print(d.n, d.sq); }
```

```text
2 4
4 16
6 36
```

One trap: `cursor.map()` looks like `Array.prototype.map` but returns another **cursor**:

```text
> const r = db.nums.find({even:true}).sort({n:1}).limit(4).map(d => d.n)
> r.constructor.name
Cursor
> db.nums.find({even:true}).map(d => d.n).slice(0, 5)
TypeError: db.nums.find({ ... =>d.n).slice is not a function
> r.toArray().join(",")
2,4,6,8
```

Call `.toArray()` when you want an array.

## Scripts: --eval and files

For anything you will run more than once, write a file:

```javascript
// seed.js
const d = db.getSiblingDB("shell101");
d.dropDatabase();
const docs = [];
for (let i = 1; i <= 45; i++) docs.push({ n: i, sq: i * i, even: i % 2 === 0 });
d.nums.insertMany(docs);
print("inserted", d.nums.countDocuments());
```

```text
$ mongosh --quiet seed.js
inserted 45
```

`--quiet` suppresses the connection banner. In a script, nothing is printed automatically — use
`print()` for plain values or `printjson()` for documents. Inside the REPL you may also `load("seed.js")`.

## Output formats for other tools

When another program will read the output, ask for real JSON instead of the shell's JavaScript-ish
display. `--json=relaxed` gives readable JSON:

```text
$ mongosh --quiet shell101 --json=relaxed --eval 'db.nums.find({n:{$lte:2}},{_id:0}).sort({n:1}).toArray()'
[
  {
    "n": 1,
    "sq": 1,
    "even": false
  },
  {
    "n": 2,
    "sq": 4,
    "even": true
  }
]
```

`--json=canonical` preserves every BSON type exactly, at the cost of verbosity — this is
**Extended JSON**, which you will meet again with `mongoexport`:

```text
$ mongosh --quiet shell101 --json=canonical --eval 'db.nums.findOne({n:3},{_id:0})'
{
  "n": {
    "$numberInt": "3"
  },
  "sq": {
    "$numberInt": "9"
  },
  "even": false
}
```

## Numbers in the shell

JavaScript has one number type; BSON has several. When the shell inserts a whole number that fits in
32 bits it sends an `int`, otherwise a `double`. Use the constructors when the type matters:

```javascript
db.types.insertOne({ a: 1, b: 1.5, c: NumberInt(7), d: NumberLong("7"), e: new Date(0) })
db.types.aggregate([{ $project: { _id: 0, a: { $type: "$a" }, b: { $type: "$b" },
  c: { $type: "$c" }, d: { $type: "$d" }, e: { $type: "$e" } } }])
```

```text
[ { a: 'int', b: 'double', c: 'int', d: 'long', e: 'date' } ]
```

Pass `NumberLong` a **string**. Passing a number works but the shell warns, because a JavaScript
number above 2^53 has already lost precision before `NumberLong` sees it:

```text
Warning: NumberLong: specifying a number as argument is deprecated and may lead to loss of precision, pass a string instead
```

Lesson 03 covers BSON types properly.

## Errors are exceptions

Server errors are thrown as `MongoServerError` with a numeric `code`, so you can handle them in
scripts:

```javascript
try {
  db.nums.insertOne({ _id: 1 });
  db.nums.insertOne({ _id: 1 });
} catch (e) {
  print(e.name, e.code, e.message);
}
```

```text
MongoServerError 11000 E11000 duplicate key error collection: shell101.nums index: _id_ dup key: { _id: 1 }
```

Code `11000` (duplicate key) is the one you will catch most often in application code. Match on the
code, never on the message text.

## Your own helpers: .mongoshrc.js

On start, `mongosh` runs `~/.mongoshrc.js` if it exists (skip it with `--norc`). This one changes the
prompt and adds a helper:

```javascript
// ~/.mongoshrc.js
prompt = () => `${db.getName()}@${db.hello().me ?? "standalone"}> `;
function topN(coll, field, n = 3) {
  return db[coll].find({}, { _id: 0, [field]: 1 }).sort({ [field]: -1 }).limit(n).toArray();
}
```

```text
test@127.0.0.1:27017> use shell101
switched to db shell101
shell101@127.0.0.1:27017> topN("nums","sq")
[ { sq: 2025 }, { sq: 1936 }, { sq: 1849 } ]
```

Keep anything destructive out of your rc file. A helper that drops a collection is one autocomplete
away from a bad day.

## How It Actually Works

`mongosh` is a Node.js application. It wraps the official Node.js driver in a layer called the
**shell API** that turns the asynchronous driver into something that *looks* synchronous: in the
driver, `collection.findOne()` returns a Promise; in the shell you write `db.nums.findOne()` and get
a document. The REPL inspects each line, and if an expression returns a Promise it awaits it for you
before printing. That is why loops like `while (c.hasNext())` work without `await` in the shell but
the same code copied into a Node program does not.

`db.nums.find()` creates a cursor object but sends nothing yet. When the cursor is first iterated —
by the REPL printer, `toArray()`, `next()` or `forEach()` — the driver sends a `find` command. The
server replies with the first batch and a **cursor id**; to get more, the driver sends `getMore` with
that id. Typing `it` is just one more `getMore` (or the next slice of an already-fetched batch). A
server-side cursor that is not used for 10 minutes is closed by the server, which is why an old cursor
in a forgotten shell eventually fails with "cursor not found".

Values are converted between JavaScript and BSON by the driver's BSON library. Integers in 32-bit range
become `int`, other numbers `double`, `Date` objects become BSON `date`, and the wrapper classes
(`NumberLong`, `NumberDecimal`, `ObjectId`, `UUID`…) map to their BSON types. On the way back,
`int` and `double` become plain JavaScript numbers, and `long` becomes a `Long` object so no precision
is lost — which is why you sometimes see `Long('3758096384')` instead of a bare number.

## Common mistakes

- **Using `use` or `show` in a script.** They are REPL commands; use `getSiblingDB` and
  `getCollectionNames()`.
- **Expecting `cursor.map()` to return an array.** It returns a cursor; add `.toArray()`.
- **Reading from a cursor's printout as if it were complete.** The REPL only shows one batch; a
  "result" of 20 documents may be 20 of 20 million.
- **Building numbers that need exact precision in JavaScript.** Use `NumberLong("…")` and
  `NumberDecimal("…")` with string arguments.
- **Running ad-hoc writes against production in a REPL you also use for local work.** Check the
  prompt before pasting, and consider a different prompt colour or text per environment in your rc
  file.

## Exercise

1. Create a `practice.squares` collection with documents for 1–100 using a script file, run it with
   `--quiet`, and print the count.
2. In the REPL, set `displayBatchSize` to 7 and page through the even squares with `it`. Reset it to 20
   afterwards.
3. Print the five largest squares as relaxed JSON with `--json=relaxed --eval`, and pipe the output into
   `python3 -m json.tool` (or `jq`) to prove it is valid JSON.
4. Insert one document with `NumberLong("9007199254740993")` and another with the plain number
   `9007199254740993`. Read both back and explain the difference.
5. Add a helper to a `.mongoshrc.js` that prints every collection in the current database with its
   document count.
