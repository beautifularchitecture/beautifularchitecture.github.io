+++
date = '2026-06-25T12:00:00+10:00'
draft = false
title = 'Database Change Data Capture (CDC)'
tags = ['CDC', 'Change Data Capture', 'Debezium', 'Kafka', 'DMS', 'DynamoDB Streams', 'Postgres', 'Replication', 'Microservices']
summary = 'Deep dive into Change Data Capture — log-based, trigger-based and timestamp-based patterns, Debezium, AWS DMS, DynamoDB Streams, the outbox pattern, dual-write problem and exactly-once semantics.'
+++

Change Data Capture (CDC) is a pattern for capturing row-level changes in a database and streaming them to downstream systems in real time. CDC solves the data synchronisation problem between operational databases, caches, search indexes, data warehouses and microservices — without dual-write coordination. This post covers every CDC approach, tool, failure mode and production pattern you need to know.

---

## Why CDC Exists

### The Dual-Write Problem

When an application needs to update a database and publish an event to a message broker, the simplest approach is a dual write:

```python
def create_order(order):
    db.insert(order)                     # Write 1
    kafka.produce("orders", order)       # Write 2
```

**Failure scenarios:**

1. DB write succeeds, Kafka write fails → inconsistent state (event lost)
2. Kafka write succeeds, DB write fails → phantom event
3. Network partition: either side may commit, giving distributed inconsistency

Distributed transactions (XA) solve this but are expensive, slow and not supported by many modern brokers (Kafka, Pulsar).

CDC eliminates dual writes by making the database the single source of truth and deriving events directly from the database transaction log.

### When CDC Makes Sense

| Scenario                                               | CDC vs Direct Publish                                 |
| ------------------------------------------------------ | ----------------------------------------------------- |
| Microservices need consistent events                   | CDC — no dual-write risk                              |
| Populating a search index (Elasticsearch, Meilisearch) | CDC — avoids synchronisation code                     |
| Caching layer invalidation (Redis)                     | CDC — guaranteed delivery                             |
| Data lake ingestion (S3 + Athena)                      | CDC — captures deletes and updates                    |
| Audit logging                                          | CDC — captures all changes, even from bulk operations |
| Replicating to another database                        | CDC — lower latency than ETL batch                    |

---

## CDC Approaches

### Log-Based CDC (Best)

Also called "transaction log tailing" or "write-ahead log (WAL) capture."

**How it works:**

```
Database → WAL/Transaction Log → Log Reader → Change Events → Kafka / Pulsar / Stream
```

- The database writes every mutation to its transaction log before applying it
- A CDC connector reads the log sequentially at the storage level (no table scans)
- Minimal performance impact (sequential reads from the log, no triggers)
- Captures all changes including schema changes, DDL, bulk operations

**Database-specific implementations:**

| Database   | Log Source                                      | Connector Examples      |
| ---------- | ----------------------------------------------- | ----------------------- |
| PostgreSQL | WAL (pgoutput, wal2json, decoderbufs)           | Debezium, pglogical     |
| MySQL      | Binary log (binlog) — row-based format required | Debezium, Maxwell, DMS  |
| MariaDB    | Binary log                                      | Debezium                |
| SQL Server | Transaction Log (CDC tables via sys.fn_cdc)     | Debezium, DMS           |
| Oracle     | LogMiner / XStream / Redo Logs                  | Debezium, OGG, DMS      |
| MongoDB    | Oplog / Change Streams                          | Debezium, Kafka Connect |
| DynamoDB   | DynamoDB Streams (Kinesis-backed)               | Lambda, KCL             |
| Cosmos DB  | Change Feed                                     | Azure Functions         |

### Trigger-Based CDC

**How it works:**

- Create triggers (AFTER INSERT/UPDATE/DELETE) that write change records to a shadow table or emit events
- A background process polls the shadow table and publishes events

```sql
CREATE OR REPLACE FUNCTION capture_order_change() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO order_changes (order_id, old_data, new_data, operation, changed_at)
    VALUES (COALESCE(NEW.id, OLD.id), row_to_json(OLD), row_to_json(NEW), TG_OP, NOW());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_changes
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION capture_order_change();
```

**Pros:**

- Works with any database (even those without exposed transaction logs)
- No special permissions required

**Cons:**

