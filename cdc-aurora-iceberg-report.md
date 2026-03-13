# CDC Systems: AWS RDS Aurora → Apache Iceberg on S3
## A Comprehensive Technical Report for Data Engineering Teams

**Date:** March 2025  
**Scope:** Change Data Capture pipeline design from AWS Aurora (MySQL/PostgreSQL) to a Data Lake on AWS S3 using Apache Iceberg (and comparable open table formats), with consideration for future event bus expansion.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [CDC Pipeline Architecture Overview](#2-cdc-pipeline-architecture-overview)
3. [Tech Stack Breakdown by Layer](#3-tech-stack-breakdown-by-layer)
   - [Layer 1: CDC Connectors / Log Readers](#layer-1-cdc-connectors--log-readers)
   - [Layer 2: Message Brokers / Event Streams](#layer-2-message-brokers--event-streams)
   - [Layer 3: Stream Processors](#layer-3-stream-processors)
   - [Layer 4: Open Table Formats](#layer-4-open-table-formats)
   - [Layer 5: Query / Catalog Engines](#layer-5-query--catalog-engines)
4. [Architecture Patterns](#4-architecture-patterns)
5. [Future Event Bus Expansion](#5-future-event-bus-expansion)
6. [Recommendations](#6-recommendations)
7. [References](#7-references)

---

## 1. Executive Summary

Change Data Capture (CDC) is the process of tracking and streaming row-level database changes (INSERT, UPDATE, DELETE) in near real-time to downstream consumers. For data engineering teams running on AWS with Aurora as the operational database, CDC unlocks:

- **Low-latency analytics:** data in the lake within seconds or minutes, not overnight
- **Exactly-once semantics:** reliable upserts/deletes without full table scans
- **Event-driven architecture:** the same change stream powers analytics *and* microservices
- **Cost efficiency:** no repeated full-table exports; only deltas are moved

Apache Iceberg has emerged as the dominant open table format for landing CDC data in S3 data lakes, thanks to its ACID transaction support, schema evolution, hidden partitioning, and broad engine compatibility. However, the choice of connector, message broker, and stream processor dramatically affects operational complexity, latency, cost, and future extensibility.

**Key findings:**
- **Debezium + Kafka (or Redpanda) + Apache Flink + Iceberg** is the industry-standard, production-proven architecture for sub-minute latency
- **AWS DMS → Kinesis → Glue/Flink → Iceberg** is the most AWS-native path with least operational overhead
- **Airbyte → S3 → dbt** is the simplest ELT pattern but sacrifices real-time capability
- For future event bus expansion, **Kafka (MSK/Confluent)** or **Redpanda** provide the strongest foundation
- **NATS JetStream** excels at microservices messaging but lacks the CDC ecosystem depth of Kafka

---

## 2. CDC Pipeline Architecture Overview

A complete CDC pipeline from Aurora to an Iceberg data lake consists of five logical layers:

```
┌─────────────────┐    ┌──────────────┐    ┌───────────────┐    ┌────────────┐    ┌──────────────┐
│  Aurora MySQL/  │    │   Message    │    │    Stream     │    │   Open     │    │  Query /     │
│  PostgreSQL     │───▶│   Broker     │───▶│   Processor   │───▶│  Table     │───▶│  Catalog     │
│                 │    │              │    │               │    │  Format    │    │  Engine      │
│  (Source DB)    │    │  Kafka /     │    │  Flink /      │    │  Iceberg / │    │  Athena /    │
│                 │    │  Redpanda /  │    │  Spark /      │    │  Hudi /    │    │  Trino /     │
│  CDC Connector  │    │  Kinesis /   │    │  Glue         │    │  Delta     │    │  Glue Cat.   │
│  (Debezium /    │    │  NATS        │    │               │    │            │    │              │
│  AWS DMS /      │    │              │    │               │    │            │    │              │
│  Airbyte)       │    │              │    │               │    │            │    │              │
└─────────────────┘    └──────────────┘    └───────────────┘    └────────────┘    └──────────────┘
     Layer 1                Layer 2             Layer 3             Layer 4            Layer 5
```

Each layer can be independently swapped, creating dozens of valid architectural combinations. The sections below evaluate the major options per layer.

---

## 3. Tech Stack Breakdown by Layer

### Layer 1: CDC Connectors / Log Readers

This layer reads the database's binary/write-ahead log (binlog for MySQL, WAL for PostgreSQL) and produces a stream of structured change events. It is the most critical layer — a faulty connector can corrupt downstream state.

#### 3.1.1 Debezium

**Description:** Open-source distributed CDC platform built on Apache Kafka Connect. The de facto standard for log-based CDC. Originally developed at Red Hat, now under the Commonhaus Foundation (formerly incubating in Apache). Supports MySQL, PostgreSQL, MongoDB, SQL Server, Oracle, and many more.

**Key Features:**
- Reads binlog (MySQL) or WAL (PostgreSQL) directly via logical replication slots
- Produces JSON or Avro change events with `before`/`after` image of each row
- Supports schema registry integration (Confluent, Apicurio)
- Can run as a Kafka Connect plugin, as a standalone server, or embedded in applications
- Schema change events automatically tracked
- Exactly-once delivery possible with Kafka's exactly-once semantics

**Strengths:**
- Battle-tested in thousands of production systems (LinkedIn, Netflix, Goldman Sachs use similar patterns)
- Rich event format: `op` field (c=create, u=update, d=delete, r=read/snapshot)
- Active community; new connectors and features released regularly
- Kafka Connect deployment means it scales horizontally with Connect workers
- Flink CDC connector for Debezium format allows bypassing Kafka entirely

**Weaknesses:**
- Requires careful PostgreSQL configuration: `wal_level = logical`, dedicated replication slot
- Replication slots on Aurora can lag and cause WAL accumulation if consumer is slow — **critical operational risk**
- Snapshot phase (initial load) can be large and long-running
- Operational complexity: Kafka Connect cluster management, schema registry, connector configuration
- MySQL requires GTID mode or binlog position management
- Aurora MySQL: requires `binlog_format = ROW` and specific parameter group settings

**Aurora-Specific Notes:**
- **Aurora PostgreSQL:** Works with Debezium PostgreSQL connector. Must configure `rds.logical_replication = 1` in parameter group. Use `pgoutput` or `wal2json` plugin. **Beware:** if the consumer falls behind, Aurora will keep accumulating WAL — set `max_slot_wal_keep_size` to prevent disk-full scenarios.
- **Aurora MySQL:** Works with Debezium MySQL connector. Requires binary logging enabled (default on Aurora). `binlog_format` must be `ROW`. Aurora MySQL supports GTID which simplifies failover.

**Best for:** High-throughput CDC with sub-second latency; teams with Kafka infrastructure; complex transformations via Kafka Streams/Flink; future event bus plans.

---

#### 3.1.2 AWS Database Migration Service (AWS DMS)

**Description:** AWS-managed service for database migration and CDC. Supports one-time migrations and ongoing replication. Integrates natively with Aurora, Kinesis, S3, and other AWS targets.

**Key Features:**
- Fully managed: no infrastructure to maintain
- Supports Aurora MySQL and PostgreSQL as sources out-of-the-box
- CDC can stream to Amazon Kinesis Data Streams, Amazon S3, or Kafka (MSK)
- Replication instances are EC2-based, sized by workload
- Serverless DMS option available (scales automatically)
- Native support for schema conversion and table mapping rules

**Strengths:**
- Zero infrastructure management — no Kafka Connect cluster to operate
- Deep Aurora integration: reads binlog/WAL natively with AWS support
- Works well with Kinesis for downstream AWS-native processing
- Multi-AZ replication instances for HA
- AWS Console/CLI/CDK for setup
- Cost: pay-per-use on serverless, or fixed EC2 for standard instances

**Weaknesses:**
- Less flexible event format compared to Debezium (JSON but with limited metadata)
- Kinesis Data Streams target has shard limitations (stream shards ≠ Kafka partitions in flexibility)
- Not ideal for building a full event bus — Kinesis is not a general-purpose message broker
- DMS S3 target produces CSV/Parquet files with `Op` column, not Iceberg natively — requires post-processing step via Glue/Athena
- Schema changes require manual handling or can break pipelines
- Latency: seconds to tens-of-seconds depending on task settings
- **Important 2025 Note:** AWS announced that CDC Firehose streams (the native Firehose → Iceberg path) will stop working after September 30, 2025 — recommend migrating to DMS → Kinesis → Flink/Glue → Iceberg pipeline instead

**AWS DMS → Iceberg Pattern:**
DMS writes CDC events to S3 as CSV or Parquet (with `Op`, `schema_name`, `table_name` columns), then AWS Glue or Athena performs MERGE INTO operations on Iceberg tables. This is the most common AWS-native pattern per AWS Big Data Blog (November 2024).

**Best for:** Teams wanting AWS-managed CDC with minimal operational overhead; pure AWS shops; migrations from legacy databases to data lakes; moderate latency requirements (5–30 seconds acceptable).

---

#### 3.1.3 Airbyte

**Description:** Open-source data integration platform with 300+ connectors. Supports log-based CDC for PostgreSQL and MySQL. Popular for ELT workloads.

**Key Features:**
- Log-based CDC for PostgreSQL (using pgoutput) and MySQL (binlog)
- Destinations include S3, Snowflake, Redshift, BigQuery, and (alpha) Apache Iceberg
- Open-source self-hosted or cloud-managed (Airbyte Cloud)
- No-code UI + YAML-based connector configuration
- Normalization via dbt integration

**Strengths:**
- Easy to set up; excellent UI
- Large connector library
- Community-driven; rapidly growing ecosystem
- Supports Iceberg S3 destination (as of 2024, in alpha/beta)
- Good for ELT patterns (land raw CDC data, transform with dbt)

**Weaknesses:**
- **Aurora CDC caching layer incompatibility:** Airbyte's PostgreSQL CDC implementation is incompatible with Aurora's CDC caching layer by default — must disable Aurora's CDC cache or use workarounds
- Latency: Airbyte syncs run on schedules (minimum ~5 minutes for CDC), not true streaming
- Not designed for sub-second or sub-minute CDC
- The Iceberg destination connector is not production-grade as of early 2025
- Limited transformation capabilities during ingestion
- No native message broker integration — all data lands directly in S3

**Best for:** Teams prioritizing simplicity over latency; ELT-style pipelines; batch or micro-batch CDC (5+ minute latency acceptable); small-to-medium scale.

---

#### 3.1.4 Fivetran

**Description:** Fully managed, commercial ELT platform with 500+ connectors. Enterprise-grade CDC for Aurora and other databases.

**Key Features:**
- Log-based CDC using proprietary agents
- Syncs as frequently as every 5 minutes
- Destinations: Snowflake, BigQuery, Redshift, Databricks (Delta Lake), S3
- Automatic schema drift handling
- SOC 2 Type II, ISO 27001 compliant

**Strengths:**
- Zero-maintenance managed service
- Excellent reliability and monitoring
- Handles schema evolution automatically
- Strong enterprise support
- Works with Aurora MySQL and PostgreSQL

**Weaknesses:**
- **Expensive:** pricing based on Monthly Active Rows (MAR) can escalate rapidly for high-volume tables
- Closed-source proprietary system — vendor lock-in risk
- Limited control over event format and processing
- No native Iceberg support as of early 2025 (destinations are warehouses, not data lakes)
- Not suitable for building an event bus

**Best for:** Teams that want to pay for reliability and simplicity; analytics-focused ELT to data warehouses; organizations without dedicated data engineering capacity.

---

#### 3.1.5 Striim

**Description:** Enterprise-grade CDC and real-time data streaming platform. Focuses on continuous, low-latency data ingestion with in-flight transformations.

**Key Features:**
- Log-based CDC for 50+ sources including Aurora MySQL and PostgreSQL
- Built-in SQL-based stream transformations (WQL — Westar Query Language)
- Delivers to Kafka, S3, Snowflake, BigQuery, and more
- Pre-packaged applications for initial loads + CDC
- Enterprise schema change replication
- Cloud and on-premise deployment options

**Strengths:**
- Rich in-flight transformation capabilities
- Reliable checkpointing for fault tolerance
- Good for complex multi-source CDC pipelines
- Strong enterprise support and SLAs

**Weaknesses:**
- **Expensive:** enterprise pricing, not for startups
- Proprietary platform; significant vendor lock-in
- Less community/open-source ecosystem than Debezium
- Complex licensing and configuration for large deployments

**Best for:** Enterprise teams with budget, complex transformation needs, and multi-source CDC requirements.

---

#### 3.1.6 Redpanda Connect (formerly Bento / Benthos)

**Description:** Originally open-sourced as **Benthos** (Go-based stream processor), renamed to **Bento**, then acquired by Redpanda in May 2024 and rebranded as **Redpanda Connect**. It is a standalone, declarative data pipeline tool — not just a companion to the Redpanda broker. Single Go binary, 300+ connectors, zero JVM. Positioned as a full alternative to Kafka Connect with native CDC input connectors added post-acquisition.

**Key Features:**
- Native `postgres_cdc` connector: reads Aurora PostgreSQL WAL via logical replication
- Native `mysql_cdc` connector: reads Aurora MySQL binlog (GTID support coming; currently binlog position + external cache)
- `iceberg` sink connector: writes directly to Iceberg tables on S3
- Bloblang: a powerful built-in mapping/transformation language for in-flight data shaping
- Declarative YAML pipeline config: input → pipeline (processors) → output
- At-least-once delivery guarantee with built-in checkpointing
- Runs standalone or as a sidecar — no separate Connect cluster needed

**Killer Feature — Parallel Snapshotting:**
Debezium performs the initial database snapshot serially (one table at a time, one row at a time). Redpanda Connect shards large tables dynamically and reads them in parallel, dramatically reducing initial load time for large Aurora tables. This is a meaningful operational advantage for databases with hundreds of millions of rows.

**Aurora-Specific Notes:**
- **Aurora PostgreSQL:** Requires `rds.logical_replication = 1`, same as Debezium. Uses `pgoutput` plugin. Replication slot management applies — monitor slot lag.
- **Aurora MySQL:** Requires `binlog_format = ROW`. External cache (Redis, DynamoDB, or SQL table) required for binlog offset storage. GTID mode not yet supported as of early 2025 — limits failover resilience compared to Debezium on Aurora MySQL.

**Architecture Pattern:**
```
Aurora → Redpanda Connect (postgres_cdc input) → [optional: Redpanda broker] → Redpanda Connect (iceberg output) → S3
```
The broker is optional for simple pipelines — Connect can write directly from CDC source to Iceberg sink in a single pipeline. For fan-out (multiple consumers), insert Redpanda in the middle.

**Strengths:**
- Single binary, no JVM — operationally much simpler than Kafka Connect + Debezium
- Parallel snapshot phase significantly faster than Debezium for large tables
- Built-in Iceberg sink eliminates need for separate Flink/Spark job for pure CDC → Iceberg
- Bloblang transformations handle field filtering, renaming, type coercion inline
- Works with any Kafka-compatible broker (MSK, Confluent) or standalone
- Excellent fit for all-Redpanda stacks (broker + Connect = one vendor)
- Active development: MySQL CDC added late 2024, Iceberg sink maturing through 2025

**Weaknesses:**
- MySQL CDC: no GTID support yet — external cache required for offset tracking, less robust failover
- No stateful stream processing (joins, windowed aggregations, deduplication at scale) — for complex transformations, Flink still required
- Smaller community than Debezium for troubleshooting production issues
- Iceberg sink maturity still growing (less battle-tested than Flink IcebergSink at large scale)
- Limited schema registry integration compared to Debezium + Confluent SR

**vs. Debezium:**

| Feature | Redpanda Connect | Debezium |
|---|---|---|
| Language | Go (single binary) | Java (JVM, Kafka Connect) |
| Parallel Snapshot | ✅ Native | ❌ Serial only |
| Aurora PostgreSQL CDC | ✅ | ✅ |
| Aurora MySQL CDC | ✅ (no GTID yet) | ✅ (full GTID) |
| Native Iceberg Sink | ✅ | ❌ (needs Kafka + Flink/Connect) |
| Schema Registry | Limited | ✅ Full (Confluent/Apicurio) |
| Stateful Processing | ❌ | ❌ (handled by Flink/Kafka Streams) |
| Operational Complexity | Low | Medium-High |
| Community / Maturity | Growing (2024–) | Very mature (2016–) |

**Best for:** Aurora PostgreSQL → Iceberg pipelines with simple-to-moderate transformations; teams choosing Redpanda as broker who want a unified vendor; teams that want to avoid JVM/Kafka Connect operational overhead; greenfield projects where parallel snapshot speed matters.

---

#### 3.1.7 Flink CDC Connector (Direct Database Reading)

An emerging alternative: Apache Flink's [flink-cdc](https://github.com/apache/flink-cdc) project provides native Debezium-based connectors that allow Flink to read CDC events **directly from the database** without an intermediate Kafka/message broker. This is the "Flink CDC" approach popularized by Alibaba and now an Apache project.

**Key Features:**
- Reads MySQL binlog and PostgreSQL WAL directly via Debezium engine
- Zero-copy architecture: no Kafka needed for simple Flink → Iceberg pipelines
- Supports exactly-once semantics via Flink checkpointing
- Full SQL API: `CREATE TABLE mysql_source WITH (connector='mysql-cdc', ...)`

**Strengths:**
- Simplified architecture (no Kafka)
- Lower latency
- Unified Flink job for capture + processing + write

**Weaknesses:**
- Less flexible than Kafka-based: all consumers must be within the Flink job
- Harder to fan-out to multiple downstream consumers
- Flink operational complexity still applies
- Production maturity still growing (Flink CDC 3.x released 2024)

**Best for:** Flink-first shops wanting minimal infrastructure; Iceberg-only target without event bus needs.

---

#### CDC Connectors Comparison Summary

| Tool | Type | Latency | Aurora Support | Iceberg Native | Event Bus Ready | Operational Load | Cost |
|------|------|---------|---------------|----------------|-----------------|-----------------|------|
| Debezium | Open Source | <1s | ✅ (config required) | Via Kafka/Flink | ✅ (Kafka) | Medium-High | Free |
| Redpanda Connect | Open Source | <1s | ✅ (MySQL: no GTID) | ✅ Native sink | ✅ (via Redpanda) | Low | Free/Enterprise |
| AWS DMS | Managed SaaS | 5–30s | ✅ Native | Via Glue/Flink | ❌ | Low | Pay-per-use |
| Airbyte | Open Source / SaaS | 5min+ | ⚠️ (Aurora quirks) | Alpha | ❌ | Low-Medium | Free/SaaS |
| Fivetran | Managed SaaS | 5min | ✅ | ❌ (warehouses) | ❌ | Very Low | Expensive |
| Striim | Enterprise | <1s | ✅ | Via Kafka | Partial | Medium | Expensive |
| Flink CDC | Open Source | <1s | ✅ | ✅ | ❌ (no broker) | Medium-High | Free |

---

### Layer 2: Message Brokers / Event Streams

The message broker decouples the CDC source from downstream consumers, enables fan-out, provides replay, and is the foundation for future event bus expansion.

#### 3.2.1 Apache Kafka (Confluent Cloud / Amazon MSK)

**Description:** The industry-standard distributed event streaming platform. Created at LinkedIn in 2011, now Apache Software Foundation. Kafka is the de facto backbone for CDC pipelines globally.

**Key Features:**
- Distributed log with configurable retention (hours to forever)
- Topics, partitions, consumer groups for fan-out
- Exactly-once semantics (EOS) with transactional producers
- Kafka Connect for source/sink connectors (including Debezium)
- KRaft mode eliminates ZooKeeper dependency (Kafka 3.x+)
- Schema Registry (Confluent) for Avro/Protobuf/JSON Schema governance

**Managed Offerings:**
- **Confluent Cloud:** Fully managed Kafka with 80+ managed connectors, serverless Flink, Tableflow (Kafka → Iceberg/Delta), KSQL, enterprise support
- **Amazon MSK:** Managed Kafka on AWS. Two flavors: MSK Provisioned and MSK Serverless. Deep AWS IAM integration. MSK Connect for Kafka Connect.

**Strengths:**
- Largest ecosystem: Debezium, Flink, Spark all have first-class Kafka integration
- Battle-tested at massive scale (billions of messages/day at Netflix, LinkedIn)
- Log retention enables reprocessing and audit
- Rich monitoring ecosystem (Prometheus, Grafana, Confluent Control Center)
- Schema registry for governance
- Multiple cloud-managed options

**Weaknesses:**
- JVM-based: higher memory footprint and garbage collection pauses
- Historically required ZooKeeper (eliminated in KRaft mode, stable in Kafka 3.7+)
- Complex tuning: partitions, replication factor, ISR, retention
- MSK is more expensive than self-managed Kafka at scale
- Confluent Platform licensing costs for enterprise features

**Latency:** Typically 1–10ms end-to-end within a cluster; P99 latency at 1GB/s throughput: ~5500ms per some benchmarks (vs Redpanda's 79ms)

**Best for:** Enterprise-scale CDC with rich ecosystem requirements; future event bus; teams already using Kafka.

---

#### 3.2.2 Redpanda

**Description:** C++ reimplementation of the Kafka API, built ground-up for performance and simplicity. Kafka-compatible — drop-in replacement for most use cases.

**Key Features:**
- Kafka-wire protocol compatible (works with all Kafka clients, Debezium, Flink, etc.)
- No JVM, no ZooKeeper — single binary
- Thread-per-core architecture for predictable performance
- Built-in Schema Registry (compatible with Confluent Schema Registry API)
- Tiered Storage: hot data on local disk, cold data in S3
- Redpanda Cloud: fully managed offering
- Built-in HTTP Proxy (Pandaproxy), REST API

**Performance:**
- **3–6x lower cost** vs traditional Kafka infrastructure (per Redpanda benchmarks)
- At 1GB/s throughput with 3 nodes: **P99.99 latency of 79.57ms** vs Kafka's **5509.73ms on 6 nodes** (70x improvement) — per LinkedIn benchmark data cited by Redpanda
- Significantly fewer nodes required for same throughput

**Strengths:**
- Superior latency at high throughput
- Simpler operations: fewer moving parts, no ZooKeeper
- Lower infrastructure costs
- Kafka API compatibility means zero code changes for Debezium/Flink
- Built-in Schema Registry eliminates one dependency
- Excellent for teams not already on Kafka ecosystem

**Weaknesses:**
- Smaller ecosystem than Kafka (fewer managed connectors)
- Kafka Connect ecosystem works, but some connectors have edge cases
- Confluent-specific features (Tableflow, KSQL) not available
- Smaller community; fewer Stack Overflow answers
- Enterprise features require Redpanda Enterprise license

**Best for:** Greenfield CDC projects; latency-sensitive pipelines; teams wanting Kafka compatibility with less operational overhead; resource-constrained environments.

---

#### 3.2.3 Amazon Kinesis Data Streams

**Description:** AWS-native fully managed event streaming service. Designed for AWS-native architectures.

**Key Features:**
- Shards-based scaling (each shard: 1 MB/s write, 2 MB/s read)
- Kinesis Data Streams Enhanced Fan-Out: dedicated 2 MB/s per consumer
- AWS DMS natively integrates with Kinesis as a CDC target
- AWS Lambda, AWS Glue, AWS Flink (Managed Flink) can consume from Kinesis
- Kinesis Data Firehose for no-code delivery to S3, Redshift, Iceberg

**Strengths:**
- Zero infrastructure management
- Deep AWS integration (IAM, CloudWatch, VPC)
- Works seamlessly with AWS DMS for pure-AWS CDC pipeline
- Automatic scaling with on-demand mode
- Low operational burden

**Weaknesses:**
- **Not Kafka-compatible:** cannot use Debezium Kafka Connect or Kafka-based Flink connectors directly
- Shard model less flexible than Kafka partitions
- Retention limited to 7 days default (24h default), extendable to 365 days at cost
- No schema registry
- **Less suitable as general-purpose event bus:** Kinesis is AWS-specific and lacks the rich ecosystem of Kafka
- Kinesis Data Firehose → Iceberg CDC path being **deprecated September 30, 2025**
- Higher cost at very high throughput compared to Redpanda/Kafka

**Best for:** Pure AWS DMS → Iceberg pipelines; teams committed to AWS-native stack; when operational simplicity trumps flexibility.

---

#### 3.2.4 NATS JetStream

**Description:** NATS is a lightweight, high-performance messaging system. JetStream is its persistence layer that adds streaming semantics (subject-based routing, consumers, replay).

**Key Features:**
- Subject-based publish/subscribe (not topic/partition like Kafka)
- Sub-millisecond latency
- Simple configuration: single binary, no ZooKeeper
- Multi-tenancy with Accounts
- JetStream persistent streams with configurable retention
- Object Store for large blobs (up to 64MB max message, configurable)
- Key-Value store built on JetStream
- Cluster support (RAFT-based consensus)

**Performance:**
- NATS is exceptionally fast for microservices messaging
- JetStream persistence adds some overhead but remains faster than Kafka for small messages

**Strengths:**
- Extremely simple to operate
- Very low latency
- Great for microservices event-driven communication
- Multi-tenant accounts with isolation
- Lightweight: single Go binary

**Weaknesses:**
- **No Debezium Kafka Connect integration** — Debezium cannot write directly to NATS (would need custom bridge)
- No native CDC connector ecosystem
- No schema registry comparable to Confluent/Redpanda
- Less battle-tested for data lake ingestion patterns at scale
- Smaller data engineering ecosystem (Flink has no native NATS connector)
- Max message size limits (64MB) can be challenging for large CDC events
- **Not suitable as a Kafka drop-in for CDC pipelines without significant custom work**

**Best for:** Microservices messaging; IoT event routing; internal application events; complement to Kafka rather than replacement for data lake CDC.

---

#### 3.2.5 Apache Pulsar

**Description:** Cloud-native distributed messaging and streaming platform from Yahoo/StreamNative. Uses a tiered architecture separating compute (brokers) from storage (BookKeeper).

**Key Features:**
- Topics, subscriptions, consumer groups
- Native multi-tenancy with namespaces
- Geo-replication built-in
- Tiered storage to S3 natively
- Pulsar Functions for serverless stream processing
- Pulsar IO for connectors (source/sink)
- Compatible with Kafka clients via Kafka-on-Pulsar (KoP)

**Strengths:**
- Multi-tenancy is a first-class citizen (better than Kafka for multi-tenant event buses)
- Built-in geo-replication
- Tiered storage reduces cost
- Good for complex routing patterns
- StreamNative managed cloud option

**Weaknesses:**
- More complex architecture than Kafka (separate BookKeeper cluster required)
- Smaller ecosystem than Kafka; Debezium has limited Pulsar support
- Flink Pulsar connector exists but is less mature than Kafka connector
- Less commonly used for CDC data lake pipelines specifically
- Steeper learning curve

**Best for:** Multi-tenant event buses; geo-replicated architectures; enterprises already on Pulsar.

---

#### Message Broker Comparison Summary

| Broker | Kafka Compatibility | Latency | CDC Ecosystem | Event Bus Maturity | AWS Managed | Cost Efficiency |
|--------|--------------------|---------|--------------|--------------------|-------------|-----------------|
| Kafka (MSK/Confluent) | Native | ~10ms | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ MSK | Medium |
| Redpanda | Full | ~1ms | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ Redpanda Cloud | High |
| Kinesis | ❌ | ~70ms | ⭐⭐ (AWS only) | ⭐⭐ | ✅ Native | Medium |
| NATS JetStream | ❌ | <1ms | ⭐ | ⭐⭐⭐ (micro-svc) | ❌ | Very High |
| Apache Pulsar | Via KoP | ~10ms | ⭐⭐⭐ | ⭐⭐⭐⭐ | ❌ | Medium |

---

### Layer 3: Stream Processors

Stream processors consume change events from the broker, apply transformations, handle deduplication, and write to the table format in S3.

#### 3.3.1 Apache Flink

**Description:** Stream-first distributed processing framework. Treats batch as a special case of streaming. Native Iceberg integration via `flink-iceberg` connector.

**Key Features:**
- True streaming model with event-time processing and watermarks
- Stateful computation with RocksDB/heap backends
- Exactly-once processing via Flink's two-phase commit (2PC) with checkpointing
- Flink SQL for declarative stream processing
- DataStream API for complex stateful logic
- Native Kafka source/sink connectors
- Apache Iceberg sink connector (first-class support)
- Flink CDC connectors for direct DB reads

**Flink + Iceberg Integration:**
Flink writes to Iceberg via `IcebergSink` or via `FlinkSink.forRowData()`. The two-phase commit protocol ensures that Iceberg snapshots are committed atomically with Flink checkpoints, providing exactly-once write semantics. This is the mechanism used by Etleap (per AWS Database Blog, June 2025) to achieve sub-5-second end-to-end latency.

**Strengths:**
- Sub-second to low-second latency
- Exactly-once semantics for Iceberg writes (via 2PC + checkpointing)
- Excellent CDC support (native Debezium format deserialization)
- Amazon Managed Service for Apache Flink (formerly Kinesis Data Analytics): fully managed Flink
- Strong Iceberg integration (Apache Flink + Iceberg is the recommended pattern per Iceberg docs)
- Flink SQL makes it accessible without Java/Scala expertise

**Weaknesses:**
- Complex to operate and tune (checkpointing, parallelism, state management)
- Job restarts can be disruptive if not properly managed with savepoints
- Stateful operators require careful memory management
- Not ideal for purely batch transformations
- Learning curve for stateful stream processing

**Best for:** Sub-minute CDC to Iceberg; complex streaming transformations; joins, aggregations, deduplication; enterprise-scale pipelines.

---

#### 3.3.2 Apache Spark Structured Streaming

**Description:** Micro-batch streaming extension of Apache Spark. Processes data in small batches with configurable trigger intervals.

**Key Features:**
- Spark SQL API for streaming (same API as batch)
- Trigger modes: micro-batch (default), continuous (experimental), once, available-now
- Checkpointing for fault tolerance
- Native Delta Lake integration; Iceberg integration via `iceberg-spark` runtime
- AWS Glue 4.0+ natively supports Spark Structured Streaming with Iceberg

**Strengths:**
- Familiar Spark API (many data engineers already know it)
- Excellent for mixed batch/streaming workloads (same codebase)
- AWS Glue Streaming: managed Spark Structured Streaming (no cluster management)
- Strong Delta Lake integration (Databricks ecosystem)
- Good for micro-batch CDC (1-minute intervals)
- Large Iceberg connector ecosystem

**Weaknesses:**
- Not true streaming — micro-batch means minimum latency is the batch interval
- Higher latency than Flink (typically 30s–5min for practical setups)
- More heavyweight resource requirements than Flink for streaming
- State management less elegant than Flink
- Continuous mode (true streaming) remains experimental

**Best for:** Teams already on Spark/Databricks; mixed batch+streaming workloads; micro-batch CDC (1–5 minute latency acceptable); Delta Lake targets.

---

#### 3.3.3 AWS Glue Streaming

**Description:** Managed Spark Structured Streaming service on AWS. Processes data from Kinesis or MSK and writes to S3.

**Key Features:**
- Fully managed: no cluster management
- Supports Kinesis Data Streams and MSK as sources
- Iceberg support via AWS Glue 4.0 with `--enable-glue-datacatalog` and Iceberg JAR
- Built-in Glue Data Catalog integration
- Job bookmarks for fault-tolerant processing

**Strengths:**
- Zero infrastructure — fully managed
- Native AWS integration (IAM, VPC, CloudWatch)
- Works seamlessly with DMS → Kinesis → Glue Streaming → Iceberg
- Cost-effective for sporadic workloads (pay per DPU-second)

**Weaknesses:**
- Cold start times (minutes for cluster provisioning)
- Less flexible than self-managed Flink for complex transformations
- Spark micro-batch semantics: not sub-minute latency
- Limited connector ecosystem compared to Flink/Kafka Connect
- Debugging and monitoring less ergonomic than dedicated Flink cluster

**Best for:** Fully AWS-managed pipelines; teams without streaming infrastructure expertise; DMS → Kinesis → Glue → Iceberg pattern.

---

#### 3.3.4 Kafka Streams

**Description:** Client library for stream processing built into Apache Kafka. Runs inside your application, not a separate cluster.

**Key Features:**
- Java library (no separate processing cluster)
- Exactly-once semantics with Kafka EOS
- Stateful processing with KTable, GlobalKTable
- Interactive queries for state store inspection
- No separate cluster management

**Strengths:**
- Operationally simple: just a library in your app
- Excellent for Kafka-to-Kafka transformations
- Real-time aggregations and joins
- Great for CDC fan-out and enrichment patterns

**Weaknesses:**
- **No native Iceberg sink** — Kafka Streams writes back to Kafka topics; you need a separate Kafka Connect sink or Flink job to write to Iceberg
- Not designed for batch/data lake ingestion
- Less suitable for complex stateful streaming at very large scale (Flink scales better)

**Best for:** CDC enrichment, filtering, and fan-out within Kafka ecosystem; preprocessing before Flink/Spark writes to Iceberg.

---

#### Stream Processor Comparison Summary

| Processor | Latency | Iceberg Integration | Exactly-Once | Managed Option | Operational Load |
|-----------|---------|--------------------|--------------|--------------|--------------------|
| Apache Flink | <1s–5s | ⭐⭐⭐⭐⭐ | ✅ (2PC) | Amazon Managed Flink | Medium |
| Spark Structured Streaming | 30s–5min | ⭐⭐⭐⭐ | ✅ | AWS Glue Streaming | Low-Medium |
| AWS Glue Streaming | 1–5min | ⭐⭐⭐ | ✅ | ✅ Fully managed | Very Low |
| Kafka Streams | <1s | ❌ (Kafka only) | ✅ | ❌ | Low |

---

### Layer 4: Open Table Formats

Open table formats define how data files are organized, tracked, and queried in S3. They provide ACID transactions, schema evolution, time travel, and efficient upsert/delete handling — essential for CDC workloads.

#### 3.4.1 Apache Iceberg

**Description:** Open table format designed for large-scale analytics. Created at Netflix, donated to Apache. Supported by AWS, Apple, LinkedIn, Netflix, Dremio, and many others.

**Key Features:**
- ACID transactions on S3 (optimistic concurrency via catalog CAS)
- Hidden partitioning: partition transforms defined in metadata, not column values
- Schema evolution: add, drop, rename, reorder columns without rewriting
- Partition evolution: change partition strategy without rewriting historical data
- Time travel and snapshot isolation
- Row-level deletes via deletion vectors (as of Iceberg v2/v3 spec)
- Multiple catalog support: Hive Metastore, Glue Data Catalog, Nessie, REST Catalog

**CDC Support in Iceberg:**
- Iceberg v2 Format Version introduced **position deletes** and **equality deletes** for efficient row-level operations
- Deletion Vectors (v3 spec, experimental 2024) further improve delete performance
- `create_changelog_view` Spark procedure to view changes between snapshots
- Iceberg's `MERGE INTO` SQL command handles CDC upserts efficiently
- However: Iceberg's incremental read only supports **appends** natively — for full CDC (insert/update/delete), you need external merge logic via Flink or Spark

**Strengths:**
- **Broadest engine support:** Athena, Trino, Spark, Flink, Hive, Dremio, Snowflake, BigQuery, Redshift Spectrum, Databricks — all support Iceberg
- Best-in-class AWS support (AWS Glue, Athena, Lake Formation all first-class Iceberg)
- Apache Software Foundation governance (no vendor lock-in)
- `MERGE INTO` in Athena (DML) for CDC upsert patterns
- Strong schema evolution — critical for production CDC
- Partition evolution reduces operational pain

**Weaknesses:**
- Manual compaction required (unlike Hudi which has automated table services)
- Catalog required for consistency (cannot use without a catalog like Hive/Glue/Nessie)
- Iceberg's performance trails Hudi and Delta in some TPC-DS benchmarks
- Multi-writer concurrency uses OCC — competing writes fail and retry
- Iceberg's incremental read limitation (appends only) means CDC via Iceberg requires careful implementation

**Best for:** AWS-native data lakes; broadest query engine compatibility; teams prioritizing vendor neutrality; Apache ecosystem preference.

---

#### 3.4.2 Apache Hudi

**Description:** Open-source data lakehouse platform from Uber. The most feature-complete of the three formats, originally designed for CDC/update-heavy workloads.

**Key Features:**
- Copy-on-Write (CoW) and Merge-on-Read (MoR) table types
- Record-level indexing (8+ index types) for efficient upserts
- **Non-blocking concurrency control (NBCC)** — competing writers don't fail
- Automated table services: compaction, cleaning, clustering
- Incremental query: `hoocice.datasource.query.type=incremental` for change stream
- DeltaStreamer (now HoodieStreamer) for managed ingestion from Kafka/DMS
- Primary key support (unlike Iceberg/Delta which have no native primary keys)
- Hudi 1.0 (January 2025): native Apache Iceberg format support added

**Strengths:**
- **Best CDC support** natively: before/after images, change operations, incremental query
- Efficient upserts with record-level index (avoids full partition scans)
- Automated table maintenance (compaction, cleaning) — lower ops burden than Iceberg
- NBCC means less write contention in high-frequency CDC scenarios
- DeltaStreamer simplifies Kafka → Hudi pipelines
- Excellent for update-heavy workloads (Uber uses it for all ride data)

**Weaknesses:**
- More complex to operate than Iceberg for read-only/append workloads
- Smaller AWS native support (Athena: read-only; no Athena DML for Hudi)
- Less broad engine support for writes vs Iceberg
- Performance (TPC-DS) comparable to Delta, but lags in append-optimized workloads
- Hudi 1.0 adds Iceberg support — future convergence likely

**Best for:** Update-heavy CDC workloads; teams wanting native incremental pipeline support; organizations with existing Hudi investment; use cases needing upsert efficiency.

---

#### 3.4.3 Delta Lake

**Description:** Open-source storage layer from Databricks. First-class integration with Databricks platform; OSS version available for Spark and other engines.

**Key Features:**
- ACID transactions with optimistic concurrency (OCC)
- Time travel via transaction log
- Change Data Feed (CDF): enables incremental reads of changes (insert/update/delete)
- Schema enforcement and evolution
- Z-Order clustering for query performance
- Liquid Clustering (Delta 3.0+): auto-managing Z-order without manual maintenance
- Deletion Vectors for efficient deletes (since Delta 2.3)
- Unity Catalog for governance

**Strengths:**
- **Best Databricks integration** — native format for Databricks
- Change Data Feed provides CDC-like semantics within Delta tables
- Strong performance (comparable to Hudi, better than Iceberg in some benchmarks)
- Excellent Spark integration (Delta was built by Spark creators)
- Z-Order and Liquid Clustering for query optimization
- Growing OSS ecosystem (Delta Standalone, delta-rs in Rust)

**Weaknesses:**
- **Limited AWS-native support:** Athena supports Delta read with limitations; no Athena DML on Delta
- Proprietary features remain in Databricks Runtime (Autoloader, some clustering features)
- OCC only (no NBCC): competing writes fail and require retry
- Less broad engine support outside Databricks/Spark
- No primary key concept (like Iceberg)
- Historically more Databricks-locked than Iceberg

**Best for:** Databricks-centric architectures; teams using Spark as primary engine; organizations wanting the best Databricks ecosystem integration.

---

#### Open Table Format Comparison Summary

| Feature | Apache Iceberg | Apache Hudi | Delta Lake |
|---------|---------------|-------------|------------|
| CDC Native Support | Moderate (MERGE INTO) | ⭐⭐⭐⭐⭐ (native) | Good (CDF) |
| AWS Athena DML | ✅ Full | ❌ Read-only | ⚠️ Limited |
| AWS Glue Support | ✅ First-class | ✅ | ✅ |
| Flink Integration | ✅ Excellent | ✅ Good | ✅ Good |
| Spark Integration | ✅ Excellent | ✅ Excellent | ✅ Best (Databricks) |
| Automatic Compaction | ❌ Manual | ✅ Managed | ✅ (Databricks) |
| Primary Keys | ❌ | ✅ | ❌ |
| Multi-Writer Concurrency | OCC only | OCC + NBCC | OCC only |
| Vendor Neutrality | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Community (GitHub PRs/12mo) | 4,180 | 2,893 | 1,978 |
| Performance (TPC-DS) | Slowest | Medium | Medium |

---

### Layer 5: Query / Catalog Engines

#### 3.5.1 AWS Glue Data Catalog

The primary metadata catalog in the AWS ecosystem. Stores table definitions, schemas, and partition information. Iceberg tables can use Glue as their catalog via `catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog`.

**Strengths:** AWS-native; free tier generous; integrates with Athena, EMR, Glue ETL, Redshift Spectrum, Lake Formation.  
**Weaknesses:** AWS-only; no versioning of catalog state; limited branching/tagging.

#### 3.5.2 Apache Hive Metastore (HMS)

The original Hadoop-era metadata store. Widely supported by all engines.

**Strengths:** Universal support; battle-tested; works with on-premise and cloud.  
**Weaknesses:** Single point of failure; no ACID transactions on catalog; lacks modern features.

#### 3.5.3 Project Nessie

Git-like versioned catalog for data lakes. Supports branching, tagging, and merging of catalog state. Used by Dremio's Lakehouse platform.

**Strengths:** Multi-table transactions; branch-based isolation for CI/CD; time travel at catalog level.  
**Weaknesses:** Requires separate service; less AWS-native support; smaller adoption than Glue.

#### 3.5.4 AWS Athena

Serverless query engine for S3 data. Full Iceberg DML support (SELECT, INSERT, UPDATE, DELETE, MERGE INTO) as of Athena v3 (engine version 3). Critical for CDC upsert patterns without Spark/Flink.

**Use in CDC:** After DMS dumps CDC files to S3, Athena can execute `MERGE INTO iceberg_table USING staging_table ON key_match WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`

#### 3.5.5 Trino / Presto

Distributed SQL query engines. Trino (formerly PrestoSQL) has full Iceberg read+write support. Used for ad-hoc analytics on Iceberg tables.

**Strengths:** Fast interactive queries; Iceberg full read+write; open-source governance.  
**Weaknesses:** Requires separate cluster (use Amazon EMR or Starburst for managed offering).

---

## 4. Architecture Patterns

### Pattern 1: Debezium → Kafka → Apache Flink → Apache Iceberg
**"The Classic Enterprise Pattern"**

```
Aurora DB ──(binlog/WAL)──▶ Debezium ──▶ Kafka Topic ──▶ Apache Flink ──▶ S3 (Iceberg)
                           (Kafka Connect)              (Iceberg Sink)       │
                                                                              ▼
                                                                      Glue/Hive Catalog
                                                                              │
                                                                              ▼
                                                                    Athena / Trino / Spark
```

**How It Works:**
1. Debezium connector reads Aurora binlog/WAL and publishes change events to Kafka topics (one topic per table, typically)
2. Events include `before`/`after` row images and `op` field (c/u/d/r)
3. Apache Flink consumes Kafka topics, deserializes Debezium events, performs deduplication and transformations
4. Flink's IcebergSink commits data atomically to Iceberg tables using 2-phase commit aligned with Flink checkpoints
5. Iceberg snapshots committed every checkpoint interval (typically 1–5 minutes for batch commits, or per-record with write-ahead log)

**Pros:**
- Sub-minute to sub-second latency depending on Flink checkpoint interval
- Exactly-once semantics via Flink 2PC + Kafka EOS
- Horizontally scalable at every layer
- Kafka retains CDC events for replay/reprocessing
- Kafka becomes the foundation for future event bus
- Richest transformation capability via Flink SQL/DataStream

**Cons:**
- High operational complexity: Aurora config, Kafka cluster, Kafka Connect, Debezium connectors, Flink cluster, Iceberg catalog
- Typically requires 6–8 distinct components
- Initial setup and debugging takes weeks for teams new to this stack
- WAL replication slot on Aurora must be carefully managed

**Known Constraints:**
- Debezium's initial snapshot can take hours for large tables; must handle gracefully
- Flink job restarts require savepoints to maintain state
- Schema changes in Aurora require Debezium schema evolution handling

**Companies Using This Pattern:**
Netflix, LinkedIn, Goldman Sachs, Uber (with variations), many enterprises with Kafka + Iceberg/Hudi investments.

**AWS Deployment:**
- MSK for Kafka (or Redpanda self-managed on EC2)
- MSK Connect for Kafka Connect + Debezium
- Amazon Managed Service for Apache Flink (formerly KDA) for Flink
- S3 + Glue Data Catalog for Iceberg

---

### Pattern 2: Debezium → Redpanda → Flink/Spark → Iceberg
**"The Modern Lightweight Pattern"**

```
Aurora DB ──(binlog/WAL)──▶ Debezium ──▶ Redpanda Topic ──▶ Apache Flink ──▶ S3 (Iceberg)
                                          (Kafka-compatible)   (Iceberg Sink)
```

**How It Works:**
Identical to Pattern 1 except Redpanda replaces Kafka. Since Redpanda is fully Kafka-compatible, zero code changes are needed in Debezium or Flink.

**Pros:**
- 3–6x lower infrastructure cost vs MSK/self-managed Kafka
- 70x lower P99 tail latency at high throughput
- Simpler operations: no ZooKeeper, no JVM tuning, single binary
- Built-in Schema Registry eliminates separate Confluent Schema Registry
- Redpanda Cloud for fully managed option

**Cons:**
- Smaller managed connector ecosystem vs Confluent (some Kafka Connect connectors have edge cases)
- No native AWS managed option (Redpanda Cloud is self-managed on EC2 within AWS)
- If you need Confluent-specific features (Tableflow, KSQL), unavailable
- Smaller community than Kafka for troubleshooting

**Real-World Nuances:**
- Redpanda is a strong choice for new greenfield projects
- For teams already on MSK/Confluent, migration cost may outweigh benefits
- Redpanda's tiered storage (data offloaded to S3) can dramatically reduce broker storage costs for long-retention CDC topics

---

### Pattern 3: AWS DMS → Kinesis → Glue Streaming (Flink) → Iceberg
**"The AWS-Native Pattern"**

```
Aurora DB ──(native CDC)──▶ AWS DMS ──▶ Kinesis Data Streams ──▶ AWS Managed Flink ──▶ S3 (Iceberg)
                            (managed)                              (or Glue Streaming)     │
                                                                                            ▼
                                                                                    Glue Data Catalog
```

**How It Works:**
1. AWS DMS creates a CDC task with Aurora as source and Kinesis Data Streams as target
2. DMS streams row-level changes to Kinesis in JSON format with metadata fields
3. AWS Managed Flink (or Glue Streaming) consumes Kinesis, transforms events, writes to Iceberg
4. Glue Data Catalog registers Iceberg table metadata
5. Athena or Redshift Spectrum queries the Iceberg table

**Etleap's Production Architecture** (per AWS Database Blog, June 2025):
> DMS → Kinesis → Flink on EMR → Iceberg on S3 → Glue Catalog, with Flink 2PC checkpointing every 3 seconds achieving sub-5-second end-to-end latency. Snowflake, Athena, and Redshift all query the same Iceberg tables.

**Pros:**
- Minimal operational overhead — DMS, Kinesis, and Managed Flink are all managed services
- No Kafka or Debezium infrastructure to maintain
- Deep AWS IAM/VPC/CloudWatch integration
- AWS DMS serverless option eliminates replication instance management
- Works well for moderate throughput (up to hundreds of MB/s)

**Cons:**
- Kinesis is not a general-purpose event bus — harder to reuse for microservices events
- DMS event format is less rich than Debezium (limited `before`/`after` images, no schema events)
- Kinesis shard limits require capacity planning
- No Kafka-compatible API — tooling lock-in to AWS
- **Firehose → Iceberg path deprecated September 30, 2025** — use Managed Flink or Glue instead

**Known Constraints:**
- DMS tasks can fall behind during high Aurora write bursts — monitor `CDCLatencySource` CloudWatch metric
- Kinesis shard count must be pre-planned (or use Kinesis On-Demand)
- DMS serverless doesn't support all source/target combinations

**Companies Using This Pattern:**
European bike-sharing company (Etleap case study), various AWS-native analytics teams.

---

### Pattern 4: Debezium → Kafka → Spark → Delta Lake
**"The Databricks Ecosystem Pattern"**

```
Aurora DB ──(binlog/WAL)──▶ Debezium ──▶ Kafka ──▶ Spark Structured Streaming ──▶ S3 (Delta Lake)
                                                                                       │
                                                                                   Databricks
                                                                                  Unity Catalog
```

**How It Works:**
Spark Structured Streaming reads Kafka CDC topics, deserializes Debezium events, and writes to Delta Lake tables using `MERGE INTO`. Delta Lake's Change Data Feed (CDF) enables downstream incremental processing.

**Pros:**
- Best integration with Databricks platform (AutoLoader, Delta Live Tables, Unity Catalog)
- Spark SQL's `MERGE INTO` handles CDC upserts efficiently on Delta
- Delta's Change Data Feed enables incremental downstream pipelines
- Strong Spark/PySpark ecosystem for transformation logic
- Delta Lake's Liquid Clustering (3.0+) for automatic query optimization

**Cons:**
- Databricks costs can be significant at scale
- Delta Lake has limited Athena support (read-only, limited features)
- Less engine-agnostic than Iceberg (Delta outside Databricks/Spark is more limited)
- Spark micro-batch: higher latency than Flink
- Vendor dependencies (Databricks) for best Delta features

**Companies Using This Pattern:**
Databricks customers (thousands of enterprises), companies standardized on the Databricks Lakehouse Platform.

---

### Pattern 5: Airbyte/Fivetran → S3 → dbt → Iceberg
**"The ELT-Style Pattern"**

```
Aurora DB ──(log-based CDC)──▶ Airbyte/Fivetran ──▶ S3 (raw CDC files) ──▶ dbt ──▶ Iceberg tables
                               (managed sync)                                (transforms)
                                                                                    │
                                                                              Glue/Athena
```

**How It Works:**
1. Airbyte or Fivetran syncs CDC changes from Aurora to S3 (as JSON/Parquet files with `_ab_cdc_updated_at`, `_ab_cdc_deleted_at` columns)
2. dbt models read the raw CDC data and apply SCD (Slowly Changing Dimension) logic or MERGE INTO Iceberg tables
3. Iceberg tables on S3 serve as the final queryable data lake

**Pros:**
- Simplest to set up: no Kafka, no Flink, no custom connectors
- dbt provides excellent data transformation and testing framework
- Good for analytics teams comfortable with SQL
- Airbyte is open-source and self-hostable; Fivetran is fully managed
- Cost-effective for small-medium scale

**Cons:**
- **Not real-time:** Airbyte minimum CDC sync is ~5 minutes; Fivetran ~5 minutes
- dbt runs are batch (not streaming) — typical latency 10–60 minutes end-to-end
- Airbyte's Aurora PostgreSQL CDC has caching layer incompatibility
- Fivetran's Iceberg destination doesn't exist (data goes to warehouses)
- The Airbyte Iceberg destination connector is alpha quality as of 2025
- Not suitable for sub-5-minute freshness requirements

**Known Constraints:**
- For Aurora-specific issues: Airbyte recommends disabling Aurora's CDC caching layer for PostgreSQL source
- dbt Iceberg integration requires iceberg-compatible Spark or Athena target

**Best for:** Analytics engineering teams; BI-focused pipelines where hourly/daily freshness is acceptable; teams without streaming infrastructure expertise.

---

### Pattern 6: Redpanda Connect → Redpanda → Redpanda Connect → Iceberg
**"The All-Redpanda Stack"**

```
Aurora DB ──(WAL/binlog)──▶ Redpanda Connect ──▶ Redpanda (broker) ──▶ Redpanda Connect ──▶ S3 (Iceberg)
                            (postgres_cdc /        (Kafka-compatible)   (iceberg sink)
                             mysql_cdc input)
```

**How It Works:**
1. Redpanda Connect runs as a lightweight Go process, reading Aurora WAL/binlog via its native CDC connector
2. Change events are published to Redpanda topics (Kafka-compatible; schema registry built-in)
3. A second Redpanda Connect pipeline (or the same one) consumes from Redpanda topics and writes to Iceberg on S3 via the `iceberg` output connector
4. Bloblang processors handle inline transformations: field filtering, type coercion, routing by table name
5. Iceberg table metadata registered in Glue Data Catalog or a REST catalog

**For simpler pipelines (no fan-out needed):**
```
Aurora DB ──▶ Redpanda Connect (CDC input → Bloblang transform → Iceberg output) ──▶ S3
```
Single process, no broker, direct CDC → Iceberg. Extremely lean.

**Pros:**
- Fewest moving parts of any near-real-time CDC architecture
- No JVM anywhere in the stack — lower memory footprint, faster startup
- Parallel snapshotting dramatically reduces initial load time
- Native Iceberg sink without Flink/Spark cluster
- Single vendor (Redpanda) for broker + connectors + schema registry
- Bloblang is expressive enough for most CDC transformation needs (field rename, filter, type cast, routing)
- Redpanda Cloud provides fully managed broker option

**Cons:**
- No stateful processing: cannot do stream-stream joins, windowed aggregations, or complex deduplication at scale — need Flink for that
- MySQL CDC: GTID not yet supported — binlog position tracking requires external cache (Redis/DynamoDB); less robust for Aurora MySQL failover scenarios
- Iceberg sink less battle-tested than Flink 2PC at very high throughput (millions of events/sec)
- Smaller production reference base than Debezium + Kafka + Flink
- If you eventually need Flink for complex processing, you'll be running Redpanda Connect + Flink anyway, losing the simplicity advantage

**Best for:** Aurora PostgreSQL → Iceberg with simple transformations; teams that prioritize operational simplicity over feature richness; greenfield projects without existing Kafka/Flink investment; moderate scale (tens of thousands of events/sec).

**Demo Reference:** [`jrkinley/redpanda-postgres-cdc-iceberg`](https://github.com/jrkinley/redpanda-postgres-cdc-iceberg) — full docker-compose setup available.

---

## 5. Future Event Bus Expansion

A critical strategic decision when building a CDC pipeline is: **can this same infrastructure grow into a full event bus?** This affects business logic events, microservice communication, real-time notifications, and feature stores.

### 5.1 What Is an Event Bus?

An event bus routes events from multiple producers (not just databases) to multiple consumers. It supports:
- Application events (user signed up, order placed)
- Infrastructure events (deployment completed, alert fired)
- Business events (payment processed, inventory updated)
- CDC events from databases

### 5.2 Broker Suitability for Event Bus Expansion

**Kafka (MSK / Confluent):**
- ✅ **Best ecosystem for CDC + Event Bus**
- Kafka's topic model is identical for CDC events and application events
- Schema Registry enforces contracts across teams
- Kafka Connect, Kafka Streams, ksqlDB, Flink all operate on the same Kafka topics
- Multi-tenancy via ACLs, topic namespacing
- Confluent Tableflow can automatically materialize Kafka topics → Iceberg/Delta tables
- **Evolution path:** CDC topics → Add application event producers → Add consumer groups for microservices, analytics, ML features → No migration required

**Redpanda:**
- ✅ **Strong event bus candidate with Kafka compatibility**
- All Kafka client applications work unchanged
- Built-in Schema Registry
- Better latency makes it attractive for microservices event routing
- Multi-tenancy less mature than Kafka (no Confluent-style RBAC)
- Redpanda Connect (formerly Benthos) for ETL and routing
- **Evolution path:** Similar to Kafka; most microservice Kafka patterns work

**AWS Kinesis:**
- ⚠️ **Limited event bus capability**
- Kinesis is AWS-specific — cannot use standard Kafka clients in microservices
- No schema registry
- Shard model less flexible than Kafka partitions for multi-topic event routing
- Kinesis EventBridge integration provides some routing capability
- **Evolution path:** Kinesis → Amazon EventBridge → SNS/SQS for fan-out; but this is not a unified event bus — it's multiple AWS services stitched together

**NATS JetStream:**
- ✅ **Excellent for microservices event bus, weak for data lake CDC**
- Subject-based routing is very powerful for microservice events
- Multi-tenancy with Accounts is first-class
- However: no Debezium integration, no Flink connector — data lake pipelines require custom bridges
- **Evolution path:** Use NATS for microservice events + Kafka/Redpanda for data CDC; bridge with NATS→Kafka connector

**Apache Pulsar:**
- ✅ **Best multi-tenant event bus, weaker CDC ecosystem**
- Native multi-tenancy with namespaces/topics
- Geo-replication built-in
- Pulsar IO connectors for CDC sources
- **Evolution path:** Pulsar can serve both CDC and event bus but requires more infrastructure

### 5.3 Schema Registry & Governance

For a production event bus, schema governance is critical:

| Schema Registry | Compatible With | Features |
|----------------|----------------|---------|
| Confluent Schema Registry | Kafka, Redpanda | Avro, Protobuf, JSON Schema; compatibility checking |
| Redpanda Built-in SR | Redpanda | Confluent-compatible API |
| AWS Glue Schema Registry | Kafka, Kinesis | Avro, Protobuf, JSON Schema; integrated with Glue |
| Apicurio | Kafka, Debezium | Open-source; full Confluent API compatibility |

**Recommendation:** Use Confluent Schema Registry API (whether hosted on Confluent Cloud, Redpanda, or self-managed Apicurio). This ensures all CDC events and business events share a consistent schema registry, and tools like Debezium, Flink, and Kafka Streams all integrate natively.

### 5.4 Multi-Tenant Event Routing

For organizations with multiple teams/domains:
- **Kafka:** Topic naming conventions (`domain.entity.event`), ACLs per team
- **Redpanda:** Same as Kafka, plus resource quotas per topic
- **Pulsar:** First-class tenant/namespace isolation
- **NATS:** Account-based isolation; subject wildcards for routing

### 5.5 CDC as the Foundation for Event-Driven Architecture

CDC events are low-level, granular database change events. For event-driven microservices, you often want to translate these into higher-level **business events** (e.g., `ORDER_PLACED` instead of `INSERT INTO orders`). Common patterns:

1. **Outbox Pattern:** Application writes to an `outbox` table; CDC captures this and routes to Kafka; downstream services consume business events
2. **Event Enrichment:** Flink or Kafka Streams joins CDC events with reference data to create enriched business events
3. **CDC → Event Translation:** Lambda or Flink translates raw CDC events into domain events before publishing to separate business event topics

The Kafka/Redpanda + Flink stack supports all three patterns natively.

---

## 6. Recommendations

### 6.1 Recommended Stack: Debezium + MSK/Redpanda + Apache Flink + Apache Iceberg + Glue Catalog

```
Aurora DB ──▶ Debezium (MSK Connect) ──▶ MSK or Redpanda ──▶ Amazon Managed Flink ──▶ S3 (Iceberg)
                                          + Schema Registry                                   │
                                                                                        Glue Data Catalog
                                                                                              │
                                                                                     Athena / Trino / Spark
```

**Why this stack:**

1. **Debezium** is the gold standard for log-based CDC. Its rich event format, Aurora support, and Kafka Connect integration make it the most capable connector.

2. **Amazon MSK** (if deeply committed to AWS managed services) or **Redpanda Cloud** (if prioritizing performance and cost) both provide Kafka-compatible brokers with Schema Registry. MSK is the lower-operational-effort choice; Redpanda provides better latency economics.

3. **Amazon Managed Service for Apache Flink** eliminates Flink cluster management while providing exactly-once Iceberg writes via 2PC. The Flink + Iceberg combination is the most mature path to sub-minute CDC latency.

4. **Apache Iceberg** is the best choice for an Aurora → S3 data lake:
   - AWS Athena has full MERGE INTO support (critical for CDC upserts without Spark)
   - Best engine compatibility (Athena, Spark, Trino, Flink, Snowflake, Redshift)
   - Apache Software Foundation governance (no vendor lock-in)
   - Glue Data Catalog is first-class Iceberg catalog on AWS

5. **AWS Glue Data Catalog** as the Iceberg catalog provides seamless integration with Athena, Glue ETL, EMR, and Lake Formation permissions.

### 6.2 Tradeoffs Table

| Decision | Option A | Option B | Option C | Recommendation |
|---------|---------|---------|---------|---------------|
| Connector | Debezium | AWS DMS | Redpanda Connect | **Debezium** for max maturity + event bus; **Redpanda Connect** for simplicity + parallel snapshots (Aurora PG); **DMS** for fully managed |
| Broker | MSK (Kafka) | Redpanda | Kinesis | **MSK** for deep AWS integration; **Redpanda** for better latency/cost on greenfield |
| Stream Processor | Amazon Managed Flink | AWS Glue Streaming | Redpanda Connect (built-in) | **Managed Flink** for <1min latency + complex transforms; **Redpanda Connect** if transformations are simple; **Glue** if 5min+ OK |
| Table Format | Iceberg | Hudi | Delta Lake | **Iceberg** for AWS-native + broad query engine; **Hudi** if you need advanced incremental pipelines |
| Catalog | Glue Data Catalog | Nessie | — | **Glue** for AWS simplicity; **Nessie** if you want Git-like catalog branching |

**Decision tree for connector choice:**
- Aurora PostgreSQL + simple transforms + want simplicity → **Redpanda Connect**
- Aurora MySQL + GTID failover important + complex ecosystem → **Debezium**
- Want fully managed with no infra → **AWS DMS**
- Need stateful stream processing (joins, dedup) → **Debezium or Redpanda Connect** (CDC) **+ Flink** (processing)

### 6.3 What to Avoid

**Avoid:**
- **Amazon Data Firehose → Iceberg path:** Being deprecated September 30, 2025. Do not build new pipelines on this.
- **Airbyte for real-time CDC:** Aurora CDC caching layer incompatibility, alpha Iceberg connector, and 5-minute minimum sync intervals make it unsuitable for real-time data lakes.
- **NATS JetStream as your CDC broker:** No Debezium integration, no Flink connector, no schema registry — requires significant custom work. Use NATS for microservices messaging separately.
- **Delta Lake on AWS without Databricks:** Delta's best features are Databricks-exclusive; Athena support is limited. Iceberg is the better choice for AWS-native architectures.
- **Fivetran for Iceberg data lakes:** Fivetran doesn't have an Iceberg destination; it's built for warehouses (Snowflake, Redshift). You'll be syncing to a warehouse, not a data lake.
- **PostgreSQL replication slots without monitoring:** Unmonitored Aurora PostgreSQL replication slots will accumulate WAL indefinitely if Debezium falls behind, potentially filling Aurora's storage. Always set `max_slot_wal_keep_size` and monitor slot lag.

### 6.4 Migration Path for Future Event Bus

If you start with the **Debezium + MSK/Redpanda + Flink + Iceberg** stack:

**Phase 1 (CDC):** Aurora → Debezium → Kafka → Flink → Iceberg  
**Phase 2 (Event Bus):** Add application event producers to the same Kafka cluster; different topics for business events  
**Phase 3 (Enrichment):** Flink joins CDC events with application events; enriched events published to new topics  
**Phase 4 (Reverse ETL):** Iceberg tables serve as ML feature store input; Flink publishes feature updates back to Kafka for online serving  

No migration required — the same infrastructure scales from CDC to full event bus to streaming feature platform.

### 6.5 Cost Estimates (Rough Guidance)

For a medium-scale Aurora database (100 GB, 10K writes/second):

| Component | Option | Estimated Monthly Cost |
|-----------|--------|----------------------|
| CDC Connector | Debezium on MSK Connect | ~$50–100/mo (compute) |
| Message Broker | MSK Serverless | ~$200–500/mo |
| Message Broker | Redpanda Cloud | ~$150–300/mo |
| Stream Processor | Amazon Managed Flink (2 KPU) | ~$200–400/mo |
| Stream Processor | AWS Glue Streaming | ~$100–300/mo |
| Storage | S3 (10TB Iceberg) | ~$230/mo |
| Query | Athena ($5/TB scanned, Iceberg with metadata pruning) | ~$50–200/mo |

**Total range:** ~$800–1,700/month for a fully managed stack vs ~$300–600/month self-managed (Redpanda + self-managed Flink on EKS).

---

## 7. References

1. **Dremio Blog** - "A Guide to Change Data Capture (CDC) with Apache Iceberg" (October 3, 2024)  
   https://www.dremio.com/blog/cdc-with-apache-iceberg/

2. **AWS Database Blog** - "Real-time Iceberg ingestion with AWS DMS" (June 2025, Etleap/AWS)  
   https://aws.amazon.com/blogs/database/real-time-iceberg-ingestion-with-aws-dms/

3. **Onehouse Blog** - "Apache Iceberg™ vs Delta Lake vs Apache Hudi™ - Feature Comparison Deep Dive" (October 2025)  
   https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-vs-apache-iceberg-lakehouse-feature-comparison

4. **Fivetran** - "The top 8 change data capture (CDC) tools and solutions for 2025" (September 2024)  
   https://www.fivetran.com/learn/cdc-tools-2024

5. **Streamkap** - "Why Apache Iceberg? A Guide to Real-Time Data Lakes in 2025"  
   https://streamkap.com/blog/apache-iceberg-guide/

6. **DEV Community** - "RIP Amazon Data Firehose Change Data Capture" (2025)  
   https://dev.to/aws-builders/rip-amazon-data-firehose-change-data-capture-2dg3

7. **Redpanda** - "Redpanda vs. Apache Kafka (TCO Analysis)"  
   https://www.redpanda.com/blog/is-redpanda-better-than-kafka-tco-comparison

8. **Onehouse Blog** - "Apache Kafka® (Kafka Connect) vs. Apache Flink® vs. Apache Spark" (2024)  
   https://www.onehouse.ai/blog/kafka-connect-vs-flink-vs-spark-choosing-the-right-ingestion-framework

9. **Synadia** - "NATS and Kafka Compared"  
   https://www.synadia.com/blog/nats-and-kafka-compared

10. **Airbyte Documentation** - "Postgres Connector / Aurora CDC"  
    https://docs.airbyte.com/integrations/sources/postgres

11. **Estuary** - "Amazon Aurora for PostgreSQL to Amazon S3 Iceberg"  
    https://estuary.dev/integrations/amazon-aurora-postgres-to-s3-iceberg/

12. **AWS Big Data Blog** - "Modernize your legacy databases with AWS data lakes, Part 2" (November 2024)  
    https://aws.amazon.com/blogs/big-data/modernize-your-legacy-databases-with-aws-data-lakes-part-2-build-a-data-lake-using-aws-dms-data-on-apache-iceberg/

13. **Medium** - "Inside a Production-Grade CDC Pipeline with Debezium and Kafka"  
    https://medium.com/@kanishks772/inside-a-production-grade-cdc-pipeline-with-debezium-and-kafka-a2897b5d28a2

14. **Redpanda** - "Kafka, Redpanda, and Pulsar: Choosing the Right Streaming Platform 2026"  
    https://www.linkedin.com/pulse/kafka-redpanda-pulsar-choosing-right-streaming-2026-ashraf-kanyadakam-uehdf

15. **AWS Samples GitHub** - "Transactional Data Lake using Amazon DataFirehose Iceberg"  
    https://github.com/aws-samples/transactional-datalake-using-amazon-datafirehose-iceberg

---

*Report generated: March 2025. Technology landscape evolves rapidly — verify current versions and pricing before production deployment.*
