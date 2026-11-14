# SQL vs NoSQL: Choosing the Right Database

## The real question

Start with Postgres unless a specific access pattern, scale or consistency need says otherwise. "SQL vs NoSQL" is not one choice: NoSQL covers four very different data models. The useful question is which model fits your access patterns and consistency needs.

## SQL (relational)

Examples: Postgres, MySQL, Oracle, SQL Server.

- **Model:** tables with a fixed schema, linked by foreign keys.
- **Design approach:** model the data's natural shape (normalize), then query it any way with JOINs.
- **Transactions:** strong ACID. A transfer debits and credits together, or not at all.
- **Scaling:** traditionally vertical (bigger machine) plus read replicas. Sharding is possible but painful.

## NoSQL families

NoSQL designs around your queries, not your data. It often denormalizes and duplicates on purpose so each read hits one place.

| Family | Examples | Model | Sweet spot |
| --- | --- | --- | --- |
| Key-value | Redis, DynamoDB | key → blob | Caching, sessions, counters, rate limits |
| Document | MongoDB, Firestore | JSON-like documents | Flexible or nested entities read as a whole |
| Wide-column | Cassandra, ScyllaDB, HBase | Partition key + sorted rows | Huge write volume, time-series, logs |
| Graph | Neo4j, Neptune | Nodes + edges | Deep relationship traversal (friends-of-friends, fraud rings) |

## The real tradeoffs

1. **Query flexibility vs. speed on known queries.** SQL answers questions you never planned for. Cassandra or DynamoDB are very fast on the query a table was designed for, and nearly useless for others.
2. **Consistency vs. availability (CAP).** During a network partition you choose one. Postgres picks consistency. Cassandra and Dynamo-style stores lean toward availability with eventual consistency, often tunable (e.g. QUORUM reads and writes).
3. **Horizontal scale.** NoSQL was built to shard from day one, the main reason it exists. Distributed SQL (CockroachDB, Spanner, YugabyteDB) and sharding layers (Citus, Vitess) blur this line.
4. **Schema.** A fixed schema is a safety net that blocks bad writes. "Schemaless" moves the schema into application code: fast to iterate, painful once nobody remembers which fields exist.

## Worked example: a Strava-like activity tracker

A real system usually uses several stores (polyglot persistence). Saying so in an interview signals seniority.

| Data | Store | Why |
| --- | --- | --- |
| Users, follows, clubs, kudos, subscriptions | Postgres | Relational, needs integrity, moderate volume |
| GPS points (~1 per second per activity) | Cassandra, or TimescaleDB | Write-heavy, append-only, always read as "all points for activity X in order"; partition key `activity_id`, clustering key `timestamp` |
| Activity feed | Redis | Precomputed feeds per user (fan-out on write) |
| Segment leaderboards | Redis sorted sets | `ZADD` / `ZREVRANGE` give ranked reads cheaply |
| Route search ("runs near me") | Postgres + PostGIS, or Elasticsearch | Geo queries |

For a first Play Store launch, Postgres + PostGIS alone goes a long way. Add Cassandra only when a single node actually struggles.

## How to answer in an interview

Never open with "NoSQL because it scales." Walk through these five questions, then pick:

1. **Entities and relationships?** Heavily relational → SQL.
2. **Access patterns?** A few known patterns at huge scale → NoSQL. Ad-hoc queries and reporting → SQL.
3. **Read/write ratio and volume?** Millions of writes per second → wide-column.
4. **Consistency needs?** Money or inventory → strong. Likes and view counts → eventual is fine.
5. **Pick, and name the tradeoff you accept.**

Default line: *"I'd start with Postgres unless there's a specific reason not to. Here the reason is X, so for this part I'd use Y."*

## Next topics to study

- [ ] How Cassandra works: partitioning, consistent hashing, tunable consistency
- [ ] Scaling SQL: replication, sharding strategies, cross-shard joins, resharding
- [ ] Isolation levels and transactions: dirty reads, phantom reads, MVCC
- [ ] Mock design: choose databases for WhatsApp or Uber
