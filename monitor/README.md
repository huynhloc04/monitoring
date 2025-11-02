# Monitoring & Alerting Stack

## Overview

This folder contains a **complete monitoring and alerting stack** for the Kafka CDC platform. It collects metrics, stores them, visualizes them, and sends alerts when issues occur.

The stack includes:

- **Prometheus**: Metrics collection and storage (TSDB)
- **Grafana**: Dashboards and visualization
- **Alertmanager**: Alert routing and grouping
- **Calert**: Google Chat webhook dispatcher
- **Node Exporter**: System metrics collection

---

## Table of Contents

- [Architecture](#architecture)
- [How It Works](#how-it-works)
- [Components](#components)
- [Configuration Details](#configuration-details)
- [Alert Rules](#alert-rules)
- [How to Use](#how-to-use)
- [Creating Dashboards](#creating-dashboards)
- [Troubleshooting](#troubleshooting)
- [Production Considerations](#production-considerations)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  METRIC SOURCES                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │ Kafka      │  │ Debezium   │  │ Node       │            │
│  │ Brokers    │  │ Connect    │  │ Exporter   │            │
│  │ (5004-6)   │  │ (5000)     │  │ (9100)     │            │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘            │
│         │ JMX            │ JMX            │ /proc, /sys     │
│         │ Exporter       │ Exporter       │                 │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ↓ HTTP :5004-6   ↓ HTTP :5000    ↓ HTTP :9100
┌─────────────────────────────────────────────────────────────┐
│  PROMETHEUS (Port 9090)                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Scrape Configuration                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │ │
│  │  │ kafka-brokers│  │  debezium    │  │node-exporter │ │ │
│  │  │ scrape:10s   │  │  scrape:5s   │  │ scrape:10s   │ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Time Series Database (TSDB)                          │ │
│  │  - Stores metrics with timestamps                     │ │
│  │  - Retention: 15 days (default)                       │ │
│  │  - Compression: Efficient time series storage         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Rule Evaluation (every 10s)                          │ │
│  │  ┌──────────────────┐  ┌──────────────────┐          │ │
│  │  │ debezium.yml     │  │node_exporter.yml │          │ │
│  │  │ - Connection     │  │ - CPU/RAM/Disk   │          │ │
│  │  │ - Lag            │  │ - Network        │          │ │
│  │  │ - Errors         │  │ - Uptime         │          │ │
│  │  └──────────────────┘  └──────────────────┘          │ │
│  └───────────────────┬────────────────────────────────────┘ │
└────────────────────────┼─────────────────────────────────────┘
                         │ Alerts (when rules trigger)
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  ALERTMANAGER (Port 9094)                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Alert Processing Pipeline                             │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │ │
│  │  │ Receive │→ │ Group   │→ │Inhibit  │→ │ Route   │  │ │
│  │  │ Alerts  │  │ (20s)   │  │ Rules   │  │         │  │ │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  │ │
│  │                                                        │ │
│  │  Grouping Example:                                     │ │
│  │  - Wait 20s to collect related alerts                 │ │
│  │  - Group by: alertname, component, instance           │ │
│  │  - Send as single notification                        │ │
│  │                                                        │ │
│  │  Inhibit Example:                                      │ │
│  │  - Critical alert suppresses warning                  │ │
│  │  - Avoids duplicate notifications                     │ │
│  └───────────────────┬────────────────────────────────────┘ │
└────────────────────────┼─────────────────────────────────────┘
                         │ Webhook POST
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  CALERT (Port 6000)                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Webhook Handler                                       │ │
│  │  1. Receive Alertmanager webhook                      │ │
│  │  2. Parse alert JSON                                  │ │
│  │  3. Apply message template                            │ │
│  │  4. Format for Google Chat                            │ │
│  │  5. Send to Google Chat webhook                       │ │
│  └───────────────────┬────────────────────────────────────┘ │
└────────────────────────┼─────────────────────────────────────┘
                         │ HTTPS POST
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  GOOGLE CHAT                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  🔥 FIRING ALERT                                       │ │
│  │  (CRITICAL) DebeziumInstanceDown - Firing              │ │
│  │                                                        │ │
│  │  Title: Debezium Instance is DOWN                     │ │
│  │  Description: Instance unreachable for 1+ minute      │ │
│  │  Instance: debezium:8083                              │ │
│  │  Started: 2024-11-01 10:30:00                         │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  GRAFANA (Port 3000)                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Visualization Layer                                   │ │
│  │  - Data Source: Prometheus                            │ │
│  │  - Dashboards: Kafka, Debezium, System                │ │
│  │  - Query Language: PromQL                             │ │
│  │  - Alerting: Can also send alerts (parallel)          │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## How It Works

### Monitoring Flow

```
┌──────────────────────────────────────────────────────────────┐
│  STEP 1: Metrics Generation                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Kafka Broker (JMX)                                    │  │
│  │  - kafka.server:type=BrokerTopicMetrics,name=...      │  │
│  │  - kafka.server:type=ReplicaManager,name=...          │  │
│  │  - kafka.network:type=RequestMetrics,name=...         │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 2: JMX Exporter Transformation                         │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Reads JMX MBeans via Java agent                      │  │
│  │  Applies pattern matching rules (exporter/*.yml)       │  │
│  │  Converts to Prometheus format                        │  │
│  │                                                        │  │
│  │  Example:                                              │  │
│  │  JMX:  kafka.server:type=BrokerTopicMetrics,          │  │
│  │        name=MessagesInPerSec,Count=12345               │  │
│  │                                                        │  │
│  │  Prometheus:                                           │  │
│  │  kafka_server_BrokerTopicMetrics_MessagesIn_total     │  │
│  │    {job="kafka-brokers", instance="broker-1:5556"}    │  │
│  │    12345                                               │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 3: Prometheus Scraping                                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  HTTP GET http://broker-1:5556/metrics                │  │
│  │                                                        │  │
│  │  Response:                                             │  │
│  │  # HELP kafka_server_BrokerTopicMetrics_MessagesIn... │  │
│  │  # TYPE kafka_server_BrokerTopicMetrics_MessagesIn... │  │
│  │  kafka_server_BrokerTopicMetrics_MessagesIn_total     │  │
│  │    {instance="broker-1:5556", job="kafka-brokers"}    │  │
│  │    12345 1698843000000                                │  │
│  │                                                        │  │
│  │  Stored in TSDB with timestamp                        │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 4: Rule Evaluation                                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Every 10 seconds, Prometheus evaluates rules:        │  │
│  │                                                        │  │
│  │  Rule: DebeziumInstanceDown                           │  │
│  │  expr: up{job="debezium"} == 0                        │  │
│  │  for: 1m                                               │  │
│  │                                                        │  │
│  │  Query Result:                                         │  │
│  │  up{instance="debezium:5556", job="debezium"} = 0     │  │
│  │                                                        │  │
│  │  Condition met for 1 minute → Fire alert!             │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 5: Alert Sending                                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Prometheus → Alertmanager (HTTP POST)                │  │
│  │                                                        │  │
│  │  {                                                     │  │
│  │    "status": "firing",                                 │  │
│  │    "labels": {                                         │  │
│  │      "alertname": "DebeziumInstanceDown",              │  │
│  │      "severity": "critical",                           │  │
│  │      "instance": "debezium:5556"                       │  │
│  │    },                                                  │  │
│  │    "annotations": {                                    │  │
│  │      "title": "Debezium Instance is DOWN",            │  │
│  │      "description": "Instance unreachable..."          │  │
│  │    }                                                   │  │
│  │  }                                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 6: Alertmanager Processing                             │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Group alerts (wait 20s for more)                  │  │
│  │  2. Apply inhibit rules (suppress duplicates)         │  │
│  │  3. Route to receiver: ggchat_alerts                  │  │
│  │  4. Send webhook to Calert                            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 7: Calert Formatting                                   │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Parse Alertmanager JSON                           │  │
│  │  2. Load template: templates/message.tmpl             │  │
│  │  3. Render template with alert data                   │  │
│  │  4. POST to Google Chat webhook                       │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────┐
│  STEP 8: Google Chat Notification                            │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  User sees message in Google Chat space               │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Prometheus

**Purpose**: Metrics collection, storage, and alerting engine.

**Image**: `prom/prometheus:v2.46.0`

**Key Features**:

- **Pull-based**: Scrapes metrics from exporters
- **Time Series Database**: Efficient storage with compression
- **PromQL**: Powerful query language
- **Alert Rules**: Evaluate conditions and fire alerts
- **Service Discovery**: Automatically discover targets

**Configuration**: `prometheus/prometheus.yml`

**Storage**: Volume `prometheus_data` (persistent across restarts)

**Command Line Options**:

```yaml
--config.file=/etc/prometheus/prometheus.yml   # Main config
--storage.tsdb.path=/prometheus                # Data directory
--web.config.file=/etc/prometheus/web.yml      # Auth config
--web.enable-lifecycle                         # Hot reload API
```

**Hot Reload**:

```bash
# Reload config without restart
curl -X POST http://localhost:9090/-/reload
```

### 2. Alertmanager

**Purpose**: Alert routing, grouping, and notification dispatch.

**Image**: `prom/alertmanager`

**Key Features**:

- **Grouping**: Combine related alerts
- **Inhibition**: Suppress lower-priority alerts
- **Silencing**: Temporarily mute alerts
- **Routing**: Send alerts to different receivers
- **Deduplication**: Prevent duplicate notifications

**Configuration**: `alertmanager/alertmanager.yml`

**Configuration Sections**:

```yaml
global:              # Default settings
inhibit_rules:       # Suppress alerts based on other alerts
route:               # How to route alerts
receivers:           # Where to send alerts (webhook, email, etc.)
templates:           # Custom message templates
```

### 3. Calert

**Purpose**: Webhook dispatcher for Google Chat (and other platforms).

**Image**: `ghcr.io/mr-karan/calert:latest`

**Why Use Calert?**

- Alertmanager webhook format ≠ Google Chat format
- Calert translates and formats messages
- Supports threaded replies (grouping related alerts)
- Retry logic for failed webhooks

**Configuration**: `calert/config.toml`

**Key Settings**:

```toml
[app]
address = "0.0.0.0:6000"                    # HTTP server
enable_request_logs = true                  # Log incoming webhooks

[providers.ggchat_alerts]
type = "google_chat"                        # Provider type
endpoint = "YOUR_GOOGLE_CHAT_WEBHOOK_URL"   # Webhook URL
template = "/app/templates/message.tmpl"    # Message template
threaded_replies = true                     # Group alerts in threads
thread_ttl = "12h"                          # Thread expiry
retry_max = 3                               # Retry failed sends
```

### 4. Grafana

**Purpose**: Visualization and dashboarding platform.

**Image**: `grafana/grafana:10.0.5`

**Key Features**:

- **Dashboards**: Visualize metrics with panels
- **Data Sources**: Connect to Prometheus, MySQL, etc.
- **Alerts**: Can also send alerts (parallel to Alertmanager)
- **Templating**: Dynamic dashboards with variables
- **Plugins**: Extend functionality

**Default Credentials**: admin/admin (change on first login)

**Storage**: Volume `grafana_storage` (dashboards, users, settings)

### 5. Node Exporter

**Purpose**: Collect system-level metrics (CPU, RAM, disk, network).

**Image**: `quay.io/prometheus/node-exporter:latest`

**What It Exports**:

- **CPU**: Usage, load average, context switches
- **Memory**: Total, available, swap
- **Disk**: Usage, I/O operations, throughput
- **Network**: Bytes in/out, packets, errors
- **Filesystem**: Mountpoints, inodes

**Why `--path.rootfs=/host`?**

- Container sees its own filesystem
- Mounting host root at `/host` exposes real system metrics

**Configuration**:

```yaml
volumes:
  - '/:/host:ro,rslave'      # Mount host root as read-only
command:
  - '--path.rootfs=/host'    # Tell exporter to read from /host
pid: host                    # Use host PID namespace
```

---

## Configuration Details

### Prometheus Configuration (`prometheus/prometheus.yml`)

```yaml
global:
  scrape_interval: 10s         # Default scrape frequency
  evaluation_interval: 10s     # How often to evaluate rules

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093    # Alertmanager endpoint

rule_files:
  - rules/node_exporter.yml      # System alert rules
  - rules/debezium.yml           # Debezium alert rules

scrape_configs:
  # Job 1: Prometheus itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
    basic_auth:
      username: "admin"
      password: "admin"

  # Job 2: System metrics
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node_exporter:9100"]
    basic_auth:
      username: "admin"
      password: "admin"
    relabel_configs:              # Extract instance name from address
    - source_labels: [__address__]
      separator: ':'
      regex: '(.*):(.*)'
      replacement: '${1}'
      target_label: instance

  # Job 3: Kafka brokers
  - job_name: "kafka-brokers"
    static_configs:
      - targets: ["broker-1:5556", "broker-2:5556", "broker-3:5556"]
        labels:
          role: "broker"
          cluster: "kraft-cluster"

  # Job 4: Kafka controllers
  - job_name: "kafka-controllers"
    static_configs:
      - targets: ["controller-1:5556", "controller-2:5556", "controller-3:5556"]
        labels:
          role: "controller"
          cluster: "kraft-cluster"

  # Job 5: Debezium
  - job_name: debezium
    scrape_interval: 5s           # More frequent (CDC is time-sensitive)
    static_configs:
      - targets: ["debezium:5556"]
```

**Configuration Explained**:

- **`scrape_interval`**: How often Prometheus fetches metrics (10s = every 10 seconds)
- **`evaluation_interval`**: How often alert rules are checked
- **`job_name`**: Logical grouping of targets (used in queries: `up{job="debezium"}`)
- **`targets`**: List of `host:port` endpoints to scrape
- **`labels`**: Additional labels attached to all metrics from this job
- **`basic_auth`**: Credentials for scraping (Prometheus web UI is protected)
- **`relabel_configs`**: Transform labels before storing metrics

### Basic Authentication (`prometheus/web.yml`)

```yaml
basic_auth_users:
  admin: $2b$12$fBKuqeUF63JpT3/0q7eDou2W7aJaDBfBBDAE9NaFFgTWUbjh59eVK
```

**What is this hash?**

- Bcrypt hash of password "admin"
- Protects Prometheus web UI from unauthorized access

**Generate new password**:

```bash
# Install htpasswd
sudo apt-get install apache2-utils

# Generate bcrypt hash
htpasswd -nbB admin your_password
```

**Access Prometheus**:

```bash
# Browser: http://localhost:9090 (prompts for admin/admin)
# Or use curl:
curl -u admin:admin http://localhost:9090/api/v1/query?query=up
```

### Alertmanager Configuration (`alertmanager/alertmanager.yml`)

```yaml
global:
  resolve_timeout: 5m            # Mark alert resolved after 5 min of silence

inhibit_rules:
  - source_match:
      severity: 'critical'       # If critical alert fires
    target_match:
      severity: 'warning'        # Suppress warning alerts
    equal: ['alertname', 'component', 'instance']  # With same labels

route:
  receiver: 'ggchat_alerts'      # Default receiver
  group_by: ['alertname', 'component', 'instance', 'severity']
  group_wait: 20s                # Wait 20s to collect similar alerts
  group_interval: 3m             # Min time between updates
  repeat_interval: 1h            # Resend alert every hour if still firing
```

**Configuration Explained**:

**`inhibit_rules`**:

- Prevents notification spam
- Example: If `DebeziumInstanceDown` (critical) fires, suppress `DebeziumHighLag` (warning) for same instance

**`group_by`**:

- Alerts with same values for these labels are grouped together
- Example: 3 alerts with `alertname=DebeziumInstanceDown` → 1 notification with 3 alerts

**`group_wait`**:

- Collect alerts for 20 seconds before sending
- Prevents sending multiple notifications for related alerts

**`group_interval`**:

- Minimum time between sending updates for same group
- Prevents notification spam when alerts keep firing/resolving

**`repeat_interval`**:

- Send reminder notification every hour if alert still firing
- Prevents forgetting about ongoing issues

### Calert Configuration (`calert/config.toml`)

```toml
[app]
address = "0.0.0.0:6000"
enable_request_logs = true
log = "info"

[providers.ggchat_alerts]
type = "google_chat"
endpoint = "https://chat.googleapis.com/v1/spaces/SPACE_ID/messages?key=KEY&token=TOKEN"
template = "/app/templates/message.tmpl"
threaded_replies = true
thread_ttl = "12h"
retry_max = 3
retry_wait_min = "1s"
retry_wait_max = "5s"
```

**Configuration Explained**:

- **`endpoint`**: Google Chat webhook URL (get from Google Chat space settings)
- **`template`**: Path to message template (Golang template format)
- **`threaded_replies`**: Group related alerts in same thread
- **`thread_ttl`**: Keep thread active for 12 hours, then start new thread
- **`retry_max`**: Retry failed webhook 3 times
- **`retry_wait_min/max`**: Exponential backoff between retries

**Message Template** (`calert/templates/message.tmpl`):

```go
🔥 *FIRING ALERT* 🔥

*({{ .Labels.severity | toUpper }}) {{ .Labels.alertname | Title }} - Firing*

{{ range .Annotations.SortedPairs -}}
*{{ .Name | Title }}:* {{ .Value }}
{{ end -}}

{{ if .Labels.instance }}*Instance:* {{ .Labels.instance }}{{ end }}
{{ if .Labels.job }}*Job:* {{ .Labels.job }}{{ end }}

*Started:* {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}
```

**Template Variables**:

- `{{ .Labels.severity }}`: Alert severity (critical/warning)
- `{{ .Labels.alertname }}`: Alert name
- `{{ .Annotations }}`: Alert annotations (title, description)
- `{{ .StartsAt }}`: When alert started firing

---

## Alert Rules

### Debezium Alert Rules (`prometheus/rules/debezium.yml`)

```yaml
groups:
- name: debezium_connection
  rules:
  # Alert 1: Instance Down
  - alert: DebeziumInstanceDown
    expr: up{job="debezium"} == 0
    for: 1m
    labels:
      severity: critical
      component: debezium
    annotations:
      title: "🔴 Debezium Instance is DOWN"
      description: |
        Debezium Connect instance is unreachable.
        Duration: More than 1 minute
        Impact: ALL CDC operations are stopped

  # Alert 2: Connector Disconnected
  - alert: DebeziumConnectorNotConnected
    expr: debezium_metrics_Connected{job="debezium"} == 0
    for: 30s
    labels:
      severity: critical
      component: debezium
    annotations:
      title: "🔴 Debezium Connector NOT CONNECTED"
      description: |
        Connector '{{ $labels.name }}' is NOT connected to database.
        This means CDC is NOT capturing changes!

  # Alert 3: No Changes Applied
  - alert: DebeziumConnectorNoChanges
    expr: debezium_metrics_ChangesApplied{job="debezium"} == 0
    for: 5m
    labels:
      severity: critical
      component: debezium
    annotations:
      title: "🔴 Debezium NOT APPLYING CHANGES"
      description: |
        Connector has not applied changes for 5+ minutes.
        Connector might be stuck or database has no activity.

  # Alert 4: Invalid Binlog Timestamp
  - alert: DebeziumInvalidBinlogTimestamp
    expr: debezium_metrics_MilliSecondsSinceLastAppliedChange{job="debezium"} < 0
    for: 2m
    labels:
      severity: warning
      component: debezium
    annotations:
      title: "⚠️ Debezium Invalid Binlog Timestamp"
      description: |
        Connector reports invalid timestamp ({{ $value }}).
        May have lost binlog position.

  # Alert 5: Metrics Disappeared
  - alert: DebeziumConnectorMetricsGone
    expr: |
      (
        count by (name) (debezium_metrics_Connected{context="streaming"} offset 5m)
        unless
        count by (name) (debezium_metrics_Connected{context="streaming"})
      ) > 0
    for: 2m
    labels:
      severity: critical
      component: debezium
    annotations:
      title: "🔴 Debezium Metrics Disappeared"
      description: "Connector {{ $labels.name }} stopped reporting metrics."
```

**Rule Breakdown**:

**Alert Structure**:

```yaml
- alert: AlertName                # Unique name
  expr: PromQL_query              # Condition to check
  for: 1m                         # Condition must be true for 1 minute
  labels:                         # Labels attached to alert
    severity: critical/warning
    component: debezium
  annotations:                    # Human-readable info
    title: "Alert Title"
    description: "Detailed description"
```

**PromQL Queries Explained**:

1. **`up{job="debezium"} == 0`**

   - `up` is automatically added by Prometheus (1 = scrape success, 0 = failure)
   - Checks if Debezium exporter is reachable
2. **`debezium_metrics_Connected{job="debezium"} == 0`**

   - Checks Debezium's connection status metric
   - 1 = connected, 0 = disconnected
3. **`debezium_metrics_ChangesApplied{job="debezium"} == 0`**

   - Checks if connector is applying changes
   - Remains 0 if connector is stuck or idle
4. **`debezium_metrics_MilliSecondsSinceLastAppliedChange < 0`**

   - Negative value indicates invalid state
   - Usually -1 means connector never processed changes
5. **Metrics Disappeared** (complex query):

   ```promql
   (
     count by (name) (debezium_metrics_Connected offset 5m)  # Metrics 5 min ago
     unless
     count by (name) (debezium_metrics_Connected)            # Metrics now
   ) > 0
   ```

   - Checks if metrics existed 5 minutes ago but not now
   - Indicates connector stopped or crashed

### Node Exporter Alert Rules (`prometheus/rules/node_exporter.yml`)

```yaml
groups:
  - name: node_exporter
    rules:
    # Alert 1: Node Down
    - alert: NodeDown
      expr: up == 0
      for: 1m
      labels:
        severity: critical
        component: node_exporter
      annotations:
        title: "Node {{ $labels.instance }} is down"
        description: "Failed to scrape for more than 1 minute."

    # Alert 2: High CPU Usage
    - alert: HighCPUUsage
      expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 2m
      labels:
        severity: warning
        component: node_exporter
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is {{ $value }}%"

    # Alert 3: High RAM Usage
    - alert: HighRAMUsage
      expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
      for: 2m
      labels:
        severity: warning
        component: node_exporter
      annotations:
        summary: "High RAM usage on {{ $labels.instance }}"
        description: "RAM usage is {{ $value }}%"

    # Alert 4: High Disk Usage
    - alert: HighDiskUsage
      expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100 > 80
      for: 2m
      labels:
        severity: warning
        component: node_exporter
      annotations:
        summary: "High disk usage on {{ $labels.instance }}"
        description: "Disk usage is {{ $value }}% on {{ $labels.device }}"

    # Alert 5: High Network Receive
    - alert: NetworkReceiveHigh
      expr: rate(node_network_receive_bytes_total[5m]) > 1000000
      for: 10m
      labels:
        severity: warning
        component: node_exporter
      annotations:
        summary: "High network receive rate"
        description: "Receive rate is {{ $value }} bytes/sec"

    # Alert 6: High Network Transmit
    - alert: NetworkTransmitHigh
      expr: rate(node_network_transmit_bytes_total[5m]) > 1000000
      for: 10m
      labels:
        severity: warning
        component: node_exporter
      annotations:
        summary: "High network transmit rate"
        description: "Transmit rate is {{ $value }} bytes/sec"
```

**PromQL Queries Explained**:

**1. CPU Usage**:

```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- `node_cpu_seconds_total{mode="idle"}`: Time CPU spent idle
- `irate(...[5m])`: Rate of change over last 5 minutes
- `100 - idle%`: Convert idle to busy percentage

**2. RAM Usage**:

```promql
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
```

- `MemTotal - MemAvailable`: Used memory
- Divide by total, multiply by 100 for percentage

**3. Disk Usage**:

```promql
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100
```

- `size - available`: Used space
- Divide by total, multiply by 100 for percentage

**4. Network Rate**:

```promql
rate(node_network_receive_bytes_total[5m])
```

- `rate()`: Calculate per-second rate over 5 minutes
- Returns bytes/second

---

## How to Use

### Step 1: Prerequisites

Ensure other services are running:

```bash
# Kafka cluster
cd ../cluster
docker-compose up -d

# Debezium
cd ../connect
docker-compose up -d
```

### Step 2: Start Monitoring Stack

```bash
cd monitor
docker-compose up -d
```

**Verify services**:

```bash
docker-compose ps

# Expected output:
# prometheus     Up    9090->9090
# grafana        Up    3000->3000
# alertmanager   Up    9094->9093
# calert         Up    6000->6000
# node-exporter  Up    9100->9100
```

### Step 3: Access Web UIs

**Prometheus**:

```bash
open http://localhost:9090
# Login: admin/admin
```

**Grafana**:

```bash
open http://localhost:3000
# Login: admin/admin (change on first login)
```

**Alertmanager**:

```bash
open http://localhost:9094
```

### Step 4: Configure Google Chat Webhook

**Get webhook URL**:

1. Go to Google Chat space
2. Click space name → "Apps & integrations"
3. Click "Add webhooks"
4. Copy webhook URL

**Update Calert config**:

```toml
# Edit calert/config.toml
[providers.ggchat_alerts]
endpoint = "YOUR_WEBHOOK_URL_HERE"
```

**Restart Calert**:

```bash
docker-compose restart calert
```

### Step 5: Test Alerts

**Trigger a test alert**:

```bash
# Stop Debezium to trigger alert
cd ../connect
docker-compose stop debezium

# Wait 1 minute, check Alertmanager UI
open http://localhost:9094/#/alerts

# Should see "DebeziumInstanceDown" firing
```

**Verify Google Chat notification**:

- Check your Google Chat space
- Should see alert message

**Resolve alert**:

```bash
# Restart Debezium
docker-compose start debezium

# Wait 5 minutes, alert should resolve
```

---

## Creating Dashboards

### Grafana Setup

**1. Add Prometheus Data Source**:

```bash
# Via UI:
1. Login to Grafana (http://localhost:3000)
2. Configuration → Data Sources → Add data source
3. Select "Prometheus"
4. URL: http://prometheus:9090
5. Auth: Basic auth
   - User: admin
   - Password: admin
6. Save & Test
```

**2. Import Dashboard**:

```bash
# Use community dashboards:
1. Dashboards → Import
2. Enter dashboard ID:
   - Kafka: 11699 (Strimzi Kafka)
   - Node Exporter: 1860 (Node Exporter Full)
3. Select Prometheus data source
4. Import
```

**3. Create Custom Dashboard**:

**Panel 1: Debezium Connection Status**:

```promql
debezium_metrics_Connected{job="debezium"}
```

- Type: Stat
- Thresholds: 0 = red, 1 = green

**Panel 2: CDC Lag**:

```promql
debezium_metrics_MilliSecondsSinceLastAppliedChange{job="debezium"}
```

- Type: Graph
- Unit: milliseconds

**Panel 3: Changes Applied**:

```promql
rate(debezium_metrics_ChangesApplied{job="debezium"}[5m])
```

- Type: Graph
- Unit: per second

**Panel 4: Kafka Message Rate**:

```promql
rate(kafka_server_BrokerTopicMetrics_MessagesIn_total[5m])
```

- Type: Graph
- Legend: {{ instance }}

**Panel 5: CPU Usage**:

```promql
100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Type: Gauge
- Thresholds: 0-50 green, 50-80 yellow, 80-100 red

---

## Troubleshooting

### Problem 1: Prometheus Can't Scrape Targets

**Symptoms**:

- Targets show as DOWN in http://localhost:9090/targets
- Errors: "context deadline exceeded", "connection refused"

**Solutions**:

**A. Check target is reachable**:

```bash
# From Prometheus container
docker exec prometheus wget -O- http://broker-1:5556/metrics

# Should return metrics in Prometheus format
```

**B. Check Docker network**:

```bash
# All services should be on 'monitoring' network
docker network inspect monitoring
```

**C. Check JMX exporter**:

```bash
# Verify exporter is running
docker exec broker-1 ps aux | grep javaagent
```

**D. Check firewall**:

```bash
# On host (Linux)
sudo ufw status

# Allow port if needed
sudo ufw allow 5556
```

### Problem 2: No Alerts Firing

**Symptoms**:

- Conditions are met but no alerts in Alertmanager
- No Google Chat notifications

**Solutions**:

**A. Check rule syntax**:

```bash
# Validate rules
docker exec prometheus promtool check rules /etc/prometheus/rules/debezium.yml
```

**B. Check rule evaluation**:

```bash
# Open Prometheus UI → Alerts
open http://localhost:9090/alerts

# Should show rules and their state (inactive/pending/firing)
```

**C. Check Alertmanager connection**:

```bash
# Prometheus UI → Status → Runtime & Build Information
# Should show Alertmanager URL

# Test Alertmanager API
curl http://localhost:9094/api/v1/alerts
```

**D. Check Alertmanager logs**:

```bash
docker logs alertmanager
```

### Problem 3: Calert Not Sending to Google Chat

**Symptoms**:

- Alerts fire in Alertmanager
- No messages in Google Chat

**Solutions**:

**A. Check Calert logs**:

```bash
docker logs calert

# Should show incoming webhooks and outgoing requests
```

**B. Test webhook manually**:

```bash
curl -X POST "YOUR_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"text": "Test message"}'

# Should appear in Google Chat
```

**C. Check Calert config**:

```bash
# Verify endpoint is correct
docker exec calert cat /app/config.toml | grep endpoint
```

**D. Check template syntax**:

```bash
# View template
docker exec calert cat /app/templates/message.tmpl

# Test rendering in Calert logs
docker logs calert | grep "rendering template"
```

### Problem 4: Grafana Can't Connect to Prometheus

**Symptoms**:

- "Data source is not working"
- Query returns no data

**Solutions**:

**A. Test Prometheus URL**:

```bash
# From Grafana container
docker exec grafana curl http://prometheus:9090/api/v1/query?query=up
```

**B. Check basic auth**:

```bash
# Should match prometheus/web.yml
Username: admin
Password: admin
```

**C. Check Prometheus logs**:

```bash
docker logs prometheus | grep "401\|403"
```

**D. Recreate data source**:

- Delete and re-add Prometheus data source in Grafana

### Problem 5: Metrics Not Appearing

**Symptoms**:

- Query returns empty result
- Metrics were there before

**Solutions**:

**A. Check metric name**:

```bash
# List all metrics
curl -u admin:admin http://localhost:9090/api/v1/label/__name__/values

# Search for metric
curl -u admin:admin 'http://localhost:9090/api/v1/query?query=debezium_metrics_Connected'
```

**B. Check scrape errors**:

```bash
# Prometheus UI → Targets
# Look for errors in "Last Scrape" column
```

**C. Check time range**:

- Metrics might be outside selected time range
- Try "Last 5 minutes" in Grafana/Prometheus

**D. Check if service restarted**:

```bash
# Metrics are lost when service restarts (if no persistence)
docker ps -a | grep -E "broker|debezium"
```

---

## Production Considerations

### 1. Prometheus Retention

**Default**: 15 days

**Increase retention**:

```yaml
command:
  - "--storage.tsdb.retention.time=30d"      # 30 days
  - "--storage.tsdb.retention.size=50GB"     # Or 50GB (whichever comes first)
```

**Monitor storage**:

```promql
# Disk usage
prometheus_tsdb_storage_blocks_bytes / 1024 / 1024 / 1024
```

### 2. High Availability

**Run multiple Prometheus instances**:

- Same config, scrape same targets
- External load balancer routes queries
- Deduplication in queries

**Alertmanager cluster**:

```yaml
# alertmanager.yml
cluster:
  peers:
    - alertmanager-1:9094
    - alertmanager-2:9094
    - alertmanager-3:9094
```

### 3. Security

**Enable HTTPS**:

```yaml
# prometheus/web.yml
tls_server_config:
  cert_file: /etc/prometheus/prometheus.crt
  key_file: /etc/prometheus/prometheus.key
```

**Strong passwords**:

```bash
# Generate secure password
htpasswd -nbB admin $(openssl rand -base64 32)
```

**Network isolation**:

```yaml
# Only expose necessary ports
ports:
  - "9090:9090"   # Prometheus (restrict to VPN)
  - "3000:3000"   # Grafana (restrict to VPN)
  # Don't expose 5556, 9100, etc. publicly
```

### 4. Performance Tuning

**Prometheus**:

```yaml
command:
  - "--storage.tsdb.path=/prometheus"
  - "--storage.tsdb.wal-compression"          # Compress write-ahead log
  - "--query.max-concurrency=50"              # Max concurrent queries
  - "--query.timeout=2m"                      # Query timeout
```

**Reduce cardinality**:

- Avoid high-cardinality labels (user IDs, timestamps, etc.)
- Use recording rules for expensive queries

**Recording rules** (pre-compute metrics):

```yaml
# prometheus/rules/recording.yml
groups:
  - name: recording
    interval: 10s
    rules:
      - record: job:kafka_messages_in:rate5m
        expr: rate(kafka_server_BrokerTopicMetrics_MessagesIn_total[5m])
```

### 5. Backup and Restore

**Backup Prometheus data**:

```bash
# Snapshot API
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Creates snapshot in /prometheus/snapshots/
# Copy snapshot directory to backup location
```

**Restore**:

```bash
# Stop Prometheus
docker-compose stop prometheus

# Copy snapshot to data directory
cp -r snapshot_dir/* prometheus_data/

# Start Prometheus
docker-compose start prometheus
```

**Backup Grafana**:

```bash
# Export dashboards via API
curl -u admin:admin http://localhost:3000/api/search?type=dash-db | jq

# Backup database
docker exec grafana cp /var/lib/grafana/grafana.db /tmp/
docker cp grafana:/tmp/grafana.db ./grafana_backup.db
```

### 6. Alert Fatigue Prevention

**Tune thresholds**:

- Start with loose thresholds
- Tighten based on baseline behavior
- Use percentiles: `> quantile(0.95, metric)`

**Use inhibit rules**:

- Suppress low-severity when high-severity fires
- Suppress dependent alerts (e.g., if broker down, suppress topic alerts)

**Group related alerts**:

- Group by component, instance
- Single notification for related issues

**Set appropriate `for` duration**:

- Too short: Alert on temporary blips
- Too long: Miss critical issues
- Start with 2-5 minutes for most alerts

---

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Prometheus Query Language (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Calert GitHub](https://github.com/mr-karan/calert)
- [Node Exporter Metrics](https://github.com/prometheus/node_exporter)
- [Debezium monitoring](https://github.com/Redislabs-Solution-Architects/debezium-monitoring)

---

**Happy Monitoring! 📊🔔**
