---
description: "Enforce document structure with $jsonSchema: required fields, types, enums, cross-field $expr rules, oneOf variants, and rolling rules onto old data."
---

# 04 · Schema Validation with $jsonSchema

"Schemaless" is the most misleading word ever attached to MongoDB. Every application has a schema — the
only question is whether it is written down and enforced in one place, or scattered across the code of
every service that writes to the database. A flexible schema is a feature while a design is evolving; in
production, a document with `tier: "gold"` when the code only knows `"free"`, `"pro"` and `"enterprise"`
is a bug waiting for a customer to find it.

MongoDB lets each collection carry a **validator**, checked by the server on every insert and update. This
lesson builds validators, reads their error reports, and — the part that matters in real systems — adds
them to collections that already contain bad data.

## A first validator

```javascript
db.createCollection("customers", { validator: { $jsonSchema: {
  bsonType: "object", required: ["email", "name", "tier"],
  properties: {
    email: { bsonType: "string", pattern: "^[^@\\s]+@[^@\\s]+$", description: "must be an email address" },
    name: { bsonType: "string", minLength: 1 },
    tier: { enum: ["free", "pro", "enterprise"] },
    credit: { bsonType: "decimal", minimum: 0 },
    tags: { bsonType: "array", maxItems: 5, items: { bsonType: "string" }, uniqueItems: true }
  } } } });
```

`$jsonSchema` is MongoDB's dialect of JSON Schema (draft 4), with `bsonType` added so you can require BSON
types that JSON doesn't have — `decimal`, `date`, `objectId`, `long`. Fields not listed under `properties`
are allowed unless you set `additionalProperties: false`.

A valid document goes in as usual. An invalid one:

```javascript
db.customers.insertOne({ email: "not-an-email", name: "", tier: "gold", credit: -5 })
```

```text
121 Document failed validation
```

Code 121 is the validation error. The exception's `errInfo.details` explains exactly what failed — every
rule, not just the first:

```text
[
  {
    operatorName: 'properties',
    propertiesNotSatisfied: [
      { propertyName: 'email', description: 'must be an email address',
        details: [ { operatorName: 'pattern', specifiedAs: { pattern: '^[^@\\s]+@[^@\\s]+$' },
                     reason: 'regular expression did not match', consideredValue: 'not-an-email' } ] },
      { propertyName: 'name',
        details: [ { operatorName: 'minLength', specifiedAs: { minLength: 1 },
                     reason: 'specified string length was not satisfied', consideredValue: '' } ] },
      { propertyName: 'tier',
        details: [ { operatorName: 'enum', specifiedAs: { enum: [ 'free', 'pro', 'enterprise' ] },
                     reason: 'value was not found in enum', consideredValue: 'gold' } ] },
      { propertyName: 'credit',
        details: [ { operatorName: 'minimum', specifiedAs: { minimum: 0 },
                     reason: 'comparison failed', consideredValue: -5 },
                   { operatorName: 'bsonType', specifiedAs: { bsonType: 'decimal' },
                     reason: 'type did not match', consideredValue: -5, consideredType: 'int' } ] }
    ]
  }
]
```

Log `errInfo` when a write fails with code 121; it turns a vague "validation failed" report into a
one-line fix. Note the `credit` entry: `-5` failed twice, because it's below the minimum **and** it's an
`int` rather than a `decimal`. Type rules are exact — a validator requiring `decimal` rejects a perfectly
reasonable `int`, so make sure every writer sends the right type (Level 1 · 08 showed how PyMongo needs
`Decimal128`).

## Updates are validated too

The server validates the document **as it will be after the update**:

```javascript
db.customers.updateOne({ email: "ana@example.com" }, { $set: { tier: "platinum" } })
db.customers.updateOne({ email: "ana@example.com" }, { $unset: { name: "" } })
```

```text
update: 121 Plan executor error during update :: caused by :: Document failed validation
unset: 121 Plan executor error during update :: caused by :: Document failed validation
```

That's the real value of server-side validation: it applies to the admin script someone runs at 2 a.m.
just as much as to your carefully reviewed application code.

## Rules across fields: $expr

`$jsonSchema` validates fields one at a time. For rules relating fields to each other, combine it with a
query expression:

```javascript
db.createCollection("bookings", { validator: { $and: [
  { $jsonSchema: { required: ["start", "end"],
                   properties: { start: { bsonType: "date" }, end: { bsonType: "date" } } } },
  { $expr: { $lt: ["$start", "$end"] } } ] } });
db.bookings.insertOne({ start: new Date("2026-10-12"), end: new Date("2026-10-10") })
```

```text
bookings: 121
{
  operatorName: '$and',
  clausesNotSatisfied: [
    { index: 1, details: { operatorName: '$expr', specifiedAs: { '$expr': { '$lt': [ '$start', '$end' ] } },
                           reason: 'expression did not match', expressionResult: false } }
  ]
}
```

Any query operator can appear in a validator, except a few that need context (`$near`, `$text`, `$where`).

## Variants: oneOf

For polymorphic collections (lesson 03), `oneOf` requires a document to match exactly one variant:

```javascript
db.createCollection("payments", { validator: { $jsonSchema: {
  bsonType: "object", required: ["type"], oneOf: [
    { properties: { type: { enum: ["card"] }, last4: { bsonType: "string", pattern: "^[0-9]{4}$" } }, required: ["last4"] },
    { properties: { type: { enum: ["upi"] }, vpa: { bsonType: "string" } }, required: ["vpa"] } ] } } });
db.payments.insertOne({ type: "upi", vpa: "ana@bank" })     // accepted
db.payments.insertOne({ type: "card", vpa: "ana@bank" })    // 121: a card without last4
```

## Adding a validator to a collection that already has data

This is the situation you'll actually be in. A `legacy` collection has three documents, two of them bad:

```javascript
db.legacy.insertMany([{ _id: 1, email: "a@x.io", qty: 3 }, { _id: 2, email: "bad", qty: -1 }, { _id: 3, qty: 2 }]);
const schema = { $jsonSchema: { required: ["email"],
  properties: { email: { bsonType: "string", pattern: "@" }, qty: { bsonType: "int", minimum: 0 } } } };
```

**Step 1: find the violators first.** A validator is just a query filter, so `$nor` finds every document
that would fail it:

```javascript
db.legacy.find({ $nor: [schema] })
```

```text
violating now: 2,3
```

**Step 2: add the rule without breaking existing writes.** `collMod` sets the validator. With
`validationLevel: "moderate"`, documents that *already* violate the rules can still be updated, while valid
documents must stay valid and new inserts must be valid:

```javascript
db.runCommand({ collMod: "legacy", validator: schema, validationLevel: "moderate" });
db.legacy.updateOne({ _id: 2 }, { $set: { note: "still invalid but already was" } })  // modifiedCount 1
db.legacy.updateOne({ _id: 1 }, { $set: { qty: -9 } })                                // 121
db.legacy.insertOne({ _id: 4, email: "nope" })                                         // 121
```

**Or observe before enforcing.** `validationAction: "warn"` accepts invalid writes but logs them:

```javascript
db.runCommand({ collMod: "legacy", validationAction: "warn" });
db.legacy.insertOne({ _id: 5, email: "nope" })   // acknowledged: true
```

The server log gained a line with message `Document would fail validation` (log id 20294). Run in warn mode
for a while, grep the log to find which writers send bad data, fix them, then switch to `"error"`.

**Step 3: fix the old documents** (with a migration, lesson 03's schema-versioning approach or a one-off
script), and finally set `validationLevel: "strict"` so every write is checked.

## Escape hatches

Users with the `bypassDocumentValidation` privilege can skip validation per command:

```javascript
db.runCommand({ insert: "customers", documents: [{ x: 1 }], bypassDocumentValidation: true })
```

The document `{x: 1}` went into `customers` despite having none of the required fields. Restore tools and
data-repair scripts sometimes need this; ordinary application users should never have that privilege
(Level 4 · 05). Also note that `mongorestore` validates documents unless told otherwise
(`--bypassDocumentValidation`).

## Where validation belongs

Server-side validation is the **last** line of defence, not the only one. Validate in the application too
(Pydantic, Zod, your framework's forms), because that's where you can give a user a friendly message. The
database validator catches what the application misses: other services, scripts, old code paths, and bugs.
Keep it to rules that are truly invariant — types, required fields, enums, ranges — and leave volatile
business rules in code.

## How It Actually Works

The validator is stored in the collection's catalog entry. On every insert, and on every update after the
update has been applied to an in-memory copy, the server evaluates the validator against the new document
using the **same match-expression engine** that runs `find()` filters — `$jsonSchema` is compiled into a
tree of match expressions, which is why you can combine it with `$expr` and other query operators, and why
`find({$nor: [schema]})` finds violators. If the expression evaluates false, the write is rejected before it
reaches the storage engine (`validationAction: "error"`) or logged and allowed (`"warn"`).

`validationLevel: "moderate"` works by evaluating the validator against the document **before** the update
too: if the old version was already invalid, the update is allowed regardless. That is why the update to
document 2 succeeded while the update that would have made document 1 invalid failed.

Validation is cheap compared to the write itself, but not free — a large schema with regex patterns is
evaluated on every write. And it only applies on writes: changing a validator never re-checks existing
documents, so the `$nor` query is the only way to know what's already in there.

## Common mistakes

- **No validator at all** on collections written by more than one program.
- **Adding a strict validator to a live collection** without checking existing data, then discovering that
  routine updates to old documents now fail.
- **Requiring `double` or `decimal`** when writers send integers (or vice versa).
- **`additionalProperties: false` without listing `_id`** — every insert fails, because `_id` is a property
  too.
- **Putting fast-changing business rules in the validator**, so every rule change becomes a database change.
- **Giving application users `bypassDocumentValidation`.**

## Exercise

1. Write a validator for an `orders` collection: required `customerId` (objectId), `status` (enum),
   `items` (non-empty array of `{sku: string, qty: int ≥ 1, price: decimal ≥ 0}`), and `total` (decimal).
   Insert one valid and three different invalid orders and read each `errInfo`.
2. Add an `$expr` rule that `shippedAt`, if present, must be later than `placedAt`. (Hint: `$or` with
   `$exists`-like logic using `{ $eq: [{ $type: "$shippedAt" }, "missing"] }`.)
3. Take the `library.books` collection from Level 1 · 04, write a schema for it, and list the violators
   with `$nor`. Add the validator with `moderate` + `warn`, perform some writes, and find the warnings in the
   log.
4. Try `additionalProperties: false` without `_id` in `properties` and observe what happens on insert.