- Significant performance impact (triggers execute in the same transaction)
- Does not capture schema changes, TRUNCATE, COPY or direct file-level modifications
- Triggers add latency to every write transaction
- Changes written by triggers are visible only after the outer transaction commits
- Managing triggers across schema migrations is error-prone

### Timestamp-Based CDC (Lowest Fidelity)

**How it works:**

- Tables include `updated_at` / `created_at` columns
- A poller periodically queries `WHERE updated_at > :last_seen`
- Under high throughput, use `updated_at, id` to avoid missing rows with identical timestamps

```sql
SELECT * FROM orders
WHERE (updated_at > :last_updated_at)
   OR (updated_at = :last_updated_at AND id > :last_id)
ORDER BY updated_at, id
LIMIT 1000;
```

**Limitations:**

- Does not capture **deletes** — you need soft deletes or a tombstone mechanism
- Misses changes from bulk operations that bypass `updated_at` (e.g. direct SQL, backfills)
- Polling latency (seconds to minutes depending on poll interval)
- Pagination is complex under heavy concurrent writes
- No schema change detection

### API-Based CDC (e.g. DynamoDB Streams)

Some databases expose changes as a native streaming API:

- **DynamoDB Streams** — ordered stream of item-level changes retained for 24 hours; for longer retention, route to Kinesis Data Streams (up to 365 days)
- **MongoDB Change Streams** — oplog-based, real-time, supports resume tokens, aggregation pipeline filtering
- **Cosmos DB Change Feed** — sorted by modification time, persisted in a container

These are specialised forms of log-based CDC but surfaced through a higher-level API.

---

## Debezium (The Reference Implementation)

Debezium is an open-source distributed CDC platform built on Kafka Connect. It is the de-facto standard for log-based CDC. The current stable release is **3.4.x** (built against Kafka Connect 4.1.1). PostgreSQL 13 support was dropped as of Debezium 3.4.

### Deployment Modes

Debezium supports two primary deployment models:

**1. Kafka Connect (standalone cluster)**

The traditional architecture: Debezium runs as a Kafka Connect source connector on a dedicated Connect worker cluster. This is the right choice for high-throughput, multi-table, production pipelines.

```
Source DB → Debezium Connector (Kafka Connect worker) → Kafka Topic → Sink Connector (Elasticsearch, S3, etc.)
```

**2. Embedded via Quarkus Extension (introduced in 3.4)**

Debezium 3.4 introduced a Quarkus DevService extension that embeds CDC directly inside a Quarkus application — no separate Kafka Connect cluster required. Supported connectors: PostgreSQL, MySQL, MariaDB, MongoDB, SQL Server.

```xml
<dependency>
    <groupId>io.debezium.quarkus</groupId>
    <artifactId>quarkus-debezium-postgres</artifactId>
    <version>3.4.3.Final</version>
</dependency>
```

```properties
# application.properties
quarkus.debezium.name=orders-app
quarkus.debezium.topic.prefix=prod-db
quarkus.debezium.offset.storage=org.apache.kafka.connect.storage.FileOffsetBackingStore
```

This is well-suited for microservices that need to react to their own database changes locally, or for lower-throughput use cases where running a full Kafka Connect cluster is not justified.

### Kafka Connect Architecture

Each Debezium connector is a Kafka Connect source connector that:

1. Connects to the database as a replica client
2. Reads the transaction log from a given offset (LSN for Postgres, binlog position for MySQL)
3. Converts raw log events into structured change records (Avro, JSON, Protobuf)
4. Emits records to a Kafka topic (one topic per table)
5. Commits offsets to Kafka so the connector can resume on restart

### Change Event Structure (PostgreSQL example)

```json
{
  "schema": {
    "type": "struct",
    "fields": [
      { "field": "before", "type": "struct", "optional": true },
      { "field": "after", "type": "struct", "optional": true },
      { "field": "source", "type": "struct" },
      { "field": "op", "type": "string" },
      { "field": "ts_ms", "type": "int64" }
    ]
  },
  "payload": {
    "before": null,
    "after": {
      "id": 42,
      "customer_name": "Acme Corp",
      "status": "SHIPPED",
      "total": 299.99
    },
    "source": {
      "version": "3.4.3.Final",
      "connector": "postgresql",
      "name": "prod-db",
      "ts_ms": 1750000000000,
      "snapshot": "false",
      "db": "orders_db",
      "sequence": "1234567:8765432",
      "schema": "public",
      "table": "orders",
      "txId": 98765,
      "lsn": 123456789,
      "xmin": null
    },
    "op": "c",
    "ts_ms": 1750000000001,
    "transaction": null
  }
}
```

