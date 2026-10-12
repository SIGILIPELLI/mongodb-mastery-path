---
description: "MongoDB Level 4: replica sets, write and read concerns, sharding, shard keys, security, backups and PITR, monitoring, tuning, upgrades, capstone."
---

# Level 4 · Operations & Production

Levels 1–3 were about using MongoDB well. Level 4 is about running it: keeping data safe when machines fail,
choosing the guarantees each operation needs, scaling past one machine, locking the doors, recovering from
mistakes, and changing a live system without stopping it.

Every lesson breaks something on purpose and measures what the application saw. You'll kill a primary in the
middle of a stream of writes and find the driver hid it completely — then turn off retryable writes and watch the
same failure surface as an error. You'll force a rollback and find five acknowledged writes in a BSON file in the
data directory. You'll see `w: 3` beat `w: "majority"` (and learn why that's not the good news it looks like), a
restore come back 42 units short of a bank's total because it skipped the oplog, a sharding balancer stop with the
cluster "unbalanced" because the donor's counts included orphans, and a capstone acceptance script fail twice
before passing — once because cluster users don't exist on shards, which had left every shard's localhost
exception open.

**Prerequisites:** Levels 1–3. These lessons run several `mongod` processes at once (up to ten in the capstone),
so close other labs and cap each server's cache with `--wiredTigerCacheSizeGB 0.25`. Python with PyMongo is used
for anything involving concurrency or timing; OpenSSL for certificates.

## Modules

1. **[Replica Sets: Elections, the Oplog & Failover](01-replica-sets.md)** — a three-node set, oplog entries
   read directly, step-down vs kill -9, retryable writes, and a real rollback.
2. **[Write Concern, Read Concern & Read Preference](02-read-write-concerns.md)** — the price of each write
   concern, stale secondaries, causal sessions, and `majority` versus `local` reads.
3. **[Sharding Architecture](03-sharding-architecture.md)** — `mongos`, config servers and shards; the routing
   table; targeted and broadcast queries; the balancer and orphans.
4. **[Choosing & Changing a Shard Key](04-shard-keys.md)** — cardinality, frequency, monotonicity; indivisible
   chunks; unique indexes; `reshardCollection` and `refineCollectionShardKey`.
5. **[Authentication, Roles & TLS](05-security.md)** — the localhost exception, built-in and custom roles, TLS
   and mutual TLS, x.509 users, and keyFile member authentication.
6. **[Backup, Restore & Point-in-Time Recovery](06-backup-restore.md)** — consistent dumps under load, recovery
   to the operation before a bad `deleteMany`, and file-level copies.
7. **[Monitoring & the Profiler](07-monitoring-profiler.md)** — `serverStatus`, `mongostat`, `mongotop`, the
   slow-query log, the profiler, and killing runaway operations.
8. **[Performance Tuning: WiredTiger, Memory & Connections](08-performance-tuning.md)** — cache sizing and the
   working set, compression trade-offs, connection pools, and OS settings.
9. **[Upgrades, FCV & Online Schema Migrations](09-upgrades-migrations.md)** — rolling restarts under load,
   featureCompatibilityVersion, online index builds, and idempotent batched migrations.
10. **[Capstone — A Secured Sharded Cluster](10-capstone-sharded-cluster.md)** — ten processes with TLS, keyFile
    and least-privilege roles, accepted by a 12-check script including a live failover.

## What wasn't run

Managed services (MongoDB Atlas), Ops Manager and Cloud Manager, and Enterprise-only features (auditing,
encryption at rest, LDAP and Kerberos authentication) aren't part of the Community server used here, so they are
described where relevant but never shown with output. Only one server version was installed, so the version
upgrade lesson performs the rolling procedure and FCV changes for real but not a binary swap between releases.

## After Level 4

You've finished the course. The best next step is to take the capstone's acceptance-script idea back to a real
system: write down what your MongoDB deployment is supposed to guarantee — durability, failover, access, recovery
time — and write a test for each one. Related courses in the series:
[System Design](https://sigilipelli.github.io/system-design-mastery-path/) for where MongoDB fits in larger
architectures, [Kubernetes](https://sigilipelli.github.io/kubernetes-mastery-path/) and
[Docker](https://sigilipelli.github.io/docker-mastery-path/) for running it in containers, and
[PostgreSQL](https://sigilipelli.github.io/postgresql-mastery-path/) for the relational side of the same problems.
