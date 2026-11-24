---
title: "Free MongoDB Course: Documents, CRUD, Data Modeling, Indexes, Aggregation, Transactions, Replica Sets, Sharding and Operations"
description: "Free MongoDB course: 36 lessons and 4 projects run on MongoDB 9.0 — BSON, CRUD, schema design, indexes, aggregation, transactions, replication, sharding, security."
---

# MongoDB Mastery Path

MongoDB is easy to start with and easy to misuse. You can store a JSON-like document in minutes, and many teams
do — then discover months later that a list embedded in one document is approaching the 16 MB limit, that a
counter loses increments under load, that a "successful" restore restored nothing, or that writes the database
acknowledged were quietly rolled back after a failover. None of those are bugs in MongoDB. They are the
consequences of decisions somebody made without knowing they were making them.

This course teaches MongoDB by making those decisions visible. Every lesson runs against a real server and
measures what actually happens: how a document is encoded, how a query is planned, what an index costs, what a
transaction retries, which writes survive which failures. It assumes you can program in some language and have
used a database before; it doesn't assume any MongoDB experience.

**Level 1** covers the foundations: a throwaway local lab and what lives in its data directory, `mongosh`, BSON
and the types JSON doesn't have (and why `0.1 + 0.2` matters for prices), queries and update operators, arrays and
embedded documents, bulk writes, the Python and Node.js drivers, and the import/export and dump/restore tools —
ending with a tested data layer for a recipe app. **Level 2** is data modeling and indexes: embed or reference,
relationship patterns and schema design patterns, `$jsonSchema` validation, how B-tree indexes and compound key
order work, special index types, reading `explain()`, the query planner, and time series collections — ending with
a modeled, indexed store catalog. **Level 3** covers the aggregation pipeline and application features: grouping
and reshaping, `$lookup` joins, window functions, pipeline performance and `$merge` rollups, multi-document
transactions, change streams, geospatial queries and text search — ending with an order analytics backend kept
exact by a change-stream worker. **Level 4** is production: replica sets and failover, write and read concerns,
sharding and shard keys, security and TLS, backups and point-in-time recovery, monitoring, tuning and online
migrations — ending with a secured sharded cluster accepted by an automated test script.

Every lesson has a **How It Actually Works** section — why an `$inc` is recorded in the oplog as an absolute value,
why a compound index's field order can mean 71,514 keys examined or 219, why a transaction under contention fails
instantly instead of waiting, why `w: 3` can be faster than `w: "majority"`, why a range of one shard-key value
can never be split, and how a former primary decides which of its writes to throw away.

!!! note "Versions and what was actually run"
    Every command, query and script in this course was run on **MongoDB 9.0.2** Community Server (the official
    macOS tarball), and every output shown is the output it produced — including the errors, which are some of
    the most useful parts. Supporting tools: **mongosh 2.13.0**, **MongoDB Database Tools 100.19.1**
    (`mongodump`, `mongorestore`, `mongostat`…), **PyMongo 4.18.3** on Python 3.14, and the **Node.js driver
    7.7.0**. Timings come from a laptop and show proportions, not benchmarks; where a lesson compares numbers,
    both sides were measured the same way on the same machine.

    Things that were not run — MongoDB Atlas and other managed services, Ops Manager, Enterprise-only features
    such as auditing, encryption at rest and LDAP, Atlas Search and Vector Search — are described where relevant,
    never shown with invented output.

## How the program is organized

| Level | Focus | Modules | What you can do after |
|-------|-------|---------|------------------------|
| **1 · MongoDB Foundations** | Server, mongosh, BSON, CRUD, arrays, updates, drivers, tools | 10 | Write correct application code against MongoDB |
| **2 · Data Modeling & Indexes** | Embedding, patterns, validation, indexes, explain, the planner | 10 | Design schemas and indexes that stay fast as data grows |
| **3 · Aggregation & Application Features** | Pipelines, joins, analytics, transactions, change streams, geo, text | 10 | Build reporting, real-time and transactional features |
| **4 · Operations & Production** | Replication, concerns, sharding, security, backups, monitoring, tuning | 10 | Run MongoDB in production and recover when things go wrong |

Each level has 9 teaching modules plus a hands-on project (module 10), which makes **36 lessons and 4 projects** in total.

## How to use this site

- **Run your own server.** Level 1 · 01 sets up a lab you can delete and rebuild in seconds. Later lessons start
  several servers on different ports; a laptop is ideal, a shared company database is not.