**`op` field values:**

- `c` — Create (INSERT)
- `r` — Read (initial snapshot)
- `u` — Update
- `d` — Delete
- `t` — Truncate

### Critical Configuration Parameters

```json
{
  "name": "orders-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-primary",
    "database.port": 5432,
    "database.user": "debezium",
    "database.password": "${POSTGRES_PASSWORD}",
    "database.dbname": "orders_db",
    "topic.prefix": "prod-db",
    "table.include.list": "public.orders,public.order_items",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_orders_slot",
    "publication.name": "debezium_orders_pub",
    "publication.autocreate.mode": "filtered",
    "snapshot.mode": "initial",
    "tombstones.on.delete": "true",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081"
  }
}
```

| Parameter                     | Purpose                                                                                                 |
| ----------------------------- | ------------------------------------------------------------------------------------------------------- |
| `plugin.name`                 | The logical decoding plugin — `pgoutput` (default PG 14+), `wal2json`, `decoderbufs`                    |
| `slot.name`                   | The Postgres replication slot name — each connector needs a unique slot                                 |
| `snapshot.mode`               | `initial` (snapshot then stream), `never` (stream only), `always` (resnapshot), `no_data` (schema only) |
| `tombstones.on.delete`        | Emit a tombstone (null value) after a delete event for log compaction                                   |
| `publication.autocreate.mode` | `all_tables`, `filtered`, `disabled`                                                                    |
| `heartbeat.interval.ms`       | Send heartbeats on idle connections to prevent replication slot lag                                     |

### PostgreSQL Replication Slot Management

Debezium uses PostgreSQL logical replication slots. Critical operational knowledge:

- **Each slot tracks a WAL position** — the server retains WAL segments until the slot consumer acknowledges them
- **If Debezium is down for too long**, WAL accumulates and can fill disk (WAL explosion)
- **Monitor `pg_replication_slots`**:
  ```sql
  SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn
  FROM pg_replication_slots;
  ```
- **Set `max_slot_wal_keep_size`** (PG 13+) to limit WAL retention per slot
- **Drop dangling slots immediately** if a connector is permanently removed:
  ```sql
  SELECT pg_drop_replication_slot('debezium_orders_slot');
  ```

### Schema Evolution and Migration

Debezium captures schema changes via the `schema_only` recovery mode or by connecting to a database schema registry. For Avro serialisation:

- Each change event includes both the schema and the payload
- Schema Registry stores a versioned history
- When a column is added/dropped, Schema Registry evolves the schema (BACKWARD, FORWARD, FULL, NONE compatibility)
- Downstream consumers recompile with the new Avro schema or use Schema Registry at runtime

### Failure Modes

| Failure                              | Impact                                                         | Mitigation                                                    |
| ------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------------------- |
| Source DB restarts                   | Connector resumes from last committed offset                   | Built-in (Kafka Connect source offset)                        |
| Kafka unavailable                    | Logs accumulate in Debezium's buffer; WAL grows                | Set `max.request.size`; monitor WAL size                      |
| Schema change breaks deserialisation | Connector fails; events cannot be parsed                       | Use Schema Registry with `schema.compatibility`               |
| Network partition                    | Connector cannot read WAL; WAL accumulation                    | Set `heartbeat.interval.ms`; use `wal2json` streaming         |
| Connector OOM                        | JVM heap exhaustion                                            | Tune `offset.flush.timeout.ms`, `max.batch.size`              |
| Corrupted offset topic               | Connector loses position → resnapshot or re-stream from latest | Periodic offset backups; use Kafka with `min.insync.replicas` |

---

## AWS DMS CDC

AWS Database Migration Service supports CDC as a continuous replication task.

### DMS CDC Architecture

```
Source DB → DMS Replication Instance (EC2) → Target DB / S3 / Kinesis
```

- DMS performs a **full load** first (snapshot), then switches to **ongoing replication** (CDC)
- CDC reads MySQL binlog, Postgres WAL, Oracle Redo, SQL Server transaction logs
- Target can be another database, S3 (Parquet/CSV), Kinesis stream, Kafka or Redshift

### Key Configuration Decisions

**Endpoint settings for Postgres CDC:**

