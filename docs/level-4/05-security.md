---
description: "Secure MongoDB: the localhost exception, least-privilege users and custom roles, TLS and x.509 client certificates, and keyFile member auth."
---

# 05 · Authentication, Roles & TLS

Every lab in this course so far has run without authentication, bound to `127.0.0.1`. That is fine on a laptop and
catastrophic anywhere else: for years, automated scanners have found MongoDB servers exposed to the internet with
no authentication, copied or deleted their data, and left ransom notes in its place. None of those attacks needed
an exploit — only a server that accepted connections from anyone and trusted everyone who connected.

This lesson turns on the three defences that matter most — **authentication** (who are you?), **authorization**
(what may you do?) and **encryption in transit** (can anyone read or alter the traffic?) — and shows the exact
errors you'll hit along the way, because most of the real work of securing MongoDB is understanding those errors.
Everything ran on MongoDB 9.0.2 Community with OpenSSL 4.0 for certificates. Passwords in the examples are
throwaway lab values; in real use, prompt for them (`passwordPrompt()` in mongosh) or load them from a secret store.

## Turning on authentication

Start `mongod` with `--auth` (or `security.authorization: enabled` in the config file):

```bash
mongod --port 27501 --dbpath db --bind_ip 127.0.0.1 --auth --logpath sec.log &
```

There are no users yet, so nothing is permitted:

```text
before any user: Unauthorized - not authorized on shop to execute command { find: "orders", filter: {}, limit: 1, ... }
```

The **localhost exception** solves the chicken-and-egg problem: while a server has no users, a client connecting
from the same machine may create **one** user in the `admin` database. Make it a user administrator, not an
all-powerful `root`:

```javascript
db.getSiblingDB("admin").createUser({user: "ops", pwd: passwordPrompt(),
                                     roles: ["userAdminAnyDatabase", "clusterMonitor"]})
```

```text
created first user through the localhost exception
second createUser without auth: Unauthorized
```

The exception closed the moment the first user existed. From now on, even on localhost, you must authenticate.

## Users live in a database

A user is defined in an **authentication database** — `admin` for administrators, or the application's own
database — and its roles can grant privileges on any database:

```javascript
const shop = db.getSiblingDB("shop")
shop.createUser({user: "shop_app", pwd: passwordPrompt(), roles: [{role: "readWrite", db: "shop"}]})
```

Clients must name the authentication database when they connect (`authSource`, or the path in the URI):

```text
mongodb://shop_app:<password>@127.0.0.1:27501/shop
mongodb://ops:<password>@127.0.0.1:27501/admin
```

Passwords are stored as SCRAM credentials (salted, iterated hashes), never in plain text:

```text
["SCRAM-SHA-1","SCRAM-SHA-256"]
```

A failed login is logged with the mechanism and reason — useful when debugging, and worth alerting on in bulk:

```text
"msg":"Failed to authenticate","attr":{"client":"127.0.0.1:54067", ... "mechanism":"SCRAM-SHA-256",
"user":"reporter","db":"shop","error":"AuthenticationFailed: SCRAM authentication failed, storedKey mismatch"}
```

## What the built-in roles actually allow

Roles are often assigned by name without anyone checking what the name permits. Three surprises from this lab:

```text
ops (userAdminAnyDatabase) inserting: Unauthorized
shop_app reading admin.system.users: Unauthorized
shop_app dropped orders?  [ 'customers' ]
```

- **`userAdminAnyDatabase` can't read or write data**, only manage users and roles. But it can grant *itself* any
  role — it is effectively root with one extra step, so guard it as carefully.
- **`readWrite` includes dropping collections.** The application user deleted the whole `orders` collection with
  one call. If your application never drops collections, `readWrite` is more than it needs.
- Users in one database can't see another's data unless granted — the app user can't read `admin`.

The common built-in roles:

| Role | Scope | Can |
|------|-------|-----|
| `read` / `readWrite` | one database | query / query, insert, update, delete, **create and drop collections and indexes** |
| `dbAdmin` | one database | indexes, stats, profiler, validation — but not read data |
| `userAdmin` / `userAdminAnyDatabase` | one / all databases | create users and roles, grant roles (including to itself) |
| `clusterMonitor` | cluster | read-only monitoring commands (`serverStatus`, `replSetGetStatus`, …) |
| `backup` / `restore` | cluster | what `mongodump` / `mongorestore` need |
| `root` | everything | everything |

## Custom roles for least privilege

A reporting service should read orders and nothing else. A custom role lists exact **actions** on exact
**resources**:

```javascript
shop.createRole({role: "ordersReporter",
  privileges: [{resource: {db: "shop", collection: "orders"}, actions: ["find"]}],
  roles: []})
shop.createUser({user: "reporter", pwd: passwordPrompt(), roles: ["ordersReporter"]})
```

```text
reporter find orders: 2
reporter find customers: Unauthorized
reporter update orders: Unauthorized
200
reporter $lookup into customers: Unauthorized
```

The `find` action covers queries *and* aggregation on that collection (the `200` is a `$group` total), but a
`$lookup` into `customers` is refused: every collection a pipeline reads is checked. Writing roles this narrowly
takes a few minutes per service and turns a leaked credential from "attacker has the database" into "attacker can
read one collection".

A sensible set for most applications:

- one **user administrator** (stored in a password manager, rarely used);
- one **application user per service**, with a custom role covering only the collections and actions it uses
  (often `find`, `insert`, `update`, `remove` — without `dropCollection`);
- a **migration user** with index and collection management, used only by the deploy pipeline;
- **read-only** users for humans and reporting; **`backup`** for the backup job; **`clusterMonitor`** for monitoring.

## TLS: encrypting the connection

Without TLS, every query, result and SCRAM exchange crosses the network in readable form. A lab certificate
authority and server certificate with OpenSSL — the server's certificate must name the host clients will use, in
its Subject Alternative Name:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 30 -subj "/CN=Lab CA" -keyout ca.key -out ca.pem
openssl req -newkey rsa:2048 -nodes -subj "/CN=127.0.0.1" -keyout server.key -out server.csr
printf "subjectAltName=IP:127.0.0.1,DNS:localhost\nextendedKeyUsage=serverAuth" > san.ext
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 30 -extfile san.ext -out server.crt
cat server.crt server.key > server.pem
```

```bash
mongod --port 27502 --dbpath tlsdb --bind_ip 127.0.0.1 --tlsMode requireTLS \
       --tlsCertificateKeyFile server.pem --tlsCAFile ca.pem --logpath tls.log &
```

Then four attempts to connect, each failing for a different, instructive reason:

```text
--- plain connection:
MongoServerSelectionError: connection <monitor> to 127.0.0.1:27502 closed
--- TLS without trusting the CA:
MongoNetworkError: unable to get local issuer certificate
--- TLS with CA:
MongoServerSelectionError: connection <monitor> to 127.0.0.1:27502 closed
```

The first is `requireTLS` refusing plain text. The second is the client refusing a certificate signed by an
authority it doesn't trust — exactly what should happen. The third is the surprise: the client trusted the server,
but the server's log says why it hung up:

```text
"msg":"SSL peer certificate validation failed; connection rejected","attr":{"error":"No SSL certificate provided by peer"}
```

Giving `mongod` a `--tlsCAFile` also makes it **require client certificates** signed by that CA. That's a strong
setting (mutual TLS) but it's a choice. If clients should authenticate with passwords over TLS instead, add
`--tlsAllowConnectionsWithoutCertificates`. And the CA file can't simply be left out — this version refuses to
start:

```text
"errmsg":"The use of TLS without specifying a chain of trust is no longer supported. See https://jira.mongodb.org/browse/SERVER-72839 for details."
```

Host name verification is the other half of TLS. A server whose certificate names `db.lab.internal`, reached as
`127.0.0.1`:

```text
MongoServerSelectionError: Hostname/IP does not match certificate's altnames: IP: 127.0.0.1 is not in the cert's list:
--- same with tlsAllowInvalidHostnames:
connected 9.0.2
```

`tlsAllowInvalidHostnames` (and its cousin `tlsAllowInvalidCertificates`) make the error go away by turning off
the check that stops a man-in-the-middle. Fix the certificate or the connection string instead.

## x.509: certificates as identities

With mutual TLS in place, the client certificate can *be* the credential. A certificate for a service:

```bash
openssl req -newkey rsa:2048 -nodes -subj "/O=Lab/OU=apps/CN=billing-svc" -keyout client.key -out client.csr
printf "extendedKeyUsage=clientAuth" > cli.ext
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 30 -extfile cli.ext -out client.crt
cat client.crt client.key > client.pem
openssl x509 -in client.pem -noout -subject -nameopt RFC2253
```

```text
subject=CN=billing-svc,OU=apps,O=Lab
```

The user is created in the virtual `$external` database, named by the certificate's subject exactly as RFC 2253
prints it:

```javascript
db.getSiblingDB("$external").createUser({user: "CN=billing-svc,OU=apps,O=Lab",
                                         roles: [{role: "read", db: "billing"}]})
