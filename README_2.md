# Comprehensive Change Data Capture Platform

## Overview

This is a comprehensive, production-ready Change Data Capture (CDC) platform built on Apache Kafka and Debezium. This platform captures database changes in real-time and streams them to Kafka topics, with full monitoring, alerting, and visualization capabilities.

This system is designed for enterprise-scale data synchronization, event streaming, and real-time data pipelines.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Components](#system-components)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [System Architecture Diagram](#system-architecture-diagram)
- [Project Structure](#project-structure)
- [Configuration Details](#configuration-details)
- [Monitoring and Alerting](#monitoring-and-alerting)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Architecture Overview

The Barall CDC platform consists of three main subsystems:

1. **Kafka Cluster** (`cluster/`): A highly available Apache Kafka cluster running in KRaft mode (no Zookeeper)
2. **Kafka Connect & Debezium** (`connect/`): CDC pipeline for capturing database changes
3. **Monitoring Stack** (`monitor/`): Full observability with Prometheus, Grafana, and Alertmanager

### Key Features

✅ **High Availability**: 3-node Kafka controller quorum + 3 brokers
✅ **CDC Streaming**: Real-time MySQL change data capture with Debezium
✅ **Schema Management**: Confluent Schema Registry with Avro serialization
✅ **Custom Transformations**: SMT (Single Message Transform) for data cleaning
✅ **Full Observability**: Metrics, alerts, and dashboards
✅ **Production Ready**: JMX monitoring, alerting, and auto-recovery

---

## System Components

### 1. Kafka Cluster (KRaft Mode)

| Component             | Count | Description                                  |
| --------------------- | ----- | -------------------------------------------- |
| **Controllers** | 3     | Manage cluster metadata (replaces Zookeeper) |
| **Brokers**     | 3     | Handle data storage and client requests      |
| **Kafka UI**    | 1     | Web interface for cluster management         |

**Ports:**

- Controllers: 5001, 5002, 5003 (JMX Exporter), 9101, 9102, 9103 (JMX)
- Brokers: 9093, 9095, 9096 (External), 5004, 5005, 5006 (JMX Exporter)
- Kafka UI: 8080

### 2. Kafka Connect & Debezium

| Component                  | Count | Description                              |
| -------------------------- | ----- | ---------------------------------------- |
| **Debezium Connect** | 1     | MySQL CDC connector with JDBC support    |
| **Schema Registry**  | 1     | Avro schema management                   |
| **Control Center**   | 1     | Confluent management UI                  |
| **Custom SMT**       | 1     | Data transformation (removes null bytes) |

**Ports:**

- Debezium: 8083 (REST API), 5000 (JMX Exporter)
- Schema Registry: 8081
- Control Center: 9021

### 3. Monitoring Stack

| Component               | Description                    |
| ----------------------- | ------------------------------ |
| **Prometheus**    | Metrics collection and storage |
| **Grafana**       | Visualization dashboards       |
| **Alertmanager**  | Alert routing and grouping     |
| **Calert**        | Google Chat webhook dispatcher |
| **Node Exporter** | System metrics collection      |

**Ports:**

- Prometheus: 9090
- Grafana: 3000
- Alertmanager: 9094
- Calert: 6000
- Node Exporter: 9100

---

## Prerequisites

Before setting up the platform, ensure you have:

- **Docker** (version 20.10 or higher)
- **Docker Compose** (version 2.0 or higher)
- **Linux/macOS/Windows with WSL2**
- **Minimum Resources:**
  - 16 GB RAM
  - 4 CPU cores
  - 20 GB free disk space
- **Network Access:**
  - Access to MySQL database (for CDC)
  - Internet access for Docker images
  - Open ports: 3000, 8080, 8081, 8083, 9021, 9090-9096

---

## Quick Start

### Step 1: Create Docker Network

All services communicate through a shared Docker network:

```bash
docker network create monitoring
```

### Step 2: Start the Kafka Cluster

```bash
cd cluster
docker-compose up -d
```

**Verify cluster health:**

```bash
# Check all containers are running
docker ps | grep -E "controller|broker|kafka-ui"

# Access Kafka UI
open http://localhost:8080
```

### Step 3: Start Kafka Connect & Debezium

```bash
cd ../connect
docker-compose up -d
```

**Verify Debezium is ready:**

```bash
# Wait for health check
docker logs debezium -f

# List all available connectors
curl http://localhost:8083/connectors
```

### Step 4: Configure the CDC Connector

Edit `connect/connector.json` with your MySQL database details:

```json
{
  "name": "your-connector-name",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "your-mysql-host",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "your-password",
    "database.server.name": "your-server-name",
    "table.include.list": "your_database.*"
  }
}
```

**Deploy the connector:**

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @connector.json
```

### Step 5: Start Monitoring Stack

```bash
cd ../monitor
docker-compose up -d
```

**Access monitoring UIs:**

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin)
- Alertmanager: http://localhost:9094

### Step 6: Configure Alerts (Optional)

Edit `monitor/calert/config.toml` to set your Google Chat webhook:

```toml
[providers.ggchat_alerts]
endpoint = "YOUR_GOOGLE_CHAT_WEBHOOK_URL"
```

Restart the monitoring stack:

```bash
cd monitor
docker-compose restart calert
```

---

## System Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                CDC PLATFORM                                │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│  SOURCE DATABASE                                                           │
│  ┌──────────────┐                                                          │
│  │   MySQL RDS  │  (Binary Log enabled)                                    │
│  │   Database   │                                                          │
│  └──────┬───────┘                                                          │
│         │ Binlog Stream                                                    │
└─────────┼──────────────────────────────────────────────────────────────────┘
          │
          ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  KAFKA CONNECT LAYER                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  Debezium MySQL Connector (Port 8083)                                 │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐                │  │
│  │  │   Snapshot  │→ │  CDC Stream  │→ │ Custom SMT     │                │  │
│  │  │   Module    │  │  (Binlog)    │  │ (Clean nulls)  │                │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘                │  │
│  │         ↓                                    ↓                        │  │
│  │  ┌────────────────────────────────────────────────────┐               │  │
│  │  │        Schema Registry (Port 8081)                 │               │  │
│  │  │        Avro Schema Management                      │               │  │
│  │  └────────────────────────────────────────────────────┘               │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │ CDC Events (Avro)
                                   ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  KAFKA CLUSTER (KRaft Mode)                                                 │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  CONTROLLERS (Metadata Management)                              │        │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │        │
│  │  │ Controller-1 │  │ Controller-2 │  │ Controller-3 │           │        │
│  │  │   Node ID: 1 │  │   Node ID: 2 │  │   Node ID: 3 │           │        │
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │        │
│  │         Quorum Consensus (Raft Protocol)                        │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  BROKERS (Data Storage & Client Handling)                       │        │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │        │
│  │  │   Broker-1   │  │   Broker-2   │  │   Broker-3   │           │        │
│  │  │   Port 9093  │  │   Port 9095  │  │   Port 9096  │           │        │
│  │  │  Node ID: 4  │  │  Node ID: 5  │  │  Node ID: 6  │           │        │
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  Topics: database_tablename (Avro serialized)                               │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ↓ Metrics (JMX → Prometheus)
┌─────────────────────────────────────────────────────────────────────────────┐
│  MONITORING & OBSERVABILITY STACK                                           │
│                                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                 │
│  │  JMX         │────→│  Prometheus  │────→│   Grafana    │                 │
│  │  Exporters   │     │  (Port 9090) │     │  (Port 3000) │                 │
│  │  (5000-5006) │     │              │     │  Dashboards  │                 │
│  └──────────────┘     └──────┬───────┘     └──────────────┘                 │
│                               │                                             │
│                               ↓ Alert Rules                                 │
│                      ┌────────────────┐                                     │
│                      │  Alertmanager  │                                     │
│                      │  (Port 9094)   │                                     │
│                      └────────┬───────┘                                     │
│                               │                                             │
│                               ↓ Webhook                                     │
│                      ┌────────────────┐                                     │
│                      │     Calert     │                                     │
│                      │  (Port 6000)   │                                     │
│                      └────────┬───────┘                                     │
│                               │                                             │
│                               ↓                                             │
│                      ┌────────────────┐                                     │
│                      │  Google Chat   │                                     │
│                      │  Notifications │                                     │
│                      └────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│  MANAGEMENT UIs                                                             │
│  • Kafka UI (8080)          - Topic/Consumer management                     │
│  • Control Center (9021)    - Connector management                          │
│  • Grafana (3000)           - Metrics visualization                         │
│  • Prometheus (9090)        - Query metrics                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Explanation

1. **Source Database** → Binary logs (binlog) capture all database changes
2. **Debezium** → Reads binlog and converts to CDC events
3. **Custom SMT** → Cleans data (removes null characters)
4. **Schema Registry** → Validates and stores Avro schemas
5. **Kafka Brokers** → Store CDC events in topics (format: `database_tablename`)
6. **JMX Exporters** → Export metrics from Kafka, Controllers, Debezium
7. **Prometheus** → Scrapes and stores metrics
8. **Alertmanager** → Evaluates alert rules and triggers notifications
9. **Calert** → Formats and sends alerts to Google Chat
10. **Grafana** → Visualizes metrics in dashboards

---

## Project Structure

```
barall-cdc/
├── cluster/                    # Kafka cluster (KRaft mode)
│   ├── docker-compose.yml      # 3 controllers + 3 brokers + Kafka UI
│   ├── exporter/               # JMX exporter configurations
│   │   ├── broker.yml          # Broker metrics mapping
│   │   └── controller.yml      # Controller metrics mapping
│   └── jmx_prometheus_javaagent-1.5.0.jar
│
├── connect/                    # Kafka Connect & Debezium
│   ├── docker-compose.yml      # Debezium, Schema Registry, Control Center
│   ├── connector.json          # MySQL CDC connector configuration
│   ├── exporter/
│   │   └── debezium.yml        # Debezium metrics mapping
│   ├── jmx_prometheus_javaagent-1.5.0.jar
│   └── smt/                    # Custom Single Message Transform
│       ├── pom.xml             # Maven project definition
│       ├── src/main/java/      # Java source code
│       └── target/             # Compiled JAR
│
└── monitor/                    # Monitoring stack
    ├── docker-compose.yml      # Prometheus, Grafana, Alertmanager, Calert
    ├── prometheus/
    │   ├── prometheus.yml      # Scrape configs and targets
    │   ├── web.yml             # Basic auth configuration
    │   └── rules/              # Alert rules
    │       ├── debezium.yml    # CDC-specific alerts
    │       └── node_exporter.yml  # System alerts
    ├── alertmanager/
    │   └── alertmanager.yml    # Alert routing and grouping
    └── calert/
        ├── config.toml         # Google Chat webhook config
        └── templates/
            └── message.tmpl    # Alert message template
```

---

## Configuration Details

### Kafka Cluster Configuration

**KRaft Mode (No Zookeeper):**

- **Cluster ID**: `MkU3OEVBNTcwNTJENDM2Qk` (must be identical across all nodes)
- **Controller Quorum**: 3 nodes for high availability
- **Replication Factor**: 1 (for development; increase to 3 for production)

**Key Environment Variables:**

```bash
KAFKA_ENABLE_KRAFT: "true"              # Enable KRaft mode
KAFKA_PROCESS_ROLES: "controller"/"broker"  # Node role
KAFKA_CONTROLLER_QUORUM_VOTERS: "1@controller-1:29092,2@controller-2:29092,3@controller-3:29092"
KAFKA_MESSAGE_MAX_BYTES: 20971520       # 20MB max message size
```

### Debezium Configuration

**Connector Settings:**

```json
"snapshot.mode": "schema_only"          // Don't snapshot existing data
"snapshot.locking.mode": "none"         // No table locks during snapshot
"delete.handling.mode": "rewrite"       // Rewrite tombstones as delete events
"tombstones.on.delete": "true"          // Emit tombstone after delete
```

**Schema Registry:**

- Stores Avro schemas for all topics
- Enables schema evolution
- URL: `http://schema-registry:8081`

### Monitoring Configuration

**Prometheus Scrape Intervals:**

- Default: 10s
- Debezium: 5s (more frequent for CDC metrics)

**Alert Rules:**

- **Critical**: Instance down, connector disconnected
- **Warning**: High lag, no events processing, high resource usage

**Alertmanager:**

- **Group Wait**: 20s (collect alerts before sending)
- **Group Interval**: 3m (wait before sending updates)
- **Repeat Interval**: 1h (reminder frequency)

---

## Monitoring and Alerting

### Available Metrics

**Kafka Metrics:**

- Broker: Request rate, byte in/out, topic metrics
- Controller: Active controllers, leader elections
- Topics: Message rate, partition count

**Debezium Metrics:**

- Connection status (`debezium_metrics_Connected`)
- Changes applied (`debezium_metrics_ChangesApplied`)
- Lag (`debezium_metrics_MilliSecondsSinceLastAppliedChange`)
- Event processing rate

**System Metrics:**

- CPU, RAM, Disk usage
- Network throughput

### Alert Types

| Alert                         | Severity | Condition                   | Action                   |
| ----------------------------- | -------- | --------------------------- | ------------------------ |
| DebeziumInstanceDown          | Critical | Instance unreachable for 1m | Check Docker container   |
| DebeziumConnectorNotConnected | Critical | No DB connection for 30s    | Verify MySQL credentials |
| DebeziumConnectorNoChanges    | Critical | No changes for 5m           | Check connector status   |
| HighCPUUsage                  | Warning  | >80% for 2m                 | Investigate workload     |
| HighRAMUsage                  | Warning  | >90% for 2m                 | Check for memory leaks   |

### Accessing Dashboards

**Grafana Setup:**

1. Access http://localhost:3000
2. Login: admin/admin
3. Add Prometheus data source: http://prometheus:9090
4. Import dashboards:
   - Kafka Cluster Metrics
   - Debezium CDC Monitoring
   - System Resources

---

## Troubleshooting

### Common Issues

#### 1. Kafka Brokers Not Starting

**Symptom:** Brokers crash or fail to connect to controllers

**Solution:**

```bash
# Check logs
docker logs broker-1

# Ensure all nodes have same CLUSTER_ID
# Verify controller quorum is healthy
docker logs controller-1
```

#### 2. Debezium Connector Fails

**Symptom:** Connector status shows FAILED

**Solution:**

```bash
# Check connector status
curl http://localhost:8083/connectors/your-connector/status

# View detailed logs
docker logs debezium

# Common fixes:
# - Verify MySQL binlog is enabled: SET GLOBAL binlog_format = 'ROW';
# - Check database user permissions
# - Verify network connectivity to MySQL
```

#### 3. No Metrics in Prometheus

**Symptom:** Targets show as DOWN in Prometheus

**Solution:**

```bash
# Check Prometheus targets
open http://localhost:9090/targets

# Verify JMX exporters are running
curl http://localhost:5004  # Broker-1 exporter
curl http://localhost:5000  # Debezium exporter

# Check Prometheus config
docker logs prometheus
```

#### 4. Alerts Not Firing

**Symptom:** No Google Chat notifications

**Solution:**

```bash
# Check Alertmanager
open http://localhost:9094

# Verify Calert is receiving webhooks
docker logs calert

# Test Google Chat webhook manually
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"text": "Test message"}'
```

#### 5. Schema Registry Connection Failed

**Symptom:** Connector cannot serialize messages

**Solution:**

```bash
# Check Schema Registry health
curl http://localhost:8081/subjects

# Verify Schema Registry is connected to Kafka
docker logs schema-registry

# Re-register schemas if needed
```

---

## Best Practices

### 1. Production Deployment

- **Increase replication factors:**

  ```bash
  KAFKA_DEFAULT_REPLICATION_FACTOR: 3
  KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
  KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
  ```
- **Use persistent volumes:**

  ```yaml
  volumes:
    - /mnt/kafka-data/broker1:/var/lib/kafka/data
  ```
- **Enable SSL/TLS:**

  - Configure Kafka with SSL listeners
  - Use authenticated Schema Registry

### 2. Database Configuration

**MySQL Setup for CDC:**

```sql
-- Enable binlog
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- Create Debezium user
CREATE USER 'debezium'@'%' IDENTIFIED BY 'password';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT 
  ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

### 3. Monitoring

- **Set up Grafana dashboards** for all components
- **Configure alert thresholds** based on your workload
- **Use log aggregation** (ELK/Loki) for centralized logging
- **Regular backups** of Kafka data and Prometheus metrics

### 4. Performance Tuning

**Kafka:**

- Increase `num.network.threads` for high throughput
- Tune `replica.fetch.max.bytes` for large messages
- Use compression: `compression.type=lz4`

**Debezium:**

- Adjust `max.queue.size` for buffering
- Use `snapshot.mode=schema_only` for large databases
- Enable `provide.transaction.metadata` for transactional consistency

### 5. Maintenance

**Regular Tasks:**

- Monitor disk usage on Kafka brokers
- Clean up old topics: `kafka-topics --delete --topic old-topic`
- Update Docker images: `docker-compose pull && docker-compose up -d`
- Review and tune alert rules based on false positives

**Backup Strategy:**

- Export Kafka topics with MirrorMaker or Kafka Connect
- Backup Schema Registry schemas
- Snapshot Prometheus data

---

## Additional Resources

### Documentation Links

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Debezium Documentation](https://debezium.io/documentation/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

### Useful Commands

**Kafka Topics:**

```bash
# List topics
docker exec broker-1 kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
docker exec broker-1 kafka-topics --describe --topic your-topic --bootstrap-server localhost:9092

# Consume messages
docker exec broker-1 kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic your-topic --from-beginning
```

**Debezium Connector Management:**

```bash
# List connectors
curl http://localhost:8083/connectors

# Get connector config
curl http://localhost:8083/connectors/your-connector/config

# Restart connector
curl -X POST http://localhost:8083/connectors/your-connector/restart

# Delete connector
curl -X DELETE http://localhost:8083/connectors/your-connector
```