```json
{
  "EndpointIdentifier": "source-postgres",
  "EngineName": "postgres",
  "PostgreSQLSettings": {
    "DatabaseName": "orders_db",
    "SlotName": "dms_slot",
    "PluginName": "pglogical",
    "FailTasksOnTruncationOfDDL": true
  }
}
```

**Task settings (`cdc_task.json`):**

```json
{
  "TargetMetadata": {
    "TargetSchema": "",
    "SupportLobs": true
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "DO_NOTHING",
    "CreatePkAfterFullLoad": true
  },
  "Logging": {
    "EnableLogging": true
  }
}
```

### DMS Limitations vs Debezium

| Aspect           | DMS                                      | Debezium + Kafka              |
| ---------------- | ---------------------------------------- | ----------------------------- |
| Event schema     | Fixed — DMS normalises column types      | Custom Avro/JSON/Protobuf     |
| Target options   | DB, S3, Kinesis, Kafka, Redshift         | Any Kafka sink                |
| Schema evolution | Limited (must stop task, change mapping) | Schema Registry handles       |
| Latency          | 1–10s typically                          | Sub-second                    |
| Cost             | Pay per replication instance + storage   | Pay for Kafka + Kafka Connect |
| LOB support      | Up to 32 MB (configurable)               | No special handling needed    |

DMS is best when you need a managed, no-code replication to another database or S3. Debezium is better when you need fine-grained event-driven architecture with Kafka.

---

## DynamoDB Streams

DynamoDB Streams is AWS's native CDC for DynamoDB. It captures a time-ordered sequence of item-level changes.

### Architecture

```
DynamoDB Table → DynamoDB Stream → AWS Lambda / KCL Consumer → Downstream
```

- A stream is enabled per table (NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES, KEYS_ONLY)
- **Data is retained for exactly 24 hours** — this is a hard limit with no configuration option to extend it
- For retention beyond 24 hours, route changes to **Kinesis Data Streams** (a separate service), which supports retention up to 365 days and integrates with Kinesis Data Firehose and Kinesis Data Analytics
- Each stream is composed of shards:
  - Shards are automatically created/removed based on table throughput
  - Ordering is guaranteed within a shard but not across shards
  - Shards are hashed by partition key

> **KCL note:** DynamoDB Streams added support for Kinesis Client Library (KCL) 3.0 in June 2025, reducing compute costs by up to 33% compared to earlier KCL versions. KCL 1.x reached end-of-support on January 30, 2026 — migrate to KCL 3.x.

### Stream Record Structure

```json
{
  "eventID": "1",
  "eventVersion": "1.0",
  "eventSource": "aws:dynamodb",
  "awsRegion": "us-east-1",
  "eventName": "INSERT",
  "eventSourceARN": "arn:aws:dynamodb:...",
  "dynamodb": {
    "ApproximateCreationDateTime": 1750000000,
    "Keys": { "order_id": { "S": "ORD-001" } },
    "NewImage": {
      "order_id": { "S": "ORD-001" },
      "status": { "S": "PENDING" },
      "total": { "N": "299.99" }
    },
    "OldImage": null,
    "SequenceNumber": "100000000001",
    "SizeBytes": 128,
    "StreamViewType": "NEW_AND_OLD_IMAGES"
  }
}
```

### Lambda as Consumer

```python
import boto3
import json

def handler(event, context):
    for record in event['Records']:
        ddb = record['dynamodb']
        event_name = record['eventName']   # INSERT, MODIFY, REMOVE

        if event_name == 'INSERT':
            process_new_order(ddb['NewImage'])
        elif event_name == 'MODIFY':
            process_order_update(ddb['OldImage'], ddb['NewImage'])
        elif event_name == 'REMOVE':
            process_order_cancellation(ddb['OldImage'])

        if record.get('eventSourceARN'):
            log_processed(record['eventSourceARN'],
                          ddb['SequenceNumber'])
```

### Exactly-Once Semantics with DynamoDB Streams

DynamoDB Streams provides at-least-once delivery within a shard. Lambda processes a batch of records, then advances the shard iterator. If the Lambda fails, the same batch is retried. To achieve exactly-once:

- Make the consumer idempotent (upsert semantics)
- Use a deduplication store (Redis set, DynamoDB TTL table) keyed by `eventID`
- Track `SequenceNumber` per shard to skip already-processed records

---

## MongoDB Change Streams

MongoDB offers CDC natively through Change Streams, available since MongoDB 3.6.

