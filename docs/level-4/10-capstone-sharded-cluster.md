---
description: "Capstone: a ten-process MongoDB sharded cluster with TLS, keyFile auth and least-privilege roles, proven by a 12-check script with a failover."
---

# 10 · Capstone — A Secured Sharded Cluster

This capstone puts Level 4 together. You'll build a sharded cluster the way it's built for production — every
component a three-member replica set, TLS required on every connection, members authenticating to each other,
least-privilege users for each client — and then accept it the way production systems should be accepted: with
a script that tries to break each guarantee and reports PASS or FAIL.

The finished cluster is ten processes on one machine:

| Component | Members | Ports |
|-----------|---------|-------|
| Config server replica set `cfg` | 3 | 27901–27903 |
| Shard replica set `shardA` | 3 | 27911–27913 |
| Shard replica set `shardB` | 3 | 27921–27923 |
| Router `mongos` | 1 | 27900 |

Everything below was run on MongoDB 9.0.2 with PyMongo 4.18.3 and OpenSSL 4.0, including two mistakes that turned
into the most useful parts of the lesson. On a laptop the ten processes use a few hundred MB each with the cache
capped at 0.25 GB; close other labs first.

## Requirements

1. All three replica sets have three data-bearing members; losing any one member loses nothing and stops nothing.
2. Every connection uses TLS; plain-text connections are refused.
3. Members authenticate to each other; clients must authenticate; no anonymous access.
4. Three client identities: a cluster administrator, an application user that can read and write orders (but not
   drop or index them, or touch other collections beyond reading products), and a read-only reporting user.
5. `shop.orders` is sharded on `{customerId: 1, placedAt: 1}` (Lesson 04), spread across both shards, with
   customer queries targeted to one shard.
6. An acceptance script verifies all of the above, including a shard primary killed during writes.

## Step 1: secrets and certificates

```bash
openssl rand -base64 756 > keyfile && chmod 400 keyfile

openssl req -x509 -newkey rsa:2048 -nodes -days 30 -subj "/CN=Capstone CA" -keyout ca.key -out ca.pem
openssl req -newkey rsa:2048 -nodes -subj "/O=Capstone/OU=cluster/CN=127.0.0.1" -keyout member.key -out member.csr
printf "subjectAltName=IP:127.0.0.1,DNS:localhost\nextendedKeyUsage=serverAuth,clientAuth\n" > member.ext
openssl x509 -req -in member.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 30 -extfile member.ext -out member.crt
cat member.crt member.key > member.pem && chmod 600 member.pem ca.key
```

The member certificate carries **both** `serverAuth` and `clientAuth`, because members are clients of each other
when they replicate and heartbeat. In production every host gets its own certificate naming its own DNS name; one
certificate for `127.0.0.1` works here only because every member shares that address.

## Step 2: one configuration file per member

Command-line flags are fine for experiments; production uses configuration files, which are reviewable and
repeatable. The file for `a1`:

```yaml
storage:
  dbPath: /path/to/cap/data/a1
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.25
systemLog:
  destination: file
  path: /path/to/cap/logs/a1.log
  logAppend: true
net:
  port: 27911
  bindIp: 127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/cap/member.pem
    CAFile: /path/to/cap/ca.pem
    allowConnectionsWithoutCertificates: true
security:
  keyFile: /path/to/cap/keyfile
  authorization: enabled
replication:
  replSetName: shardA
  oplogSizeMB: 64
sharding:
  clusterRole: shardsvr
```

A small shell function generated all nine, varying the name, port, `clusterRole` (`configsvr` or `shardsvr`) and
replica set name. `allowConnectionsWithoutCertificates` lets clients authenticate with passwords over TLS (Lesson
05 explains why `CAFile` alone would demand client certificates).

## Step 3: start and initiate

```bash
for f in cfg1 cfg2 cfg3 a1 a2 a3 b1 b2 b3; do mongod --config $f.conf & done
```

A plain-text connection is already refused:

```text
MongoServerSelectionError: connection <monitor> to 127.0.0.1:27901 closed
```

With no users anywhere, the **localhost exception** lets you initiate each replica set over TLS:

```javascript
// mongosh --host 127.0.0.1 --port 27901 --tls --tlsCAFile ca.pem
rs.initiate({_id: "cfg", configsvr: true, members: [{_id: 0, host: "127.0.0.1:27901"},
  {_id: 1, host: "127.0.0.1:27902"}, {_id: 2, host: "127.0.0.1:27903"}]})
// likewise shardA on 27911 and shardB on 27921 (without configsvr)
```

```text
27901=PRIMARY 27902=SECONDARY 27903=SECONDARY
27911=PRIMARY 27912=SECONDARY 27913=SECONDARY
27921=PRIMARY 27922=SECONDARY 27923=SECONDARY
```

## Step 4: router, first administrator, shards, roles

`mongos.conf` has the same `net.tls` and `security.keyFile` sections and `sharding.configDB:
cfg/127.0.0.1:27901,127.0.0.1:27902,127.0.0.1:27903`. Through `mongos`, the localhost exception creates the first
user — stored on the config servers, so it is a *cluster* user:

```javascript
db.getSiblingDB("admin").createUser({user: "clusterAdmin", pwd: passwordPrompt(), roles: ["root"]})
```

Then, authenticated as `clusterAdmin`:

```javascript
sh.addShard("shardA/127.0.0.1:27911,127.0.0.1:27912,127.0.0.1:27913")
sh.addShard("shardB/127.0.0.1:27921,127.0.0.1:27922,127.0.0.1:27923")

const shop = db.getSiblingDB("shop")
shop.createRole({role: "shopService", roles: [], privileges: [
  {resource: {db: "shop", collection: "orders"},   actions: ["find", "insert", "update", "remove"]},
  {resource: {db: "shop", collection: "products"}, actions: ["find"]}]})
shop.createUser({user: "shop_app", pwd: passwordPrompt(), roles: ["shopService"]})
db.getSiblingDB("admin").createUser({user: "reporter", pwd: passwordPrompt(), roles: [{role: "read", db: "shop"}]})

sh.shardCollection("shop.orders", {customerId: 1, placedAt: 1})
shop.orders.createIndex({status: 1, placedAt: -1})
```

### Mistake 1: a secrets file that changed every time it was read

The scripted version of this step generated passwords into a `secrets.env` file — written as
`ADMIN_PW="cap-admin-$(openssl rand -hex 6)"`. Every time the file was sourced, the command substitution ran
again and produced a *new* password. The users were created with one set of values; the next script read another.
Nobody could log in, and the localhost exception was gone because a user existed.

The way back in was the keyFile. Members authenticate to each other as the internal `__system` user, whose
password is the keyFile's contents — so anyone holding the keyFile can authenticate as `__system` directly. Users
created through `mongos` live on the config servers, so that's where to connect:

```bash
mongosh "mongodb://127.0.0.1:27901,127.0.0.1:27902,127.0.0.1:27903/?replicaSet=cfg&tls=true&tlsCAFile=ca.pem" \
  -u __system -p "$(tr -d '\n' < keyfile)" --authenticationDatabase local
```

```text
passwords reset as __system on the config server
clusterAdmin login ok: clusterAdmin
```

(Trying it through `mongos` first failed with `Can't use 'local' database through mongos`.) Two lessons. The fixed
secrets file contains literal values, generated once. And **the keyFile is equivalent to root on every member**:
protect it like the most privileged password you have — file permissions, a secrets manager, never in images or
repositories. That's one reason production clusters prefer x.509 member certificates.

## Step 5: distribute the data

A new range-sharded collection starts as one chunk on the database's primary shard, and with the default 128 MB
chunk size the balancer won't move a small collection. Pre-splitting puts half the customers on each shard before
loading:

```javascript
sh.splitAt("shop.orders", {customerId: 10000, placedAt: MinKey})
db.adminCommand({moveRange: "shop.orders", min: {customerId: 10000, placedAt: MinKey}, toShard: "shardB"})
```

```text
primary shard of shop: shardA
moveRange to shardB 1
shardA {} -> 10000
shardB 10000 -> {}
```

(`{}` is how `JSON.stringify` prints `MinKey`/`MaxKey`.) Pre-splitting is standard practice before a bulk load
into a range-sharded collection; otherwise one shard takes the whole load and the balancer moves it later.

## Step 6: the acceptance script

`verify.py` connects as each identity and checks every requirement. Its building blocks:

```python
CA = os.path.abspath("ca.pem")
def client(user=None, pw=None, source="admin", tls=True, **kw):
    auth = dict(username=user, password=pw, authSource=source) if user else {}
    return MongoClient("mongodb://127.0.0.1:27900", tls=tls, tlsCAFile=CA if tls else None,
                       serverSelectionTimeoutMS=3000, **auth, **kw)

def denied(fn):
    try: fn(); return False, "allowed"
    except OperationFailure as e: return e.code == 13, e.details.get("codeName")   # 13 = Unauthorized
```

The checks, in order: plain-text connection refused; unauthenticated read refused; the app user inserts 60,000
orders; the app user is denied four things its role doesn't grant; the reporter reads but can't write; the orders
are on both shards; a customer query is `SINGLE_SHARD`; and finally a writer runs as the app user while the
`shardA` primary is killed with `SIGKILL`:

```python
prim = shard_a.admin.command("hello")["primary"].split(":")[1]
pid = subprocess.check_output(["pgrep", "-f", f"a{int(prim) - 27910}.conf"]).decode().split()[0]
os.kill(int(pid), signal.SIGKILL); time.sleep(15)
```

### Mistake 2: cluster users don't exist on the shards

The first run:

The first eleven checks passed; the last one didn't:

```text
     (could not ask shardA for its primary: AuthenticationFailed )
FAIL  shardA primary killed: no write errors  (53 inserts, slowest 0.12s, errors [])
```

To find the shard's primary, the script connected **directly to the shard** as `clusterAdmin` — and failed.
Users created through `mongos` are stored on the config servers; a shard has its own, separate user database,
which was empty. Which also meant something worse:

