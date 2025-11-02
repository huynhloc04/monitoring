# Kafka Connect & Debezium CDC

## Overview

This folder contains the **Kafka Connect** ecosystem with **Debezium** for MySQL Change Data Capture (CDC). It captures database changes in real-time and streams them to Kafka topics.

The setup includes:

- **Debezium Connect**: MySQL CDC connector with JDBC support
- **Schema Registry**: Avro schema management
- **Control Center**: Confluent's web UI for connector management
- **Custom SMT**: Single Message Transform for data cleaning

---

## Table of Contents

- [Architecture](#architecture)
- [What is CDC (Change Data Capture)?](#what-is-cdc-change-data-capture)
- [Components](#components)
- [How It Works](#how-it-works)
- [Configuration Details](#configuration-details)
- [Custom SMT (Single Message Transform)](#custom-smt-single-message-transform)
- [How to Use](#how-to-use)
- [Connector Configuration](#connector-configuration)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  SOURCE DATABASE (MySQL RDS)                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  MySQL Database with Binary Logging Enabled            │ │
│  │  User: debezium (with REPLICATION privileges)          │ │
│  │  Database: trimhealthymama                             │ │
│  └────────────────┬───────────────────────────────────────┘ │
└───────────────────┼──────────────────────────────────────────┘
                    │ Binary Log Stream (Binlog)
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  DEBEZIUM CONNECT (Port 8083)                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. CONNECTOR (MySQL CDC)                              │ │
│  │     - Reads binlog events                              │ │
│  │     - Parses INSERT/UPDATE/DELETE                      │ │
│  │     - Tracks database schema changes                   │ │
│  │     - Maintains binlog position                        │ │
│  └───────────────┬────────────────────────────────────────┘ │
│                  ↓                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  2. TRANSFORMS (SMT Pipeline)                          │ │
│  │     ┌────────────────┐  ┌──────────────────┐          │ │
│  │     │  RegexRouter   │→ │  Custom Modify   │          │ │
│  │     │  (Topic route) │  │  (Clean nulls)   │          │ │
│  │     └────────────────┘  └──────────────────┘          │ │
│  └───────────────┬────────────────────────────────────────┘ │
│                  ↓                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  3. CONVERTERS (Serialization)                         │ │
│  │     - Schema validation via Schema Registry            │ │
│  │     - Avro serialization (key + value)                 │ │
│  └───────────────┬────────────────────────────────────────┘ │
└───────────────────┼──────────────────────────────────────────┘
                    │ CDC Events (Avro)
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  KAFKA BROKERS                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Topics: monitor-4_tablename                           │ │
│  │  Format: Avro (with schema evolution support)          │ │
│  │  Retention: 7 days                                     │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓ Consumers read CDC events
┌─────────────────────────────────────────────────────────────┐
│  DOWNSTREAM APPLICATIONS                                     │
│  - Data warehouses (Snowflake, BigQuery)                    │
│  - Search engines (Elasticsearch)                           │
│  - Caching layers (Redis)                                   │
│  - Analytics platforms                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  SUPPORTING SERVICES                                         │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────────┐   │
│  │ Schema Reg.   │  │ Control Center │  │ JMX Exporter │   │
│  │ (Port 8081)   │  │  (Port 9021)   │  │ (Port 5000)  │   │
│  └───────────────┘  └────────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## What is CDC (Change Data Capture)?

### Definition

**CDC** is a design pattern that captures changes (inserts, updates, deletes) from a database and makes them available to downstream systems in real-time.

### Why Use CDC?

Traditional approaches to data synchronization:

- **Batch ETL**: Queries database periodically → High latency, impacts DB performance
- **Triggers**: Database triggers write to change tables → Adds overhead, hard to maintain
- **Dual writes**: Application writes to DB and Kafka → Can lose consistency

**CDC Benefits**:

- ✅ **Real-time**: Changes captured as they happen (millisecond latency)
- ✅ **Non-invasive**: No application changes needed
- ✅ **Low impact**: Reads from binlog (no queries on production DB)
- ✅ **Complete history**: Captures all changes (even from other applications)
- ✅ **Schema evolution**: Automatically adapts to schema changes

### How Debezium CDC Works

1. **Snapshot Phase** (first run):

   - Takes consistent snapshot of database schema
   - Records binlog position
   - **Does NOT copy existing data** (configured as `snapshot.mode=schema_only`)
2. **Streaming Phase** (continuous):

   - Connects to MySQL binlog stream
   - Reads binlog events in order
   - Converts to CDC events
   - Publishes to Kafka topics
3. **Failure Recovery**:

   - Stores binlog position in Kafka topic `__consumer_offsets`
   - On restart, resumes from last committed position
   - No data loss or duplication

---

## Components

### 1. Debezium Connect

**Purpose**: Kafka Connect worker with Debezium MySQL connector installed.

**Image**: `debezium/connect:1.9`

**Key Features**:

- MySQL CDC connector (captures binlog changes)
- JDBC connector (for sink operations)
- Custom SMT support
- JMX metrics export

**Startup Process**:

```bash
# 1. Downloads JDBC driver (Confluent Kafka Connect JDBC)
curl -o /tmp/jdbc_driver.zip "https://hub-downloads.confluent.io/..."

# 2. Extracts and installs to /kafka/libs/
unzip -o /tmp/jdbc_driver.zip -d /tmp/jdbc_driver
cp -f /tmp/jdbc_driver/confluentinc-kafka-connect-jdbc-10.8.2/lib/* /kafka/libs/

# 3. Loads custom SMT from volume mount
# /kafka/libs/modify-1.0-SNAPSHOT.jar

# 4. Starts Connect worker
/docker-entrypoint.sh start
```

**Configuration**:

```yaml
BOOTSTRAP_SERVERS: "broker-1:9092,broker-2:9092,broker-3:9092"
GROUP_ID: 1                                  # Connect cluster ID
CONFIG_STORAGE_TOPIC: "connect_configs"      # Stores connector configs
OFFSET_STORAGE_TOPIC: "connect_offsets"      # Stores source offsets (binlog position)
CONNECT_TOPIC_CREATION_ENABLE: true          # Auto-create topics
KAFKA_PRODUCER_MAX_REQUEST_SIZE: 20971520    # 20MB (for large rows)
```

**Why these settings?**

- **CONFIG_STORAGE_TOPIC**: Persists connector configurations (replicated across cluster)
- **OFFSET_STORAGE_TOPIC**: Tracks binlog position for each connector (critical for recovery)
- **TOPIC_CREATION_ENABLE**: Automatically creates topics when new tables are added
- **MAX_REQUEST_SIZE**: Allows large messages (e.g., database rows with BLOB/TEXT fields)

### 2. Schema Registry

**Purpose**: Centralized schema management for Avro serialization.

**Image**: `confluentinc/cp-schema-registry:7.5.0`

**How It Works**:

1. Producer (Debezium) sends schema to Registry
2. Registry validates and assigns schema ID
3. Producer serializes data with schema ID
4. Consumer retrieves schema by ID to deserialize

**Benefits**:

- **Schema Evolution**: Add/remove fields without breaking consumers
- **Validation**: Ensures data conforms to schema
- **Efficiency**: Schema stored once, messages only contain ID
- **Compatibility**: Enforces compatibility rules (backward, forward, full)

**Configuration**:

```yaml
SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'broker-1:9092,broker-2:9092,broker-3:9092'
SCHEMA_REGISTRY_LISTENERS: "http://0.0.0.0:8081"
```

**API Endpoints**:

```bash
# List subjects (topics)
curl http://localhost:8081/subjects

# Get schema versions
curl http://localhost:8081/subjects/monitor-4_users-value/versions

# Get specific schema
curl http://localhost:8081/subjects/monitor-4_users-value/versions/1
```

### 3. Control Center

**Purpose**: Confluent's web UI for managing and monitoring Kafka Connect.

**Image**: `confluentinc/cp-enterprise-control-center:7.5.0`

**Features**:

- View connector status
- Create/edit/delete connectors
- Monitor consumer lag
- View topic data
- Manage schemas

**Configuration**:

```yaml
CONTROL_CENTER_BOOTSTRAP_SERVERS: 'broker-1:9092,broker-2:9092,broker-3:9092'
CONTROL_CENTER_CONNECT_CONNECT-DEFAULT_CLUSTER: 'debezium:8083'
CONTROL_CENTER_SCHEMA_REGISTRY_URL: "http://schema-registry:8081"
```

**Access**: http://localhost:9021

### 4. Custom SMT (Single Message Transform)

**Purpose**: Transform CDC messages before publishing to Kafka.

**Location**: `smt/target/modify-1.0-SNAPSHOT.jar`

**Functionality**: Removes null bytes (`\x00`) from string fields to prevent serialization errors.

**See**: [Custom SMT section](#custom-smt-single-message-transform) for details.

---

## How It Works

### Complete Data Flow

```
┌──────────────────────────────────────────────────────────────┐
│  STEP 1: Database Change                                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  MySQL Query: UPDATE users SET name='John' WHERE id=1 │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 2: Binlog Event                                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Binlog writes:                                        │  │
│  │  - Event type: UPDATE                                  │  │
│  │  - Table: users                                        │  │
│  │  - Before: {id:1, name:'Jane', ...}                    │  │
│  │  - After:  {id:1, name:'John', ...}                    │  │
│  │  - Timestamp: 2024-11-01 10:30:00                      │  │
│  │  - Binlog position: mysql-bin.000042:1234              │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 3: Debezium Reads Binlog                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Connector polls binlog stream                        │  │
│  │  Parses UPDATE event                                   │  │
│  │  Constructs CDC event structure                       │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 4: Apply Transforms (SMT)                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  A. RegexRouter: Determine topic name                 │  │
│  │     Input:  "monitor-4.trimhealthymama.users"          │  │
│  │     Regex:  "([^.]+)\\.([^.]+)\\.([^.]+)"              │  │
│  │     Output: "monitor-4_users"                          │  │
│  │                                                        │  │
│  │  B. Custom Modify: Clean null bytes                   │  │
│  │     Input:  {name: "John\x00Doe"}                      │  │
│  │     Output: {name: "JohnDoe"}                          │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 5: Serialize to Avro                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Generate Avro schema from table structure         │  │
│  │  2. Register schema in Schema Registry (get ID)       │  │
│  │  3. Serialize key (primary key fields)                │  │
│  │  4. Serialize value (before + after + metadata)       │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 6: Publish to Kafka                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Topic: monitor-4_users                                │  │
│  │  Key: {id: 1} (Avro)                                   │  │
│  │  Value:                                                │  │
│  │  {                                                     │  │
│  │    "before": {"id":1, "name":"Jane", ...},             │  │
│  │    "after": {"id":1, "name":"John", ...},              │  │
│  │    "source": {                                         │  │
│  │      "version": "1.9",                                 │  │
│  │      "connector": "mysql",                             │  │
│  │      "name": "monitor-4",                              │  │
│  │      "ts_ms": 1698843000000,                           │  │
│  │      "db": "trimhealthymama",                          │  │
│  │      "table": "users",                                 │  │
│  │      "file": "mysql-bin.000042",                       │  │
│  │      "pos": 1234                                       │  │
│  │    },                                                  │  │
│  │    "op": "u",  // operation: c=create, u=update, d=delete │
│  │    "ts_ms": 1698843000500                              │  │
│  │  }                                                     │  │
│  └───────────────────┬────────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 7: Store Offset                                        │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Debezium commits offset to connect_offsets topic:    │  │
│  │  {                                                     │  │
│  │    "connector": "monitor-4",                           │  │
│  │    "file": "mysql-bin.000042",                         │  │
│  │    "pos": 1234,                                        │  │
│  │    "row": 1,                                           │  │
│  │    "snapshot": false                                   │  │
│  │  }                                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Configuration Details

### Connector Configuration (`connector.json`)

```json
{
  "name": "monitor-4",
  "config": {
    // ===== CONNECTOR CLASS =====
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",  // Number of parallel tasks (1 for single DB)

    // ===== DATABASE CONNECTION =====
    "database.hostname": "mysql.com",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "password",
    "database.server.name": "monitor-4",    // Logical name (used in topic prefix)
    "database.server.id": "4",              // Unique ID for this connector (MySQL requirement)

    // ===== SNAPSHOT CONFIGURATION =====
    "snapshot.mode": "schema_only",         // Options: initial, schema_only, never
    "snapshot.locking.mode": "none",        // Options: minimal, extended, none

    // ===== BINLOG CONFIGURATION =====
    "time.precision.mode": "connect",       // How to represent timestamps

    // ===== SCHEMA HISTORY =====
    "database.history.store.only.captured.tables.ddl": "true",
    "database.history.kafka.bootstrap.servers": "broker-1:9092,broker-2:9092,broker-3:9092",
    "database.history.kafka.topic": "dbhistory.thm_multisite.monitor-4",

    // ===== DELETION HANDLING =====
    "delete.handling.mode": "rewrite",      // Options: none, rewrite
    "tombstones.on.delete": "true",         // Emit tombstone after delete

    // ===== FILTERING =====
    "table.include.list": "trimhealthymama.*",      // Only capture these tables
    "database.include.list": "trimhealthymama",     // Only capture this database
    "include.schema.changes": "true",               // Capture DDL changes

    // ===== SERIALIZATION =====
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter.schema.registry.url": "http://schema-registry:8081",

    // ===== TRANSFORMS =====
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$1_$3"
  }
}
```

### Configuration Explained

#### Snapshot Mode

| Mode            | Behavior                                           | Use Case                               |
| --------------- | -------------------------------------------------- | -------------------------------------- |
| `initial`     | Takes full snapshot of tables, then streams binlog | First-time setup, need existing data   |
| `schema_only` | Only captures schema, skips data snapshot          | Existing system, only need new changes |
| `never`       | Starts from current binlog position                | Testing, or when restarting connector  |

**This project uses `schema_only`** because:

- Existing data is already in the target system
- Only need real-time changes going forward
- Avoids long-running snapshot on large tables

#### Snapshot Locking Mode

| Mode         | Behavior                             | Impact                                  |
| ------------ | ------------------------------------ | --------------------------------------- |
| `minimal`  | Locks tables briefly during snapshot | Brief write blocking                    |
| `extended` | Locks tables for entire snapshot     | Long write blocking (avoid)             |
| `none`     | No locks at all                      | No impact, but may have inconsistencies |

**This project uses `none`** because:

- `snapshot.mode=schema_only` doesn't copy data anyway
- No risk of production impact

#### Delete Handling Mode

**Problem**: Deletes in databases remove the row, but Kafka consumers need to know what was deleted.

**Solutions**:

1. `delete.handling.mode=none`: Only emit tombstone (no data in message)
2. `delete.handling.mode=rewrite`: Emit full delete event with "before" data, then tombstone

**This project uses `rewrite`** because:

- Consumers can see what was deleted (important for auditing)
- Tombstone allows Kafka log compaction to work correctly

**Example Delete Event**:

```json
{
  "before": {"id": 1, "name": "John", ...},  // Row that was deleted
  "after": null,                              // No "after" for deletes
  "op": "d",                                  // operation: delete
  "ts_ms": 1698843000000
}
```

#### Topic Naming Transform

**Default Debezium topic format**: `<server-name>.<database>.<table>`

- Example: `monitor-4.trimhealthymama.users`

**After RegexRouter transform**: `<server-name>_<table>`

- Example: `monitor-4_users`

**Regex Explanation**:

```regex
Pattern:     ([^.]+)\.([^.]+)\.([^.]+)
             └─$1──┘ └─$2──┘ └─$3──┘
             server  database table

Replacement: $1_$3
Result:      monitor-4_users
```

**Why simplify topic names?**

- Shorter, cleaner names
- Easier to work with in downstream systems
- Database name is redundant (already filtered to single DB)

---

## Custom SMT (Single Message Transform)

### Purpose

Removes **null bytes** (`\x00`) from string fields, which can cause issues with:

- Avro serialization
- JSON converters
- Downstream systems (Postgres, Elasticsearch)

### Java Implementation

**Location**: `smt/src/main/java/org/apache/kafka/connect/transforms/modify.java`

**Key Methods**:

```java
public abstract class modify<R extends ConnectRecord<R>> implements Transformation<R> {
  
  @Override
  public void configure(Map<String, ?> props) {
    // Read configuration: which fields to clean
    String fieldsList = config.getString(ConfigName.FIELDS_TO_CLEAN);
    fieldsToClean = fieldsList.split(",");
  }

  @Override
  public R apply(R record) {
    // If record has schema (Avro), clean with schema
    if (operatingSchema(record) == null) {
      return applySchemaless(record);
    } else {
      return applyWithSchema(record);
    }
  }

  private R applyWithSchema(R record) {
    final Struct value = requireStruct(operatingValue(record), PURPOSE);
    final Struct updatedValue = new Struct(value.schema());

    // Copy all fields
    for (Field field : value.schema().fields()) {
      Object fieldValue = value.get(field);
    
      // Clean specified string fields
      if (fieldValue instanceof String && contains(fieldsToClean, field.name())) {
        String cleanedValue = ((String) fieldValue).replaceAll("\\x00", "");
        updatedValue.put(field.name(), cleanedValue);
      } else {
        updatedValue.put(field.name(), fieldValue);
      }
    }

    return newRecord(record, updatedSchema, updatedValue);
  }
}
```

### Maven Build

**Build the JAR**:

```bash
cd smt
mvn clean package
```

**Output**: `target/modify-1.0-SNAPSHOT.jar`

**Usage in Connector**:

```json
"transforms": "route,clean",
"transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
"transforms.route.replacement": "$1_$3",
"transforms.clean.type": "org.apache.kafka.connect.transforms.modify$Value",
"transforms.clean.field.name": "description,notes,comments"
```

---

## How to Use

### Step 1: Prerequisites

Ensure Kafka cluster is running:

```bash
cd ../cluster
docker-compose up -d
```

Ensure MySQL is configured for CDC:

```sql
-- Enable binlog (as root)
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- Create Debezium user
CREATE USER 'debezium'@'%' IDENTIFIED BY 'your_password';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
  ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;

-- Verify binlog is enabled
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
```

### Step 2: Start Connect Services

```bash
cd connect
docker-compose up -d
```

**Wait for services to be healthy**:

```bash
# Watch logs
docker-compose logs -f

# Check health
curl http://localhost:8083/connectors         # Debezium Connect
curl http://localhost:8081/subjects           # Schema Registry
```

### Step 3: Configure Connector

Edit `connector.json` with your database details:

```json
{
  "name": "my-mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "your-mysql-host",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "your_password",
    "database.server.name": "my-server",
    "database.server.id": "1",
    "table.include.list": "mydb.*",
    "database.include.list": "mydb"
  }
}
```

### Step 4: Deploy Connector

**Via REST API**:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector.json
```

**Via Control Center**:

1. Open http://localhost:9021
2. Navigate to "Connect" → "connect-default"
3. Click "Add connector"
4. Paste connector configuration

### Step 5: Verify CDC is Working

**Check connector status**:

```bash
curl http://localhost:8083/connectors/my-mysql-connector/status
```

**Expected output**:

```json
{
  "name": "my-mysql-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "debezium:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "debezium:8083"
    }
  ]
}
```

**Check topics were created**:

```bash
docker exec broker-1 kafka-topics --list --bootstrap-server localhost:9092 | grep my-server
```

**Consume messages**:

```bash
docker exec broker-1 kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic my-server_users \
  --from-beginning \
  --property print.key=true
```

### Step 6: Test with Database Change

**Make a change in MySQL**:

```sql
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
```

**Verify event appears in Kafka**:

```bash
docker exec broker-1 kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic my-server_users \
  --from-beginning \
  --max-messages 1
```

---

## Monitoring

### Debezium Metrics

**JMX Exporter endpoint**:

```bash
curl http://localhost:5000/metrics
```

**Key Metrics**:

```bash
# Connection status (1 = connected, 0 = disconnected)
debezium_metrics_Connected{context="streaming"}

# Number of changes applied
debezium_metrics_ChangesApplied

# Lag (milliseconds behind source)
debezium_metrics_MilliSecondsSinceLastAppliedChange

# Total events seen
debezium_metrics_TotalNumberOfEventsSeen

# Error count
debezium_metrics_NumberOfErroneousEvents

# Disconnection count
debezium_metrics_NumberOfDisconnects
```

### REST API

**List connectors**:

```bash
curl http://localhost:8083/connectors
```

**Get connector status**:

```bash
curl http://localhost:8083/connectors/my-connector/status
```

**Get connector config**:

```bash
curl http://localhost:8083/connectors/my-connector/config
```

**Restart connector**:

```bash
curl -X POST http://localhost:8083/connectors/my-connector/restart
```

**Pause connector**:

```bash
curl -X PUT http://localhost:8083/connectors/my-connector/pause
```

**Resume connector**:

```bash
curl -X PUT http://localhost:8083/connectors/my-connector/resume
```

**Delete connector**:

```bash
curl -X DELETE http://localhost:8083/connectors/my-connector
```

---

## Troubleshooting

### Problem 1: Connector Fails to Start

**Symptoms**:

- Status shows "FAILED"
- Error in logs

**Common Causes**:

**A. MySQL binlog not enabled**:

```bash
# Check MySQL
mysql> SHOW VARIABLES LIKE 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |  <-- Should be ON
+---------------+-------+

# Fix: Add to MySQL config (my.cnf)
[mysqld]
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
server_id = 1
```

**B. Insufficient permissions**:

```bash
# Check grants
mysql> SHOW GRANTS FOR 'debezium'@'%';

# Required grants:
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
```

**C. Network connectivity**:

```bash
# Test from Debezium container
docker exec debezium telnet your-mysql-host 3306
```

**D. Wrong credentials**:

```bash
# Check connector logs
docker logs debezium | grep -i "authentication"
```

### Problem 2: No Events Appearing in Kafka

**Symptoms**:

- Connector shows RUNNING
- No messages in Kafka topics

**Solutions**:

**A. Check if tables match filter**:

```json
// Ensure table.include.list matches your tables
"table.include.list": "mydb.*"  // All tables in mydb
"table.include.list": "mydb.users,mydb.orders"  // Specific tables
```

**B. Verify binlog has events**:

```sql
-- Check binlog files
SHOW BINARY LOGS;

-- View binlog events
SHOW BINLOG EVENTS IN 'mysql-bin.000042' LIMIT 10;
```

**C. Check connector offset**:

```bash
# View connect_offsets topic
docker exec broker-1 kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic connect_offsets \
  --from-beginning
```

**D. Make a test change**:

```sql
INSERT INTO users (name) VALUES ('test');
```

### Problem 3: High Lag

**Symptoms**:

- `debezium_metrics_MilliSecondsSinceLastAppliedChange` is high
- Connector is behind MySQL

**Solutions**:

**A. Increase task parallelism** (only works for multi-table):

```json
"tasks.max": "4"  // One task per table (max)
```

**B. Tune queue size**:

```json
"max.queue.size": "16384",        // Increase buffer
"max.batch.size": "4096"          // Process more per batch
```

**C. Check Kafka broker performance**:

```bash
# Monitor broker metrics
curl http://localhost:5004/metrics | grep kafka_server_BrokerTopicMetrics
```

**D. Check network latency**:

```bash
# Ping from Debezium to MySQL
docker exec debezium ping -c 10 your-mysql-host
```

### Problem 4: Schema Registry Errors

**Symptoms**:

- "Failed to serialize Avro data"
- "Schema not found"

**Solutions**:

**A. Check Schema Registry is reachable**:

```bash
curl http://localhost:8081/subjects
```

**B. Check connector converter config**:

```json
"key.converter": "io.confluent.connect.avro.AvroConverter",
"value.converter": "io.confluent.connect.avro.AvroConverter",
"key.converter.schema.registry.url": "http://schema-registry:8081",
"value.converter.schema.registry.url": "http://schema-registry:8081"
```

**C. View registered schemas**:

```bash
# List subjects
curl http://localhost:8081/subjects

# Get schema
curl http://localhost:8081/subjects/my-server_users-value/versions/1
```

**D. Delete corrupt schema** (if needed):

```bash
# Delete subject
curl -X DELETE http://localhost:8081/subjects/my-server_users-value
```

### Problem 5: Connector Disconnects Frequently

**Symptoms**:

- `debezium_metrics_NumberOfDisconnects` increases
- Connector state flips between RUNNING and FAILED

**Solutions**:

**A. Increase connection timeout**:

```json
"connect.timeout.ms": "30000",
"database.query.timeout.ms": "600000"
```

**B. Enable binlog heartbeat**:

```json
"heartbeat.interval.ms": "5000",
"heartbeat.topics.prefix": "__debezium-heartbeat"
```

**C. Check MySQL connection limits**:

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;
```

**D. Check AWS RDS maintenance window** (if using RDS):

- RDS maintenance can drop connections
- Schedule connector restart after maintenance

---

## Production Considerations

### 1. Database Setup

**MySQL Configuration**:

```ini
[mysqld]
# Binlog settings
log_bin = mysql-bin
binlog_format = ROW
binlog_row_image = FULL
binlog_expire_logs_seconds = 604800  # 7 days retention
max_binlog_size = 1G

# Performance
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

# Replication
server_id = 1  # Unique per MySQL instance
```

**User Permissions**:

```sql
CREATE USER 'debezium'@'%' IDENTIFIED BY 'secure_password';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
  ON *.* TO 'debezium'@'%';

-- For specific database only
GRANT SELECT ON mydb.* TO 'debezium'@'%';

FLUSH PRIVILEGES;
```

### 2. Connector Tuning

**For High Throughput**:

```json
"max.batch.size": "8192",
"max.queue.size": "16384",
"poll.interval.ms": "100"
```

**For Low Latency**:

```json
"max.batch.size": "1024",
"poll.interval.ms": "10"
```

**For Large Rows**:

```json
"max.request.size": "20971520",  // 20MB
"database.query.timeout.ms": "600000"  // 10 minutes
```

### 3. Schema Evolution

**Enable compatible schema evolution**:

```json
"key.converter.schema.registry.url": "http://schema-registry:8081",
"value.converter.schema.registry.url": "http://schema-registry:8081",
"key.converter.use.latest.version": "true",
"value.converter.use.latest.version": "true"
```

**Schema Registry compatibility modes**:

```bash
# Set backward compatibility (can add fields)
curl -X PUT http://localhost:8081/config/my-server_users-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD"}'
```

### 4. Monitoring and Alerts

**Critical Metrics to Alert On**:

- `debezium_metrics_Connected == 0`: Connector disconnected
- `rate(debezium_metrics_ChangesApplied[5m]) == 0`: No changes processing
- `debezium_metrics_MilliSecondsSinceLastAppliedChange > 30000`: High lag (>30s)
- `debezium_metrics_NumberOfErroneousEvents > 0`: Errors occurring

### 5. Disaster Recovery

**Backup critical topics**:

```bash
# Backup connect_offsets (contains binlog position)
docker exec broker-1 kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic connect_offsets \
  --from-beginning > connect_offsets_backup.json
```

**Re-snapshot strategy**:

```json
// Option 1: Snapshot all data
"snapshot.mode": "initial"

// Option 2: Snapshot new tables only
"snapshot.mode": "when_needed"

// Option 3: Skip snapshot
"snapshot.mode": "never"
```

### 6. Security

**Enable SSL for MySQL connection**:

```json
"database.ssl.mode": "required",
"database.ssl.keystore": "/path/to/keystore.jks",
"database.ssl.keystore.password": "keystore_password",
"database.ssl.truststore": "/path/to/truststore.jks",
"database.ssl.truststore.password": "truststore_password"
```

**Enable authentication for Schema Registry**:

```json
"key.converter.basic.auth.credentials.source": "USER_INFO",
"key.converter.basic.auth.user.info": "username:password",
"value.converter.basic.auth.credentials.source": "USER_INFO",
"value.converter.basic.auth.user.info": "username:password"
```

---

## Additional Resources

- [Debezium Documentation](https://debezium.io/documentation/reference/stable/)
- [Debezium MySQL Connector](https://debezium.io/documentation/reference/stable/connectors/mysql.html)
- [Kafka Connect Documentation](https://kafka.apache.org/documentation/#connect)
- [Schema Registry Documentation](https://docs.confluent.io/platform/current/schema-registry/index.html)
- [Single Message Transforms](https://kafka.apache.org/documentation/#connect_transforms)

---

**Happy CDC Streaming! 🚀**