### How It Works

- Change streams read from the write-ahead log (oplog)
- They are available at the collection, database or deployment level
- A resume token allows restarting from the last processed change
- Supports aggregation pipeline filtering

```javascript
const pipeline = [
  { $match: { "fullDocument.status": "SHIPPED" } },
  { $project: { "fullDocument.orderId": 1, operationType: 1 } },
];

const changeStream = db.collection("orders").watch(pipeline, {
  // Resume from a specific token
  resumeAfter: savedResumeToken,
  fullDocument: "updateLookup",
});

changeStream.on("change", (change) => {
  console.log(change.operationType, change.fullDocument);
  // Store the resume token for restart
  saveResumeToken(change._id);
});
```

### Limitations

- **Oplog size** — if the consumer is slower than the write rate, the oplog (typically 5% of free disk) wraps and changes are lost
- **Full document lookup** — `updateLookup` fetches the current document, which can be different from the version at the time of the update
- **Resume token** — tied to a specific replica set. Cannot resume across replica set elections in older MongoDB versions (3.6–4.0). Fixed in 4.2+
- **No schema enforcement** — Change Streams emit whatever is written; upstream schema changes break downstream consumers

---

## The Outbox Pattern

The outbox pattern is the primary alternative to CDC for achieving reliable event publication without dual-write.

### How It Works

1. Application inserts an event record into an **outbox table** in the same database transaction as the business data
2. A separate process (polling publisher, CDC connector) reads the outbox table and publishes events to the broker
3. The publisher deletes or marks events as published

```python
def create_order_with_outbox(order_data):
    with db.transaction():
        # 1. Insert business data
        order_id = db.insert("orders", order_data)

        # 2. Insert outbox event (same transaction)
        db.insert("outbox", {
            "id": str(uuid.uuid4()),
            "aggregate_type": "order",
            "aggregate_id": order_id,
            "event_type": "OrderCreated",
            "payload": json.dumps(order_data),
            "created_at": now()
        })

    # Outbox publisher handles the async publish
```

### Outbox Table Schema

```sql
CREATE TABLE outbox (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id   VARCHAR(100) NOT NULL,
    event_type     VARCHAR(100) NOT NULL,
    event_version  INTEGER NOT NULL DEFAULT 1,
    payload        JSONB NOT NULL,
    traceparent    VARCHAR(55),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMPTZ,
    -- Index for polling publisher
    CONSTRAINT outbox_publish_idx UNIQUE (id, published_at)
);

CREATE INDEX idx_outbox_unpublished
    ON outbox (created_at ASC)
    WHERE published_at IS NULL;
```

### Outbox + CDC (The Best of Both Worlds)

Rather than writing a polling publisher, use CDC (Debezium) to read the outbox table and publish events:

```
Application → (insert into orders + outbox) → Postgres WAL → Debezium → Kafka → Sinks
```

This gives:

- No polling overhead
- Sub-second event delivery
- Strict ordering per aggregate (outbox events are inserted in transaction order)
- Kafka log compaction (use `aggregate_id` as the key)

### Outbox vs Pure CDC

| Aspect                        | Pure CDC                                 | Outbox                                     |
| ----------------------------- | ---------------------------------------- | ------------------------------------------ |
| Coupling                      | No application changes                   | Must insert into outbox table              |
| Event design                  | Raw DB rows (including internal columns) | Explicit event schema per domain event     |
| Deletes                       | Captured naturally                       | Must be modelled as a domain event         |
| Schema changes                | Risk of breaking downstream consumers    | Controlled — outbox table schema is stable |
| Multiple events per DB change | Not possible (one row → one event)       | One DB change can produce many events      |

**Recommendation:** Use Outbox + CDC together. Insert domain events into the outbox table; let Debezium stream them.

---

## Exactly-Once Semantics in CDC

### The Two Problems

1. **Duplicate events** — the source DB may commit a transaction, the CDC connector reads the log entry, then crashes before committing the offset. On restart, it replays the same log entry, producing a duplicate event.
2. **Reordering** — if the CDC connector processes transactions out of order (e.g. due to multi-threaded parsing), events may arrive at the downstream system in the wrong sequence.

### Achieving Exactly-Once

**At the source:**

- Use a transaction log (WAL, binlog) with atomic offset management
- Debezium commits offsets to Kafka with the same transactional semantics (`exactly.once` support with Kafka transactions)