```text
shardA before: Unauthorized
shardA localhost exception still open? yes - created shard-local clusterAdmin
shardB before: Unauthorized
shardB localhost exception still open? yes - created shard-local clusterAdmin
```

Each shard's **localhost exception was still open**, because no user had ever been created on it. Anyone with a
shell on a shard host could have created a root user there. The fix is part of the build, not an afterthought:
right after initiating each shard replica set, create a shard-local administrator (used for maintenance tasks that
must connect to a shard directly — checking replication, stepping down a primary, compacting). That closes the
exception on every shard.

### The final run

```text
PASS  plain-text connection refused
PASS  unauthenticated read refused  (Unauthorized)
PASS  app user inserts 60,000 orders  (2.1s)
PASS  app user cannot drop collection  (Unauthorized)
PASS  app user cannot create index  (Unauthorized)
PASS  app user cannot read admin.system.users  (Unauthorized)
PASS  app user cannot write products  (Unauthorized)
PASS  reporter can read
PASS  reporter cannot write  (Unauthorized)
PASS  orders on both shards  ({'shardB': 30056, 'shardA': 29944})
PASS  customer query targets one shard  (SINGLE_SHARD)
PASS  shardA primary killed: no write errors  (194 inserts, slowest 10.28s, errors [])

12/12 checks passed
```

The failover behaved exactly as Lesson 01 predicted, now through `mongos` and over TLS: the insert in flight
waited about 10 seconds for `shardA` to elect a new primary, the driver's retryable write completed it, and the
application saw no error. This writer only used customer ids below 10,000, so every write went to `shardA`; writes
routed to `shardB` would not have waited at all, since each shard elects independently — but that wasn't measured
here. (Exercise 4 below adds the check.)

Restarting the killed member with its config file brought it back as a secondary:

```text
restarting a1
27911=SECONDARY 27912=SECONDARY 27913=PRIMARY
orders via mongos: 60194
```

60,000 loaded plus 194 written during the failover test: nothing lost.

## Operating it

The runbook entries this cluster needs, each covered earlier in Level 4:

- **Backups** (Lesson 06): stop the balancer, take coordinated snapshots of every shard and the config servers, or
  use a tool that coordinates sharded backups. `mongodump` through `mongos` is not consistent across shards.
- **Monitoring** (Lesson 07): every member's `serverStatus`, replication lag per replica set, balancer state, and
  jumbo chunks.
- **Upgrades** (Lesson 09): balancer off; config servers, then shards, then routers, each rolling.
- **Certificate rotation**: issue new certificates before expiry (these expire in 30 days) and roll them member by
  member, the same way as an upgrade.
- **Users**: cluster users are managed through `mongos`; shard-local admins are separate and must be rotated too.

## How It Actually Works

Three authentication domains exist in this cluster. **Internal**: every `mongod` and `mongos` proves membership with
the keyFile (a SCRAM exchange as `__system`), which also grants full privileges — that's what lets routers and
shards perform any operation on each other's behalf. **Cluster users**: stored in the config servers'
`admin.system.users`; `mongos` authenticates clients against them and then forwards each operation to the shards
along with the authenticated user's identity and roles, which the shards trust because the request arrived over an
internally authenticated connection. **Shard-local users**: stored on each shard, used only for direct connections.

TLS wraps every connection before any of this. Each member presents its certificate as a server and, when
connecting to another member, as a client; the CA file decides which certificates are trusted. Data and operations
then flow exactly as in Lessons 03–04: `mongos` routes by the cached routing table, shards check shard versions,
and each shard replica set handles its own elections and replication independently — which is why killing the
`shardA` primary stalled only `shardA`'s writes for one election timeout.

## Common mistakes

- **Leaving shard localhost exceptions open** by never creating shard-local users.
- **Treating the keyFile as a configuration file** rather than a root credential.
- **Secrets generated on read** (command substitution in an env file) or passwords committed to a repository.
- **Certificates without `clientAuth`** for members, or naming the wrong hosts.
- **Skipping pre-splitting** before a bulk load into a range-sharded collection.
- **Single-member shards or config servers** in production — a sharded cluster is only as available as its least
  available replica set.
- **Accepting a cluster without testing it.** Every guarantee here was assumed until the script checked it — and
  two were wrong.

## Exercise

1. Rebuild the cluster from scratch with a script, including shard-local administrators created immediately after
   each `rs.initiate`. Run `verify.py` and get 12/12 on the first try.
2. Add a third shard `shardC` with three members while the writer runs. Watch the balancer move data to it and
   record how long it takes to even out.
3. Replace keyFile authentication with x.509 member certificates (`clusterAuthMode: x509`), one certificate per
   member, using a rolling procedure (`sendKeyFile` → `sendX509` → `x509`). Confirm `__system` login with the old
   keyFile no longer works.
4. Extend `verify.py` with four more checks: a writer on `shardB` customers sees no delay while `shardA` fails over,
   a config server member killed during writes, the balancer state is `full`, and every certificate expires more
   than 7 days from now.
5. Write the restore drill: stop the balancer, dump each shard and the config servers, destroy the cluster,
   rebuild it from the dumps on fresh data directories, and prove the order count matches.