```

```text
mongodb://127.0.0.1:27502/?tls=true&tlsCAFile=ca.pem&tlsCertificateKeyFile=client.pem&authMechanism=MONGODB-X509&authSource=$external
[ { user: 'CN=billing-svc,OU=apps,O=Lab', db: '$external' } ]
```

No password exists to leak; revoking access means removing the user or letting the certificate expire. The cost
is running a certificate authority properly — issuing, rotating and revoking. (This TLS server ran without `--auth`
to keep the experiment focused on TLS; in production you'd combine both.)

## Replica sets and clusters: member authentication

Members of a replica set or sharded cluster must authenticate to *each other*. With `--auth`, that's not optional:

```text
BadValue: security.keyFile is required when authorization is enabled with replica sets
```

The simplest mechanism is a **keyFile**: a shared secret every member reads.

```bash
openssl rand -base64 756 > keyfile && chmod 400 keyfile
mongod --replSet rs4 --keyFile keyfile ...      # --keyFile implies --auth
```

Every `mongod` and `mongos` in the deployment needs the same file. In production, x.509 member certificates
(`--clusterAuthMode x509`) are stronger: no shared secret, and each member has its own identity. Lesson 10 builds
a cluster with keyFile auth end to end.

## Beyond the basics

- **Network exposure** is the first control: `bindIp` only on private interfaces, firewall rules or security
  groups allowing only application hosts, never a public address.
- **Encryption at rest** (WiredTiger's encrypted storage engine), **auditing**, **LDAP** and **Kerberos**
  authentication are MongoDB Enterprise / Atlas features — not in the Community server used here, so not
  demonstrated. Community deployments usually rely on encrypted disks at the OS or cloud level.
- **Client-side field level encryption** and **Queryable Encryption** encrypt chosen fields in the driver so the
  server never sees plaintext; they need extra libraries and key management, and parts depend on edition and
  version — check current documentation before designing around them.

## How It Actually Works

SCRAM (Salted Challenge Response Authentication Mechanism) never sends the password. The server stores a salt, an
iteration count, and two keys derived from the password (`StoredKey`, `ServerKey`). On login the client and server
exchange nonces; the client proves it knows the password by sending a proof computed from the salted password and
the conversation, and the server proves it knows the stored credential by returning a signature the client checks.
A stolen user document allows offline guessing (which the salt and iterations slow down) but not direct login —
that's what "storedKey mismatch" in the log refers to.

After authentication the connection has a set of **users**, each with **roles**; roles resolve (recursively,
through inherited roles) into **privileges** — pairs of a resource (cluster, database, collection) and a list of
actions. Every command declares which actions on which resources it needs; before running it, the server checks
the connection's privilege set. An aggregation's check includes every namespace it touches, which is why the
`$lookup` failed. TLS happens a layer below all of this: the TLS handshake (certificate validation on both sides
when client certificates are required) completes before MongoDB's wire protocol starts, so a failed handshake
never reaches authentication at all — which is why the "TLS with CA" attempt showed only a closed connection on
the client and the real reason only in the server log.

## Common mistakes

- **Exposing `mongod` on a public address**, with or without auth.
- **Giving applications `root` or `dbOwner`**, or one shared user for every service.
- **Assuming `readWrite` is harmless** — it can drop collections.
- **Treating `userAdminAnyDatabase` as low-risk.** It can grant itself anything.
- **Disabling host name or certificate checks** (`tlsAllowInvalidHostnames`, `tlsAllowInvalidCertificates`) to
  silence errors.
- **Forgetting the authentication database** (`authSource`), then debugging "Authentication failed" for an hour.
- **Passwords in connection strings committed to git**, or special characters in passwords not URL-encoded.
- **Copying one keyFile into a container image** shared across environments.

## Exercise

1. Start a fresh `mongod --auth`, create the user administrator through the localhost exception, and confirm you
   can't create a second user without logging in.
2. Write a custom role `shopService` with `find`, `insert`, `update` and `remove` on `shop.orders` and `find` on
   `shop.products` — and nothing else. Prove that it can't drop a collection, create an index, or read `customers`.
3. Rotate the `shop_app` password with `db.changeUserPassword` while a Python client stays connected. Do existing
   connections break? Do new ones work with the old password?
4. Set up TLS with a CA, a server certificate and `--tlsAllowConnectionsWithoutCertificates`. Connect with PyMongo
   using `tlsCAFile`. Then capture traffic with `tcpdump` (or Wireshark) with and without TLS and compare what you
   can read.
5. Convert your three-node replica set from Lesson 01 to keyFile authentication with a rolling restart (start with
   `transitionToAuth` on each member, then without it), keeping a writer running throughout.