- **Reproduce the outputs.** Your ObjectIds, timings and sizes will differ; the behaviour should not. When
  something differs in kind, find out why — that is usually where the learning is.
- **Break things on purpose.** Kill a primary, run two writers at once, restore without the oplog. Most of what
  matters about databases only shows up under concurrency and failure.
- **Do the projects.** Each one combines its level into something you could adapt for real work, and each was
  tested end to end, failures included.

## More from the Mastery Path series

Free, structured, module-wise training across 101 other languages, platforms and disciplines — expand a category to browse:

<div class="mastery-tracks">
<details class="mastery-card">
<summary>Languages <span class="mastery-count">20</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/python-mastery-path/">🐍 Python</a>
  <a href="https://learnmasterypath.com/java-mastery-path/">☕ Java</a>
  <a href="https://learnmasterypath.com/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://learnmasterypath.com/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://learnmasterypath.com/csharp-mastery-path/">🔵 C#</a>
  <a href="https://learnmasterypath.com/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://learnmasterypath.com/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://learnmasterypath.com/c-mastery-path/">🇨 C</a>
  <a href="https://learnmasterypath.com/cpp-mastery-path/">➕ C++</a>
  <a href="https://learnmasterypath.com/go-mastery-path/">🐹 Go</a>
  <a href="https://learnmasterypath.com/rust-mastery-path/">🦀 Rust</a>
  <a href="https://learnmasterypath.com/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://learnmasterypath.com/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://learnmasterypath.com/php-mastery-path/">🐘 PHP</a>
  <a href="https://learnmasterypath.com/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://learnmasterypath.com/swift-mastery-path/">🐦 Swift</a>
  <a href="https://learnmasterypath.com/dart-mastery-path/">🎯 Dart</a>
  <a href="https://learnmasterypath.com/scala-mastery-path/">🔴 Scala</a>
  <a href="https://learnmasterypath.com/r-mastery-path/">📊 R</a>
  <a href="https://learnmasterypath.com/matlab-mastery-path/">🟧 MATLAB</a>
</div>
</details>
<details class="mastery-card">
<summary>Web Frameworks <span class="mastery-count">11</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/react-mastery-path/">⚛️ React</a>
  <a href="https://learnmasterypath.com/nextjs-mastery-path/">▲ Next.js</a>
  <a href="https://learnmasterypath.com/angular-mastery-path/">🅰️ Angular</a>
  <a href="https://learnmasterypath.com/vue-mastery-path/">💚 Vue.js</a>
  <a href="https://learnmasterypath.com/html-css-mastery-path/">🎨 HTML & CSS</a>
  <a href="https://learnmasterypath.com/tailwind-mastery-path/">🌬️ Tailwind CSS</a>
  <a href="https://learnmasterypath.com/django-mastery-path/">🎸 Django</a>
  <a href="https://learnmasterypath.com/fastapi-mastery-path/">🚀 FastAPI</a>
  <a href="https://learnmasterypath.com/graphql-mastery-path/">🕸️ GraphQL</a>
  <a href="https://learnmasterypath.com/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://learnmasterypath.com/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
</details>
<details class="mastery-card">
<summary>Mobile <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/flutter-mastery-path/">💙 Flutter</a>
  <a href="https://learnmasterypath.com/react-native-mastery-path/">📲 React Native</a>
</div>
</details>
<details class="mastery-card">
<summary>Testing & QA <span class="mastery-count">6</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://learnmasterypath.com/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://learnmasterypath.com/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://learnmasterypath.com/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
  <a href="https://learnmasterypath.com/selenium-mastery-path/">🧭 Selenium</a>
  <a href="https://learnmasterypath.com/cypress-mastery-path/">🌲 Cypress</a>
</div>
</details>
<details class="mastery-card">
<summary>Security <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
  <a href="https://learnmasterypath.com/ethical-hacking-mastery-path/">🎯 Ethical Hacking</a>
</div>
</details>
<details class="mastery-card">
<summary>Cloud Platforms <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/aws-mastery-path/">☁️ AWS</a>
  <a href="https://learnmasterypath.com/azure-mastery-path/">☁️ Azure</a>
  <a href="https://learnmasterypath.com/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://learnmasterypath.com/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://learnmasterypath.com/adobe-mastery-path/">🎨 Adobe</a>