**At the sink:**

- Make downstream consumers idempotent:
  - Elasticsearch: `_id` = `<table>|<pk>`, use upsert
  - S3: write to `s3://bucket/year/month/day/hour/<table>/<pk>.json`
  - Kafka topic: use `tombstones.on.delete` with log compaction; consumers deduplicate via idempotent writes
- For strict ordering per key, partition by primary key and use `max.in.flight.requests.per.connection=1` in the sink connector

### Kafka Transactions for CDC

Debezium can be configured to use Kafka's exactly-once semantics:

```json
{
  "transactional.id": "debezium-orders-tx",
  "exactly.once.source": "true"
}
```

- Debezium produces all change events from a single source transaction in a single Kafka transaction
- If the connector fails mid-batch, the Kafka transaction is aborted and events are not visible to consumers
- On restart, Debezium re-reads the replication slot from the last committed LSN

**Downside:** Throughput drops 30–50% due to the overhead of Kafka transactions and distributed commit coordination.

---

## Performance and Operational Considerations

### Database Impact

| CDC Method           | CPU Overhead             | Storage Overhead                 | WAL Growth                          |
| -------------------- | ------------------------ | -------------------------------- | ----------------------------------- |
| Log-based (Debezium) | < 5% on idle reads       | ~ 0% (reads existing log)        | Depends on `max_slot_wal_keep_size` |
| Trigger-based        | 10–25% per write         | Shadow table adds storage        | N/A                                 |
| Timestamp-based      | Varies by poll frequency | `updated_at` column (negligible) | N/A                                 |
| DynamoDB Streams     | ~ 0% (AWS managed)       | Stream data (24h retention)      | N/A                                 |

### Kafka Connect Tuning for High-Volume CDC

```json
{
  "config": {
    "tasks.max": "4",
    "max.batch.size": "2048",
    "poll.interval.ms": "100",
    "heartbeat.interval.ms": "3000",
    "snapshot.fetch.size": "10000",
    "incremental.snapshotting.enabled": "true",
    "signal.data.collection": "public.debezium_signal"
  }
}
```

- **`tasks.max`** — parallelism; more tasks = more concurrent WAL readers (one per partition for Postgres)
- **`max.batch.size`** — number of events per batch; larger = higher throughput but more memory
- **`incremental.snapshotting`** — take snapshots in chunks instead of locking tables; essential for large tables without downtime
- **`signal.data.collection`** — control connector signals (ad-hoc snapshot, stop, etc.) via a DB table

### Monitoring Metrics

| Metric                             | Tool                                                 | Alert Threshold          |
| ---------------------------------- | ---------------------------------------------------- | ------------------------ |
| Replication slot lag (bytes)       | `pg_replication_slots` PG view                       | > 500 MB                 |
| Kafka Connect lag (records behind) | Kafka consumer group offset                          | > 10,000 per partition   |
| Debezium snapshot progress         | `debezium-metrics` JMX MBeans                        | Stalled > 5 minutes      |
| WAL generation rate                | CloudWatch / pg_stat_statements                      | > 2x baseline for 10 min |
| Connector failure rate             | Kafka Connect REST API (`/connectors/<name>/status`) | Any failure              |

---

## Summary Decision Matrix

| Requirement                           | Best Approach                                       |
| ------------------------------------- | --------------------------------------------------- |
| Real-time event stream from Postgres  | Debezium (pgoutput plugin)                          |
| Real-time event stream from MySQL     | Debezium (binlog, row-based)                        |
| Replicate to data warehouse           | DMS (managed, no Kafka)                             |
| AWS-native DynamoDB to Lambda         | DynamoDB Streams                                    |
| MongoDB to Elasticsearch              | Kafka Connect (Debezium source, Elasticsearch sink) |
| No extra infrastructure               | Trigger-based CDC (small scale)                     |
| Exactly-once semantics                | Outbox + CDC + idempotent sink                      |
| Schema evolution support              | Debezium + Avro + Schema Registry                   |
| High-volume (10k+ events/sec)         | Debezium + Kafka (tuned)                            |
| Event-driven microservices            | Outbox pattern + CDC                                |
| Backfill old data                     | Debezium incremental snapshot                       |
| Embedded CDC in a Quarkus service     | Debezium Quarkus Extension (3.4+)                   |
| DynamoDB changes beyond 24h retention | DynamoDB Streams → Kinesis Data Streams             |
