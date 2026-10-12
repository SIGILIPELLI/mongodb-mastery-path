---
description: "Install MongoDB from the official tarball, start a throwaway mongod, and learn what lives in its data directory, logs and WiredTiger cache."
---

# 01 · How MongoDB Is Built: mongod, Storage & a Local Lab

Before you write a single query it pays to know what you are talking to. MongoDB is not a library
your program links against and it is not a file you open — it is a **server process** called
`mongod` that owns a directory of files, listens on a TCP port, and answers commands sent to it in a
binary wire protocol. Every tool you will use in this course — the `mongosh` shell, PyMongo, the
Node.js driver, `mongodump` — is a *client* that opens a connection to that process and sends it
commands.

This lesson builds a lab you can destroy and rebuild in seconds, and then looks inside it.

## The pieces

| Program | What it is | Used in |
|---------|------------|---------|
| `mongod` | The database server. Stores data, runs queries, replicates. | every lesson |
| `mongos` | A router that sits in front of a *sharded* cluster and forwards queries to the right shards. | Level 4 |
| `mongosh` | The interactive JavaScript shell. Shipped separately from the server. | every lesson |
| Database tools | `mongodump`, `mongorestore`, `mongoimport`, `mongoexport`, `mongostat`, `mongotop`, `bsondump`. Also shipped separately, with their own version numbers (100.x). | Level 1 · 09, Level 4 |
| Drivers | Libraries for each language (PyMongo, the Node.js `mongodb` package, Java, Go, C#…). | Level 1 · 08 onwards |

The server, the shell and the tools are versioned independently. The versions used to produce every
output in this course are listed on the [home page](../index.md).

## Getting the binaries

There are three common ways to get a server onto a development machine:

1. **Your OS package manager** — MongoDB publishes its own repositories for the major Linux
   distributions and a Homebrew tap for macOS. (The server is not in Homebrew's core repository
   because of its license; the shell `mongosh` is, so `brew install mongosh` works directly.)
2. **A container** — `docker run -d -p 27017:27017 mongo:<version>` with the official image. See the
   [Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/) if containers are new.
3. **The official tarball** from `fastdl.mongodb.org` — nothing to install, just unpack and run.

This course uses option 3 because it leaves nothing behind. The download page publishes a SHA-256
checksum next to each archive; always compare it:

```bash
curl -sSLO https://fastdl.mongodb.org/osx/mongodb-macos-arm64-9.0.2.tgz
curl -sSLO https://fastdl.mongodb.org/osx/mongodb-macos-arm64-9.0.2.tgz.sha256
cat mongodb-macos-arm64-9.0.2.tgz.sha256
shasum -a 256 mongodb-macos-arm64-9.0.2.tgz
tar xzf mongodb-macos-arm64-9.0.2.tgz
ls mongodb-macos-aarch64--9.0.2/bin
```

```text
9a30e942877445dc143d83671eba6cda2ec93a99160c8a8de57595853dc76de3  mongodb-macos-arm64-9.0.2.tgz
9a30e942877445dc143d83671eba6cda2ec93a99160c8a8de57595853dc76de3  mongodb-macos-arm64-9.0.2.tgz
install_compass
mongod
mongos
```

The two hashes match, so the archive is what MongoDB published. Swap `osx`/`macos-arm64` for your
platform (Linux archives are named after the distribution, e.g. `ubuntu2404`). Notice the archive
contains only the server and the router — `mongosh` and the database tools are separate downloads.

## Starting a throwaway server

```bash
mkdir lab1
mongod --port 27100 --dbpath lab1 --bind_ip 127.0.0.1 --logpath lab1.log &
```

Each flag matters:

- `--dbpath` is the directory `mongod` will own. It must exist and must not be shared with another
  running `mongod`.
- `--port` defaults to 27017. Using a different port lets you run several servers side by side,
  which you will do constantly in Level 4.
- `--bind_ip 127.0.0.1` means only processes on this machine can connect. Since MongoDB 3.6 the
  default is already localhost-only, but writing it explicitly makes the intent obvious. Binding to
  `0.0.0.0` on a machine with a public address and no authentication is how many thousands of
  databases have been emptied by automated scanners.
- `--logpath` sends the log to a file instead of the terminal.

If you are on macOS and reach for `--fork` (which daemonizes the process on Linux), this server
version refuses:

```text
BadValue: Server fork+exec via `--fork` or `processManagement.fork` is incompatible with macOS
try 'mongod --help' for more information
```

Run it in the background with your shell (`&`, `nohup`) or under a process manager (`launchd`,
`systemd`) instead. On Linux in production you would normally let `systemd` manage `mongod` and
leave `fork` off anyway.

Connect and confirm it answers:

```bash
mongosh --port 27100 --quiet --eval 'print(db.version())'
```

```text
9.0.2
```

## What's in the data directory

After inserting one document into a `shop.products` collection, the data directory looked like this:

```text
WiredTiger
WiredTiger.lock
WiredTiger.turtle
WiredTiger.wt
WiredTigerHS.wt
_mdb_catalog.wt
_tmp
collection-37409f2c-aed8-4003-8228-72c5b1010677.wt
collection-53ca67f9-bcf9-49d5-bb43-b0160f048f5c.wt
collection-577008e9-9fd2-4ab2-b218-a9abfcacabe3.wt
collection-6ebb04ca-b382-4752-b268-5a0e6e872948.wt
diagnostic.data
index-311b82c6-db79-4645-9170-04e30d948cf7.wt
index-35070c29-ef31-4d8c-93f7-6751d785721b.wt
index-a745ca20-aed7-4e00-af30-3f819d51a97a.wt
index-ab1ed0cf-8d97-4ba0-a4ab-4c1da99da13a.wt
index-dd646673-c3ea-4a74-8ded-3a924b7de804.wt
journal
mongod.lock
sizeStorer.wt
storage.bson
```

You can ask the server which file holds a collection:

```javascript
db.getSiblingDB("shop").products.stats().wiredTiger.uri
```

```text
statistics:table:collection-37409f2c-aed8-4003-8228-72c5b1010677
```

That is the new `collection-…wt` file that appeared after the insert, and the new `index-…wt` file
next to it is the mandatory `_id` index. The other files:

| File | Role |
|------|------|
| `collection-*.wt`, `index-*.wt` | One WiredTiger table per collection and one per index. |
| `_mdb_catalog.wt` | MongoDB's catalog: which namespaces (`db.collection`) exist and which files back them. |
| `WiredTiger.wt`, `WiredTiger.turtle` | The storage engine's own metadata. |
| `WiredTigerHS.wt` | The *history store*: older versions of documents kept so long-running reads still see a consistent snapshot. |
| `journal/` | The write-ahead log. Writes are recorded here before data files are updated, so a crash can be recovered. |
| `diagnostic.data/` | FTDC — "full-time diagnostic data capture", compressed server metrics sampled every second. Support engineers ask for it; Level 4 · 07 looks at it. |
| `mongod.lock` | Non-empty while a `mongod` is running on this directory. |

Do not edit, copy or delete files in a live data directory. The only safe ways to copy one are to
stop `mongod` cleanly first, or to use a filesystem snapshot that captures the journal at the same
instant (Level 4 · 06).

## Databases, collections and namespaces

Inside one `mongod` there are **databases**; inside a database, **collections**; inside a collection,
**documents**. None of them have to be created in advance: the first insert into `shop.products`
created both the `shop` database and the `products` collection.

```javascript
db.adminCommand({ listDatabases: 1 }).databases.map(d => d.name)
```

```text
[ 'admin', 'config', 'local', 'shop' ]
```

Three databases come with every server:

- `admin` — users with cluster-wide roles, and the target of administrative commands.
- `local` — never replicated. On a replica set member it holds the **oplog** (Level 4 · 01).
- `config` — sessions and, on sharded clusters, the metadata about where data lives.

Leave all three alone unless a lesson tells you otherwise.

## The log is JSON

Every log line is a JSON document, which makes it easy to filter with `jq` or `grep`:

```text
{"t":{"$date":"2026-10-11T22:52:53.729+05:30"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27100,"ssl":"off"}}
```

`s` is severity (`I` info, `W` warning, `E` error, `F` fatal, `D1`–`D5` debug), `c` the component,
`id` a stable numeric message identifier (searchable across versions even when the text changes) and
`attr` the structured details. A startup produced 132 lines on this machine; the one to look for is
`"Waiting for connections"`.

The server also keeps its startup warnings for you:

```javascript
db.adminCommand({ getLog: "startupWarnings" }).log.forEach(l => print(JSON.parse(l).msg))
```

```text
Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
```

On a laptop lab that is acceptable. Anywhere else it is not — Level 4 · 05 turns authentication on.

## A configuration file

Command-line flags are fine for experiments. Real deployments use a YAML file, which maps one-to-one
onto the flags. You can see the parsed options of a running server:

```javascript
db.adminCommand({ getCmdLineOpts: 1 }).parsed
```

```text
{
  net: { bindIp: '127.0.0.1', port: 27100 },
  storage: { dbPath: 'lab1' },
  systemLog: { destination: 'file', path: 'lab1.log' }
}
```

The equivalent `mongod.conf`:

```yaml
net:
  port: 27100
  bindIp: 127.0.0.1
storage:
  dbPath: /var/lib/mongodb
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

Start with `mongod --config mongod.conf` (or `-f`).

## Stopping cleanly

```javascript
db.adminCommand({ shutdown: 1 })
```

The shell reports that its connection closed — that is expected, the server went away. A clean
shutdown flushes everything to the data files and takes a final checkpoint. `kill -9` does not, but
thanks to the journal the next start replays recent writes and recovers; you will see that happen
in Level 4. Pressing <kbd>Ctrl</kbd>+<kbd>C</kbd> on a foreground `mongod`, or sending `SIGTERM`,
also triggers a clean shutdown.

## How It Actually Works

When `mongod` starts, it opens the data directory with the **WiredTiger** storage engine (the only
general-purpose engine since 4.2; `buildInfo` on this build listed `["devnull","wiredTiger"]`, and
`devnull` discards everything — it exists for testing). WiredTiger reads `WiredTiger.turtle` to find
its own metadata table, replays any journal records written after the last checkpoint, and then
MongoDB reads `_mdb_catalog.wt` to rebuild its list of collections and indexes.

From then on, all reads and writes go through WiredTiger's **cache** in RAM. Documents live in
B-tree pages; a read pulls the page into the cache if it isn't there; a write modifies the page in
cache and appends a record to the journal. Roughly every 60 seconds a **checkpoint** writes all dirty
pages to the data files as a consistent snapshot. If the process dies, the last checkpoint plus the
journal is enough to get back to the last acknowledged write (with the default journaled write
concern — Level 4 · 02 explains exactly what "acknowledged" means).

The cache size is the single most important memory setting. By default it is the larger of 256 MB or
half of (RAM − 1 GB):

```javascript
db.serverStatus().wiredTiger.cache["maximum bytes configured"]
```

```text
Long('3758096384')
```

On this 8 GiB machine (`sysctl -n hw.memsize` → `8589934592`) that is (8 − 1) / 2 = 3.5 GiB =
3,758,096,384 bytes, exactly the formula. The rest of the RAM is left to the operating system's file
cache (which holds compressed data-file pages), to connections, and to query execution. Running two
default-configured `mongod`s on one machine means each thinks it may use half the memory — fine for a
lab of tiny datasets, a real problem in production, which is why Level 4 sets
`--wiredTigerCacheSizeGB` explicitly for multi-process labs.

## Common mistakes

- **Pointing two servers at one `--dbpath`.** The second refuses to start because of `mongod.lock`;
  deleting the lock file to "fix" that while the first server is running corrupts data.
- **Exposing an unauthenticated server.** `bind_ip_all` or `0.0.0.0` without access control is the
  classic way MongoDB data leaks. Keep labs on `127.0.0.1`.
- **Mixing up client and server versions.** `mongosh --version` tells you the shell version, not the
  server's. Use `db.version()`.
- **Copying the data directory of a running server as a "backup".** Use the tools in Level 1 · 09 and
  Level 4 · 06.
- **Assuming databases must be created first.** They appear on first write — which also means a typo
  in a database name silently creates a new, empty database.

## Exercise

1. Download a server for your platform, verify its checksum, and start it on port 27100 with a fresh
   data directory and a log file.
2. Insert one document into `practice.notes`. Find which `.wt` file holds the collection using
   `stats().wiredTiger.uri` and confirm the file exists on disk.
3. Write a `mongod.conf` that reproduces your command-line flags, stop the server cleanly, and start
   it again with `--config`. Check `getCmdLineOpts` to confirm the file was used.
4. Find the `"Waiting for connections"` line in the log and note its `id`. Then compute the default
   WiredTiger cache size for your machine by hand and compare it with `serverStatus()`.
