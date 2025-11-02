# Kafka Cluster (KRaft Mode)

## Overview

This folder contains the configuration for a **production-ready Apache Kafka cluster** running in **KRaft mode** (Kafka Raft metadata mode), which eliminates the need for Apache Zookeeper.

The cluster consists of:

- **3 Controllers**: Manage cluster metadata and leader election
- **3 Brokers**: Handle data storage and client requests
- **1 Kafka UI**: Web-based management interface

All components are configured with **JMX exporters** for Prometheus monitoring.

---

## Table of Contents

- [Architecture](#architecture)
- [KRaft Mode Explained](#kraft-mode-explained)
- [Components](#components)
- [Configuration Details](#configuration-details)
- [Ports and Network](#ports-and-network)
- [How to Use](#how-to-use)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                 KAFKA CLUSTER (KRaft Mode)                 │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  CONTROL PLANE (Metadata Management)                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐   │   │
│  │  │ Controller-1 │  │ Controller-2 │  │Ctrlr-3   │   │   │
│  │  │   Node ID:1  │  │   Node ID:2  │  │Node ID:3 │   │   │
│  │  │  Port:29092  │  │  Port:29092  │  │Port:29092│   │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘   │   │
│  │         ↑                  ↑                 ↑      │   │
│  │         └──────────────────┴─────────────────┘      │   │
│  │              Raft Consensus Protocol                │   │
│  │         (Leader Election & Metadata Replication)    │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                 │
│                          ↓ Metadata                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  DATA PLANE (Client Requests & Data Storage)        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐   │   │
│  │  │   Broker-1   │  │   Broker-2   │  │ Broker-3 │   │   │
│  │  │   Node ID:4  │  │   Node ID:5  │  │Node ID:6 │   │   │
│  │  │  Ext:9093    │  │  Ext:9095    │  │Ext:9096  │   │   │
│  │  │  Int:9092    │  │  Int:9092    │  │Int:9092  │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MANAGEMENT UI                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │           Kafka UI (Port 8080)               │   │   │
│  │  │  • Topic Management                          │   │   │
│  │  │  • Consumer Groups                           │   │   │
│  │  │  • Broker Monitoring                         │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  JMX EXPORTERS (Prometheus Metrics)                 │   │
│  │  Controllers: 5001, 5002, 5003                      │   │
│  │  Brokers: 5004, 5005, 5006                          │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

---

## KRaft Mode Explained

### What is KRaft?

**KRaft** (Kafka Raft) is Kafka's new consensus protocol that replaces Apache Zookeeper for metadata management. Introduced in Kafka 2.8 and production-ready in Kafka 3.3+.

### How KRaft Works

1. **Controller Quorum**: A group of controller nodes form a Raft quorum
2. **Leader Election**: Controllers elect a leader using the Raft protocol
3. **Metadata Replication**: The leader replicates metadata logs to followers
4. **Broker Communication**: Brokers read metadata from the controller leader

### Benefits of KRaft vs Zookeeper

| Feature                   | KRaft                     | Zookeeper                |
| ------------------------- | ------------------------- | ------------------------ |
| **Architecture**    | Integrated into Kafka     | Separate service         |
| **Scalability**     | Supports 100K+ partitions | Limited to ~200K         |
| **Deployment**      | Simpler (one system)      | Complex (two systems)    |
| **Failover Time**   | Faster (seconds)          | Slower (tens of seconds) |
| **Metadata Access** | Read from any broker      | Must query Zookeeper     |
| **Security**        | Same as Kafka             | Separate security config |

### Key Concepts

- **Cluster ID**: A unique identifier shared by all nodes. Must be identical across controllers and brokers.
- **Node ID**: Unique identifier for each node in the cluster (1-6 in this setup).
- **Process Roles**: Each node is either a `controller`, `broker`, or `controller,broker` (combined mode).
- **Controller Quorum Voters**: List of controller nodes participating in metadata management.

---

## Components

### 1. Controllers (controller-1, controller-2, controller-3)

**Purpose**: Manage cluster metadata, topic configurations, partition leadership, and handle administrative requests.

**Key Configuration**:

```yaml
KAFKA_PROCESS_ROLES: "controller"              # This node is ONLY a controller
KAFKA_NODE_ID: 1                               # Unique ID (1, 2, or 3)
KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"  # Listener name for controller traffic
KAFKA_LISTENERS: "CONTROLLER://0.0.0.0:29092"  # Controller listens on port 29092
KAFKA_CONTROLLER_QUORUM_VOTERS: "1@controller-1:29092,2@controller-2:29092,3@controller-3:29092"
CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"           # Must be identical across all nodes
```

**Explanation**:

- **Why 3 controllers?** Provides fault tolerance.
- **CONTROLLER listener**: Internal protocol for Raft consensus and metadata replication.
- **29092 port**: Used for inter-controller communication (not accessible externally).

### 2. Brokers (broker-1, broker-2, broker-3)

**Purpose**: Store topic data, serve produce/consume requests, and replicate partitions.

**Key Configuration**:

```yaml
KAFKA_PROCESS_ROLES: "broker"                  # This node is ONLY a broker
KAFKA_NODE_ID: 4                               # Unique ID (4, 5, or 6)
KAFKA_CONTROLLER_QUORUM_VOTERS: "1@controller-1:29092,2@controller-2:29092,3@controller-3:29092"
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT"
KAFKA_INTER_BROKER_LISTENER_NAME: "INTERNAL"   # Brokers communicate via INTERNAL listener
KAFKA_LISTENERS: "INTERNAL://0.0.0.0:9092, EXTERNAL://0.0.0.0:9093"
KAFKA_ADVERTISED_LISTENERS: "INTERNAL://broker-1:9092, EXTERNAL://localhost:9093"
```

**Explanation**:

- **INTERNAL listener**: Used for inter-broker communication (within Docker network).
- **EXTERNAL listener**: Used for client connections from the host machine.
- **Advertised listeners**: Tells clients how to connect. Clients inside Docker use `broker-1:9092`, external clients use `localhost:9093`.

**Storage Configuration**:

```yaml
KAFKA_LOG_DIRS: "/var/lib/kafka/data"               # Data directory
KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"             # Auto-create topics when produced to
KAFKA_DEFAULT_REPLICATION_FACTOR: 1                 # Default replication (increase to 3 for prod)
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1           # Consumer offset replication
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1   # Transaction log replication
KAFKA_LOG_RETENTION_HOURS: 168                      # Keep data for 7 days
KAFKA_MESSAGE_MAX_BYTES: 20971520                   # 20MB max message size
```

**Why these settings?**

- **Replication Factor 1**: For development/testing. In production, use 3 for high availability.
- **20MB messages**: Allows large CDC events (e.g., database rows with BLOB fields).
- **7 days retention**: Balances storage cost with data availability.

### 3. Kafka UI

**Purpose**: Web-based UI for managing and monitoring Kafka.

**Features**:

- Browse topics and messages
- View consumer groups and lag
- Monitor broker health
- Manage schemas (if Schema Registry is connected)
- Create/delete topics

**Configuration**:

```yaml
KAFKA_CLUSTERS_0_NAME: "local"
KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: "broker-1:9092,broker-2:9092,broker-3:9092"
DYNAMIC_CONFIG_ENABLED: 'true'
```

---

## Configuration Details

### JMX and Prometheus Exporters

Each controller and broker is configured with:

1. **JMX (Java Management Extensions)**: Exposes internal Kafka metrics via RMI
2. **JMX Exporter**: Converts JMX metrics to Prometheus format

**JMX Configuration**:

```yaml
JMX_PORT: 9101
KAFKA_JMX_OPTS: >
  -Dcom.sun.management.jmxremote
  -Dcom.sun.management.jmxremote.rmi.port=9101
  -Dcom.sun.management.jmxremote.authenticate=false
  -Dcom.sun.management.jmxremote.ssl=false
  -Djava.rmi.server.hostname=broker-1
```

**JMX Exporter Configuration**:

```yaml
KAFKA_OPTS: >
  -javaagent:/usr/app/jmx_prometheus_javaagent.jar=5556:/etc/jmx_exporter/config.yml
```

**What this does**:

- Starts JMX on port 9101 (for direct JMX access)
- Starts JMX Exporter HTTP server on port 5556 (for Prometheus scraping)
- Loads exporter rules from `/etc/jmx_exporter/config.yml`

### Exporter Configuration Files

#### `exporter/broker.yml`

Maps JMX MBean attributes to Prometheus metrics:

**Example Rules**:

```yaml
# Convert JMX metric: kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec
# To Prometheus: kafka_server_BrokerTopicMetrics_MessagesInPerSec_total

- pattern: kafka.(\w+)<type=(.+), name=(.+)PerSec\w*><>Count
  name: kafka_$1_$2_$3_total
  type: COUNTER
```

**Key Metrics Exported**:

- `kafka_server_BrokerTopicMetrics_BytesInPerSec`: Data ingestion rate
- `kafka_server_BrokerTopicMetrics_BytesOutPerSec`: Data consumption rate
- `kafka_server_BrokerTopicMetrics_MessagesInPerSec`: Message rate
- `kafka_controller_KafkaController_ActiveControllerCount`: Controller status
- `kafka_server_ReplicaManager_LeaderCount`: Partition leadership

#### `exporter/controller.yml`

Same format as broker.yml, but typically exports fewer metrics since controllers don't handle data.

---

## Ports and Network

### Port Mapping

| Service                       | Internal Port | External Port | Purpose            |
| ----------------------------- | ------------- | ------------- | ------------------ |
| **controller-1**        | 29092         | -             | Controller quorum  |
| controller-1 JMX              | 9101          | 9101          | JMX monitoring     |
| controller-1 Exporter         | 5556          | 5001          | Prometheus metrics |
| **controller-2**        | 29092         | -             | Controller quorum  |
| controller-2 JMX              | 9101          | 9102          | JMX monitoring     |
| controller-2 Exporter         | 5556          | 5002          | Prometheus metrics |
| **controller-3**        | 29092         | -             | Controller quorum  |
| controller-3 JMX              | 9101          | 9103          | JMX monitoring     |
| controller-3 Exporter         | 5556          | 5003          | Prometheus metrics |
| **broker-1 (INTERNAL)** | 9092          | -             | Inter-broker       |
| **broker-1 (EXTERNAL)** | 9093          | 9093          | Client connections |
| broker-1 JMX                  | 9101          | 9104          | JMX monitoring     |
| broker-1 Exporter             | 5556          | 5004          | Prometheus metrics |
| **broker-2 (EXTERNAL)** | 9093          | 9095          | Client connections |
| broker-2 JMX                  | 9101          | 9105          | JMX monitoring     |
| broker-2 Exporter             | 5556          | 5005          | Prometheus metrics |
| **broker-3 (EXTERNAL)** | 9093          | 9096          | Client connections |
| broker-3 JMX                  | 9101          | 9106          | JMX monitoring     |
| broker-3 Exporter             | 5556          | 5006          | Prometheus metrics |
| **kafka-ui**            | 8080          | 8080          | Web UI             |

### Docker Network

All services are connected to the `monitoring` Docker network:

```yaml
networks:
  monitoring:
    external: true
```

**Important**: The `monitoring` network must be created before starting services:

```bash
docker network create monitoring
```

---

## How to Use

### Step 1: Create Docker Network

```bash
docker network create monitoring
```

### Step 2: Start the Cluster

```bash
cd cluster
docker-compose up -d
```

**What happens**:

1. All 3 controllers start and form a Raft quorum
2. One controller is elected as leader
3. All 3 brokers start and register with the controller leader
4. Kafka UI connects to the brokers

### Step 3: Verify Cluster Health

**Check containers are running**:

```bash
docker ps | grep -E "controller|broker|kafka-ui"
```

**Expected output**:

```
controller-1    Up    5001->5556, 9101->9101
controller-2    Up    5002->5556, 9102->9101
controller-3    Up    5003->5556, 9103->9101
broker-1        Up    5004->5556, 9093->9093, 9104->9101
broker-2        Up    5005->5556, 9095->9093, 9105->9101
broker-3        Up    5006->5556, 9096->9093, 9106->9101
kafka-ui        Up    8080->8080
```

**Check controller leader**:

```bash
docker logs controller-1 | grep -i "leader"
```

**Access Kafka UI**:

```bash
open http://localhost:8080
```

### Step 4: Test the Cluster

**Create a test topic**:

```bash
docker exec broker-1 kafka-topics \
  --create \
  --topic test-topic \
  --partitions 3 \
  --replication-factor 1 \
  --bootstrap-server localhost:9092
```

**Produce a message**:

```bash
docker exec -it broker-1 kafka-console-producer \
  --topic test-topic \
  --bootstrap-server localhost:9092

# Type a message and press Enter
> Hello KRaft!
```

**Consume the message**:

```bash
docker exec broker-1 kafka-console-consumer \
  --topic test-topic \
  --from-beginning \
  --bootstrap-server localhost:9092
```

### Step 5: Stop the Cluster

```bash
docker-compose down
```

**Preserve data** (keeps volumes):

```bash
docker-compose down
```

**Delete all data**:

```bash
docker-compose down -v
```

---

## Monitoring

### Accessing Metrics

**JMX Exporter endpoints** (Prometheus format):

```bash
# Controller metrics
curl http://localhost:5001/metrics
curl http://localhost:5002/metrics
curl http://localhost:5003/metrics

# Broker metrics
curl http://localhost:5004/metrics
curl http://localhost:5005/metrics
curl http://localhost:5006/metrics
```

**JMX direct access** (requires JMX client like JConsole):

```bash
# Connect to broker-1 JMX
jconsole localhost:9104
```

### Key Metrics to Monitor

**Cluster Health**:

- `kafka_controller_KafkaController_ActiveControllerCount`: Should be 1 (only one active controller)
- `kafka_server_ReplicaManager_PartitionCount`: Number of partitions per broker
- `kafka_server_ReplicaManager_LeaderCount`: Number of leader partitions

**Throughput**:

- `kafka_server_BrokerTopicMetrics_BytesInPerSec`: Incoming data rate
- `kafka_server_BrokerTopicMetrics_BytesOutPerSec`: Outgoing data rate
- `kafka_server_BrokerTopicMetrics_MessagesInPerSec`: Message rate

**Performance**:

- `kafka_network_RequestMetrics_RequestsPerSec`: Request rate by type
- `kafka_network_RequestMetrics_TotalTimeMs`: Request latency

**Storage**:

- `kafka_log_LogManager_LogDirectoryOffline`: Offline log directories (should be 0)

---

## Troubleshooting

### Problem 1: Controllers Not Forming Quorum

**Symptoms**:

- Controllers keep restarting
- Logs show "Unable to reach quorum"

**Solution**:

```bash
# Check logs for all controllers
docker logs controller-1
docker logs controller-2
docker logs controller-3

# Verify CLUSTER_ID is identical across all nodes
docker exec controller-1 cat /var/lib/kafka/data/meta.properties
docker exec controller-2 cat /var/lib/kafka/data/meta.properties
docker exec controller-3 cat /var/lib/kafka/data/meta.properties

# If CLUSTER_ID mismatch, stop all services and delete volumes
docker-compose down -v
docker-compose up -d
```

### Problem 2: Brokers Cannot Connect to Controllers

**Symptoms**:

- Brokers log "Connection refused" to controllers
- Brokers keep restarting

**Solution**:

```bash
# Verify controllers are running
docker ps | grep controller

# Check network connectivity
docker exec broker-1 ping controller-1

# Verify KAFKA_CONTROLLER_QUORUM_VOTERS is correct
docker exec broker-1 env | grep QUORUM
```

### Problem 3: Cannot Connect from Host Machine

**Symptoms**:

- Applications on host cannot connect to `localhost:9093`

**Solution**:

```bash
# Verify EXTERNAL listener is bound
docker exec broker-1 netstat -tuln | grep 9093

# Test connection
telnet localhost 9093

# Check firewall rules (Linux)
sudo ufw status

# Verify KAFKA_ADVERTISED_LISTENERS
docker exec broker-1 env | grep ADVERTISED
```

### Problem 4: High Memory Usage

**Symptoms**:

- Containers using >2GB RAM each
- System becomes slow

**Solution**:

```yaml
# Add memory limits in docker-compose.yml
services:
  broker-1:
    deploy:
      resources:
        limits:
          memory: 2G

# Adjust Kafka heap size
environment:
  KAFKA_HEAP_OPTS: "-Xms1G -Xmx1G"
```

### Problem 5: Metrics Not Appearing in Prometheus

**Symptoms**:

- Prometheus targets show as DOWN
- No metrics in Grafana

**Solution**:

```bash
# Test JMX exporter endpoint
curl http://localhost:5004/metrics

# Check if JMX agent is loaded
docker logs broker-1 | grep javaagent

# Verify exporter config is mounted
docker exec broker-1 cat /etc/jmx_exporter/config.yml

# Restart service if config was changed
docker-compose restart broker-1
```

---

## Production Considerations

### 1. Increase Replication Factors

For high availability, use replication factor of 3:

```yaml
KAFKA_DEFAULT_REPLICATION_FACTOR: 3
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
```

### 2. Use Persistent Volumes

Map volumes to host directories for data persistence:

```yaml
volumes:
  - /mnt/kafka-data/broker1:/var/lib/kafka/data
```

### 3. Enable SSL/TLS

Add SSL listeners for secure communication:

```yaml
KAFKA_LISTENERS: "INTERNAL://0.0.0.0:9092, EXTERNAL://0.0.0.0:9093, SSL://0.0.0.0:9094"
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,SSL:SSL"
KAFKA_SSL_KEYSTORE_FILENAME: "kafka.keystore.jks"
KAFKA_SSL_KEYSTORE_CREDENTIALS: "keystore_creds"
KAFKA_SSL_KEY_CREDENTIALS: "key_creds"
```

### 4. Enable SASL Authentication

Add authentication for security:

```yaml
KAFKA_SASL_ENABLED_MECHANISMS: "PLAIN"
KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: "PLAIN"
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "INTERNAL:SASL_PLAINTEXT,EXTERNAL:SASL_PLAINTEXT"
```

### 5. Configure Log Retention

Adjust retention based on storage capacity:

```yaml
KAFKA_LOG_RETENTION_HOURS: 168              # 7 days
KAFKA_LOG_RETENTION_BYTES: 1073741824       # 1GB per partition
KAFKA_LOG_SEGMENT_BYTES: 536870912          # 512MB segment size
```

---

## Useful Commands

### Topic Management

```bash
# List all topics
docker exec broker-1 kafka-topics --list --bootstrap-server localhost:9092

# Describe topic
docker exec broker-1 kafka-topics --describe --topic my-topic --bootstrap-server localhost:9092

# Delete topic
docker exec broker-1 kafka-topics --delete --topic my-topic --bootstrap-server localhost:9092

# Alter topic (change partitions)
docker exec broker-1 kafka-topics --alter --topic my-topic --partitions 6 --bootstrap-server localhost:9092
```

### Consumer Groups

```bash
# List consumer groups
docker exec broker-1 kafka-consumer-groups --list --bootstrap-server localhost:9092

# Describe consumer group
docker exec broker-1 kafka-consumer-groups --describe --group my-group --bootstrap-server localhost:9092

# Reset consumer group offset
docker exec broker-1 kafka-consumer-groups --reset-offsets --group my-group --topic my-topic --to-earliest --execute --bootstrap-server localhost:9092
```

### Cluster Information

```bash
# List brokers
docker exec broker-1 kafka-broker-api-versions --bootstrap-server localhost:9092

# Get cluster ID
docker exec broker-1 kafka-cluster --describe --bootstrap-server localhost:9092
```

---

## Architecture Diagram: Message Flow

```
┌────────────────────────────────────────────────────────────┐
│  PRODUCER                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Application sends message to topic "orders"         │  │
│  │  Bootstrap servers: broker-1:9093,broker-2:9095,...  │  │
│  └───────────────────┬──────────────────────────────────┘  │
└────────────────────────┼───────────────────────────────────┘
                         │
                         ↓ (Produce Request)
┌─────────────────────────────────────────────────────────────┐
│  BROKER-1 (Leader for partition 0)                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  1. Receives message                                 │   │
│  │  2. Writes to log segment                            │   │
│  │  3. Replicates to followers (if replication > 1)     │   │
│  │  4. Returns acknowledgment to producer               │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                         │
                         ↓ (Message stored)
┌─────────────────────────────────────────────────────────────┐
│  TOPIC: orders (3 partitions, replication factor 1)         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Part 0  │  │  Part 1  │  │  Part 2  │                   │
│  │ Broker-1 │  │ Broker-2 │  │ Broker-3 │                   │
│  │ [Msg1]   │  │ [Msg2]   │  │ [Msg3]   │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
└─────────────────────────────────────────────────────────────┘
                         │
                         ↓ (Fetch Request)
┌─────────────────────────────────────────────────────────────┐
│  CONSUMER                                                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Application polls messages from "orders"            │   │
│  │  Consumer group: "order-processor"                   │   │
│  │  Offset tracking: __consumer_offsets topic           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Additional Resources

- [Kafka KRaft Documentation](https://kafka.apache.org/documentation/#kraft)
- [Kafka Configuration Reference](https://kafka.apache.org/documentation/#configuration)
- [Kafka UI Documentation](https://docs.kafka-ui.provectus.io/)
- [JMX Exporter Rules](https://github.com/prometheus/jmx_exporter)

---