</div>
</details>
<details class="mastery-card">
<summary>Data & Analytics <span class="mastery-count">7</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://learnmasterypath.com/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://learnmasterypath.com/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://learnmasterypath.com/excel-mastery-path/">📗 Excel</a>
  <a href="https://learnmasterypath.com/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://learnmasterypath.com/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
  <a href="https://learnmasterypath.com/power-bi-mastery-path/">📉 Power BI</a>
</div>
</details>
<details class="mastery-card">
<summary>Databases <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/postgresql-mastery-path/">🐘 PostgreSQL</a>
  <a href="https://learnmasterypath.com/redis-mastery-path/">🟥 Redis</a>
  <a href="https://learnmasterypath.com/snowflake-mastery-path/">❄️ Snowflake</a>
  <a href="https://learnmasterypath.com/databricks-mastery-path/">🧱 Databricks</a>
  <a href="https://learnmasterypath.com/python-mastery-path/vector-databases-mastery-path/">🧭 Vector Databases</a>
</div>
</details>
<details class="mastery-card">
<summary>AI / ML / LLM <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://learnmasterypath.com/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://learnmasterypath.com/rag-mastery-path/">📚 RAG</a>
  <a href="https://learnmasterypath.com/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://learnmasterypath.com/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://learnmasterypath.com/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://learnmasterypath.com/ml-math-mastery-path/">➗ ML Math Foundations</a>
  <a href="https://learnmasterypath.com/prompt-engineering-mastery-path/">✍️ Prompt Engineering</a>
  <a href="https://learnmasterypath.com/ai-agents-mastery-path/">🕹️ AI Agents</a>
  <a href="https://learnmasterypath.com/mlops-mastery-path/">🔁 MLOps</a>
  <a href="https://learnmasterypath.com/computer-vision-mastery-path/">👁️ Computer Vision</a>
  <a href="https://learnmasterypath.com/python-mastery-path/mcp-mastery-path/">🔌 MCP</a>
</div>
</details>
<details class="mastery-card">
<summary>Embedded Systems <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://learnmasterypath.com/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://learnmasterypath.com/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://learnmasterypath.com/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://learnmasterypath.com/s32k-mastery-path/">🔧 S32K</a>
</div>
</details>
<details class="mastery-card">
<summary>Leadership & Management <span class="mastery-count">5</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://learnmasterypath.com/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://learnmasterypath.com/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://learnmasterypath.com/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://learnmasterypath.com/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
</details>
<details class="mastery-card">
<summary>Professional Skills <span class="mastery-count">4</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://learnmasterypath.com/workday-mastery-path/">🧑‍💼 Workday</a>
  <a href="https://learnmasterypath.com/digital-marketing-mastery-path/">📣 Digital Marketing & SEO</a>
  <a href="https://learnmasterypath.com/tally-gst-mastery-path/">🧾 Tally Prime & GST</a>
</div>
</details>
<details class="mastery-card">
<summary>Careers & Interviews <span class="mastery-count">2</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://learnmasterypath.com/system-design-mastery-path/">🏗️ System Design</a>
</div>
</details>
<details class="mastery-card">
<summary>Process & APIs <span class="mastery-count">3</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://learnmasterypath.com/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://learnmasterypath.com/playwright-mastery-path/">🎭 Playwright</a>
</div>
</details>
<details class="mastery-card">
<summary>Infrastructure & Ops <span class="mastery-count">12</span></summary>
<div class="mastery-grid">
  <a href="https://learnmasterypath.com/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://learnmasterypath.com/linux-mastery-path/">🐧 Linux</a>
  <a href="https://learnmasterypath.com/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://learnmasterypath.com/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://learnmasterypath.com/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://learnmasterypath.com/docker-mastery-path/">🐳 Docker</a>
  <a href="https://learnmasterypath.com/github-mastery-path/">🐙 GitHub & Git</a>
  <a href="https://learnmasterypath.com/cicd-mastery-path/">🔁 CI/CD</a>
  <a href="https://learnmasterypath.com/ansible-mastery-path/">🤖 Ansible</a>
  <a href="https://learnmasterypath.com/kafka-mastery-path/">📨 Kafka</a>
  <a href="https://learnmasterypath.com/microservices-mastery-path/">🧩 Microservices</a>
  <a href="https://learnmasterypath.com/observability-mastery-path/">📈 Observability</a>
</div>
</details>
</div>
