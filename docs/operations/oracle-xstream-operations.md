# Oracle XStream CDC Pipeline - Operations Guide
## Docker-Based Kafka Environment

**Environment:** Oracle 19c RAC + Docker Kafka Stack  
**Created:** April 17, 2026  
**Version:** 2.0 (Docker Edition)

---

## 1. DOCKER CONTAINER ARCHITECTURE

### 1.1 Container Overview

Your CDC pipeline runs on a **hybrid architecture**:
- **Oracle Database**: Native Oracle 19c RAC (2 nodes) on bare metal/VMs
- **Kafka Ecosystem**: Fully containerized using Docker
- **Monitoring Stack**: Docker-based Grafana/Prometheus

```
┌─────────────────────────────────────────────────────────────────┐
│                     ORACLE 19c RAC CLUSTER                       │
│                    (Native - Not Docker)                         │
│  ┌──────────────┐              ┌──────────────┐                 │
│  │   RAC Node 1 │              │   RAC Node 2 │                 │
│  │  XStream Cap │◄────────────►│  XStream Cap │                 │
│  │  Outbound Srv│              │  (Standby)   │                 │
│  └──────┬───────┘              └──────────────┘                 │
│         │                                                        │
│         │ XStream TCP Connection (Port 1522)                    │
│         │                                                        │
└─────────┼────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│               DOCKER KAFKA ECOSYSTEM                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kafka Connect (container: connect)                      │   │
│  │  - Oracle XStream Source Connector                       │   │
│  │  - REST API: localhost:8083                              │   │
│  │  - JMX Metrics: localhost:9994                           │   │
│  └───────────────────────┬──────────────────────────────────┘   │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Apache Kafka Cluster (3 brokers)                        │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐         │   │
│  │  │  kafka1    │  │  kafka2    │  │  kafka3    │         │   │
│  │  │  :9092     │  │  :9094     │  │  :9095     │         │   │
│  │  │  JMX :9990 │  │  JMX :9991 │  │  JMX :9992 │         │   │
│  │  └────────────┘  └────────────┘  └────────────┘         │   │
│  └───────────────────────┬──────────────────────────────────┘   │
│                          │                                       │
│  ┌───────────────────────┴──────────────────────────────────┐   │
│  │  Schema Registry (container: schema-registry)            │   │
│  │  - Avro schema storage                                   │   │
│  │  - REST API: localhost:8081                              │   │
│  │  - JMX Metrics: localhost:9993                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│               MONITORING STACK (Docker)                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prometheus (prometheus)                                 │   │
│  │  - Metrics database                                      │   │
│  │  - Port: 9090                                            │   │
│  │  - Scrapes: Kafka JMX, Connect JMX, kafka-exporter      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Grafana (grafana)                                       │   │
│  │  - Visualization dashboards                              │   │
│  │  - Port: 3000                                            │   │
│  │  - Data source: Prometheus, Loki                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Loki (loki)                                             │   │
│  │  - Log aggregation backend                               │   │
│  │  - Port: 3100                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Promtail (promtail)                                     │   │
│  │  - Log shipper agent                                     │   │
│  │  - Sends logs to Loki                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Kafka Exporter (kafka-exporter)                         │   │
│  │  - Exports Kafka metrics to Prometheus                   │   │
│  │  - Port: 9308                                            │   │
│  │  - Metrics: lag, offset, partition count                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 1.2 Container Details

#### **Kafka Brokers (kafka1, kafka2, kafka3)**
```bash
Image: kafka-jmx:7.9.0 (Confluent Platform 7.9.0)
Ports:
  - kafka1: 9092 (broker), 9990 (JMX)
  - kafka2: 9094 (broker), 9991 (JMX)
  - kafka3: 9095 (broker), 9992 (JMX)
Purpose: 
  - Store CDC events from Oracle XStream
  - Provide high availability (3-node cluster)
  - Expose JMX metrics for monitoring
Network: Likely bridge or custom Docker network
```

**What's Special:**
- Custom image `kafka-jmx:7.9.0` includes JMX exporter for Prometheus integration
- Each broker has unique JMX port for individual monitoring
- 3-broker setup provides fault tolerance (can lose 1 broker)

---

#### **Kafka Connect (connect)**
```bash
Image: docker-connect (custom build)
Ports:
  - 8083 (REST API)
  - 9994 (JMX metrics)
Purpose:
  - Runs Oracle XStream Source Connector
  - Pulls CDC events from Oracle XStream Outbound Server
  - Publishes events to Kafka topics
  - Transforms Oracle LogMiner records to Kafka messages
```

**What's Special:**
- Custom image includes Oracle XStream JDBC drivers
- Connects to Oracle via TNS (<database-name>PDB_POC service)
- REST API on 8083 for connector management
- Single Connect worker (not distributed mode in this setup)

**Key Configuration:**
- Connector: `confluent-xstream-source`
- Topics created: `<schema-name>.MTX_TRANSACTION_ITEMS` (and potentially 22 more MTX tables)
- Batch size: 5000 events
- Poll interval: 1000ms

---

#### **Schema Registry (schema-registry)**
```bash
Image: schema-registry-jmx:7.9.0
Ports:
  - 8081 (REST API)
  - 9993 (JMX metrics)
Purpose:
  - Store and serve Avro schemas for CDC events
  - Schema evolution management
  - Schema compatibility checks
```

**What's Special:**
- JMX-enabled for monitoring
- Stores schemas for all XStream topics
- Ensures schema compatibility during connector updates

---

#### **Kafka Exporter (kafka-exporter)**
```bash
Image: danielqsj/kafka-exporter:latest
Port: 9308 (Prometheus metrics)
Purpose:
  - Export Kafka-specific metrics to Prometheus
  - Monitor consumer lag, partition offsets, topic metrics
  - Complement JMX metrics with Kafka internals
```

**What It Monitors:**
- Consumer group lag (critical for CDC latency)
- Topic partition high-water marks
- Under-replicated partitions
- Broker availability

---

#### **Prometheus (prometheus)**
```bash
Image: prom/prometheus:v2.47.0
Port: 9090 (web UI and API)
Purpose:
  - Scrape metrics from Kafka JMX exporters (9990-9994)
  - Scrape metrics from kafka-exporter (9308)
  - Time-series metrics storage
  - Query engine for Grafana
```

**Scrape Targets:**
- kafka1:9990, kafka2:9991, kafka3:9992 (broker JMX)
- connect:9994 (Connect JMX)
- schema-registry:9993 (Schema Registry JMX)
- kafka-exporter:9308 (Kafka-specific metrics)

---

#### **Grafana (grafana)**
```bash
Image: grafana/grafana:10.2.0
Port: 3000 (web UI)
Purpose:
  - Visualize metrics from Prometheus
  - Display logs from Loki
  - Custom dashboards for CDC pipeline monitoring
```

**Access:** http://<kafka-vm-ip>:3000

**Dashboards (Likely):**
- Kafka Cluster Overview
- Kafka Connect Metrics
- XStream CDC Pipeline
- Oracle Redo Generation
- Consumer Lag Monitoring

---

#### **Loki (loki)**
```bash
Image: grafana/loki:2.9.0
Port: 3100 (API)
Purpose:
  - Log aggregation backend
  - Store logs from Kafka, Connect, Oracle (via Promtail)
  - Provide log search and querying for Grafana
```

**Log Sources:**
- Kafka broker logs (via Promtail)
- Kafka Connect logs (via Promtail)
- Container stdout/stderr

---

#### **Promtail (promtail)**
```bash
Image: grafana/promtail:2.9.0
Purpose:
  - Tail Docker container logs
  - Ship logs to Loki
  - Add labels (container name, service, etc.)
```

**What It Tails:**
- `/var/log/containers/*.log` (Docker container logs)
- Sends to Loki at localhost:3100

---

### 1.3 View All Containers

```bash
# List all running containers
docker ps

# Expected output (your environment):
CONTAINER ID   IMAGE                              PORTS                    NAMES
a1b2c3d4e5f6   grafana/promtail:2.9.0                                      promtail
b2c3d4e5f6a7   grafana/grafana:10.2.0             0.0.0.0:3000->3000/tcp   grafana
c3d4e5f6a7b8   prom/prometheus:v2.47.0            0.0.0.0:9090->9090/tcp   prometheus
d4e5f6a7b8c9   grafana/loki:2.9.0                 0.0.0.0:3100->3100/tcp   loki
e5f6a7b8c9d0   kafka-jmx:7.9.0                    9094/tcp, 9991/tcp       kafka2
f6a7b8c9d0e1   kafka-jmx:7.9.0                    9095/tcp, 9992/tcp       kafka3
a7b8c9d0e1f2   kafka-jmx:7.9.0                    9092/tcp, 9990/tcp       kafka1
b8c9d0e1f2a3   docker-connect                     8083/tcp, 9994/tcp       connect
c9d0e1f2a3b4   schema-registry-jmx:7.9.0          8081/tcp, 9993/tcp       schema-registry
d0e1f2a3b4c5   danielqsj/kafka-exporter           9308/tcp                 kafka-exporter
```

---

## 2. ORACLE DATABASE OPERATIONS

### 2.1 Oracle Database Status

**Check Oracle RAC cluster status:**
```bash
# Run on Oracle database server
srvctl status database -d <database-name>

# Expected output:
Instance <database-name>1 is running on node rac1
Instance <database-name>2 is running on node rac2
```

**Check specific instance status:**
```bash
srvctl status instance -d <database-name> -i <database-name>1
srvctl status instance -d <database-name> -i <database-name>2
```

**Check archive log mode:**
```bash
sqlplus / as sysdba <<EOF
SELECT log_mode, force_logging FROM v\$database;
EXIT;
EOF

# Expected output:
LOG_MODE     FORCE_LOGGING
------------ --------------
ARCHIVELOG   YES
```

---

### 2.2 XStream Capture Status

**Check XStream capture process:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LINESIZE 200
COL capture_name FORMAT A20
COL status FORMAT A10
COL state FORMAT A15
COL total_messages_captured FORMAT 999,999,999,999

SELECT capture_name, 
       status, 
       state, 
       total_messages_captured,
       startup_time
FROM v$xstream_capture 
WHERE capture_name = '<xstream-outbound-name>';

EXIT;
EOF
```

**Expected output:**
```
CAPTURE_NAME         STATUS     STATE           TOTAL_MESSAGES_CAPTURED STARTUP_TIME
-------------------- ---------- --------------- ----------------------- -------------------
<xstream-outbound-name>      ENABLED    CAPTURING                  14,019,801  16-APR-26 12:29:00
```

**If STATUS is DISABLED:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
BEGIN
  DBMS_XSTREAM_ADM.START_CAPTURE(capture_name => '<xstream-outbound-name>');
END;
/
EXIT;
EOF
```

---

### 2.3 XStream Outbound Server Status

**Check outbound server:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LINESIZE 200
COL server_name FORMAT A20
COL connect_user FORMAT A20
COL status FORMAT A10
COL state FORMAT A15

SELECT server_name,
       connect_user,
       status,
       state,
       total_messages_sent,
       startup_time
FROM v$xstream_outbound_server
WHERE server_name = '<xstream-outbound-name>';

EXIT;
EOF
```

**Expected output:**
```
SERVER_NAME          CONNECT_USER         STATUS     STATE           TOTAL_MESSAGES_SENT
-------------------- -------------------- ---------- --------------- -------------------
<xstream-outbound-name>      <xstream-connect-user>   ENABLED    SENDING                14,019,801
```

**If STATE is WAITING FOR CLIENT:**
- This means Kafka Connect is not connected
- Check Kafka Connect container status (section 3.2)

---

### 2.4 Redo Generation Rate

**Real-time redo generation (run during test):**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LINESIZE 200
COL thread# FORMAT 99
COL redo_mb_per_sec FORMAT 999,999.99

-- Sample redo rate every 10 seconds
SELECT s.thread#,
       ROUND((s.value - LAG(s.value) OVER (PARTITION BY s.thread# ORDER BY SYSDATE)) / 1024 / 1024 / 10, 2) AS redo_mb_per_sec
FROM (
  SELECT thread#, value 
  FROM v$sysstat 
  WHERE name = 'redo size'
) s;

EXIT;
EOF
```

**Archive log generation (post-test):**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LINESIZE 200
COL thread# FORMAT 99
COL gb FORMAT 999,999.99
COL log_switches FORMAT 999,999

-- Get redo for April 16 test (00:29:00 - 00:52:00)
SELECT thread#,
       ROUND(SUM(blocks * block_size) / 1024 / 1024 / 1024, 2) AS gb,
       COUNT(*) AS log_switches
FROM v$archived_log
WHERE completion_time >= TO_TIMESTAMP('2026-04-16 00:29:00', 'YYYY-MM-DD HH24:MI:SS')
  AND completion_time <= TO_TIMESTAMP('2026-04-16 00:52:00', 'YYYY-MM-DD HH24:MI:SS')
GROUP BY thread#;

EXIT;
EOF
```

**Expected output (April 16 test):**
```
THREAD#         GB LOG_SWITCHES
------- ---------- ------------
      1      28.15           35
      2      27.85           35
------- ---------- ------------
  Total      56.00           70
```

---

### 2.5 Start/Stop Oracle Database

**Stop Oracle RAC database:**
```bash
# Run as Oracle user (usually 'oracle')
srvctl stop database -d <database-name>

# Verify
srvctl status database -d <database-name>
```

**Start Oracle RAC database:**
```bash
srvctl start database -d <database-name>

# Verify
srvctl status database -d <database-name>
```

**Stop single instance:**
```bash
srvctl stop instance -d <database-name> -i <database-name>1
```

**Start single instance:**
```bash
srvctl start instance -d <database-name> -i <database-name>1
```

---

### 2.6 Start/Stop XStream Components

**Stop XStream capture:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
BEGIN
  DBMS_XSTREAM_ADM.STOP_CAPTURE(capture_name => '<xstream-outbound-name>');
END;
/
EXIT;
EOF
```

**Start XStream capture:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
BEGIN
  DBMS_XSTREAM_ADM.START_CAPTURE(capture_name => '<xstream-outbound-name>');
END;
/
EXIT;
EOF
```

**Restart XStream (stop + start):**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
BEGIN
  DBMS_XSTREAM_ADM.STOP_CAPTURE(capture_name => '<xstream-outbound-name>');
  DBMS_LOCK.SLEEP(5);
  DBMS_XSTREAM_ADM.START_CAPTURE(capture_name => '<xstream-outbound-name>');
END;
/
EXIT;
EOF
```

---

## 3. KAFKA ECOSYSTEM OPERATIONS (DOCKER)

### 3.1 Kafka Broker Operations

**Check Kafka broker status:**
```bash
# List all Kafka containers
docker ps --filter "name=kafka" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Expected output:
NAMES    STATUS         PORTS
kafka1   Up 2 hours     9092/tcp, 9990/tcp
kafka2   Up 2 hours     9094/tcp, 9991/tcp
kafka3   Up 2 hours     9095/tcp, 9992/tcp
```

**Start Kafka brokers:**
```bash
# Start all 3 brokers
docker start kafka1 kafka2 kafka3

# Start individually
docker start kafka1
docker start kafka2
docker start kafka3
```

**Stop Kafka brokers:**
```bash
# Stop all 3 brokers
docker stop kafka1 kafka2 kafka3

# Stop individually (graceful shutdown)
docker stop -t 30 kafka1
docker stop -t 30 kafka2
docker stop -t 30 kafka3
```

**Restart Kafka brokers:**
```bash
# Restart all
docker restart kafka1 kafka2 kafka3

# Restart individually
docker restart kafka1
```

**View Kafka broker logs:**
```bash
# Real-time logs (kafka1)
docker logs -f kafka1

# Last 100 lines
docker logs --tail 100 kafka1

# Logs with timestamps
docker logs -f --timestamps kafka1

# Search for errors
docker logs kafka1 2>&1 | grep -i error
```

---

**Execute Kafka commands inside container:**
```bash
# List topics
docker exec kafka1 kafka-topics --bootstrap-server localhost:9092 --list

# Describe specific topic
docker exec kafka1 kafka-topics --bootstrap-server localhost:9092 --describe \
  --topic <schema-name>.MTX_TRANSACTION_ITEMS

# Get topic message count
docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic <schema-name>.MTX_TRANSACTION_ITEMS \
  --time -1

# Consumer group lag
docker exec kafka1 kafka-consumer-groups --bootstrap-server localhost:9092 --list
docker exec kafka1 kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group xstream-connector-group --describe
```

---

### 3.2 Kafka Connect Operations

**Check Kafka Connect status:**
```bash
# Container status
docker ps --filter "name=connect"

# Expected output:
CONTAINER ID   IMAGE           STATUS         PORTS                    NAMES
b8c9d0e1f2a3   docker-connect  Up 3 hours     8083/tcp, 9994/tcp       connect
```

**Start Kafka Connect:**
```bash
docker start connect
```

**Stop Kafka Connect:**
```bash
# Graceful shutdown (30 second timeout)
docker stop -t 30 connect
```

**Restart Kafka Connect:**
```bash
docker restart connect
```

**View Kafka Connect logs:**
```bash
# Real-time logs
docker logs -f connect

# Last 200 lines
docker logs --tail 200 connect

# Search for connector errors
docker logs connect 2>&1 | grep -i "error\|exception\|failed"
```

---

**Kafka Connect REST API operations:**
```bash
# Check Connect health
curl -s http://localhost:8083/ | jq .

# List connectors
curl -s http://localhost:8083/connectors | jq .

# Get connector status
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq .

# Expected output:
{
  "name": "confluent-xstream-source",
  "connector": {
    "state": "RUNNING",
    "worker_id": "connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "connect:8083"
    }
  ]
}

# Get connector configuration
curl -s http://localhost:8083/connectors/confluent-xstream-source/config | jq .

# Pause connector (stop consuming from Oracle)
curl -X PUT http://localhost:8083/connectors/confluent-xstream-source/pause

# Resume connector
curl -X PUT http://localhost:8083/connectors/confluent-xstream-source/resume

# Restart connector
curl -X POST http://localhost:8083/connectors/confluent-xstream-source/restart

# Restart specific task
curl -X POST http://localhost:8083/connectors/confluent-xstream-source/tasks/0/restart

# Delete connector (WARNING: will lose CDC position)
curl -X DELETE http://localhost:8083/connectors/confluent-xstream-source
```

---

### 3.3 Schema Registry Operations

**Check Schema Registry status:**
```bash
# Container status
docker ps --filter "name=schema-registry"

# REST API health check
curl -s http://localhost:8081/ | jq .
```

**Start/Stop/Restart:**
```bash
docker start schema-registry
docker stop -t 30 schema-registry
docker restart schema-registry
```

**View logs:**
```bash
docker logs -f schema-registry
```

**Schema operations:**
```bash
# List all subjects (schemas)
curl -s http://localhost:8081/subjects | jq .

# Get schema for specific subject
curl -s http://localhost:8081/subjects/<schema-name>.MTX_TRANSACTION_ITEMS-value/versions/latest | jq .

# List all schema versions
curl -s http://localhost:8081/subjects/<schema-name>.MTX_TRANSACTION_ITEMS-value/versions | jq .
```

---

### 3.4 Complete Startup Sequence

**Recommended startup order:**
```bash
#!/bin/bash
# Start Kafka ecosystem in correct order

echo "Step 1: Start Schema Registry"
docker start schema-registry
sleep 5

echo "Step 2: Start Kafka brokers (all 3)"
docker start kafka1 kafka2 kafka3
sleep 10

echo "Step 3: Wait for Kafka cluster to form"
for i in {1..30}; do
  if docker exec kafka1 kafka-broker-api-versions --bootstrap-server localhost:9092 &>/dev/null; then
    echo "Kafka cluster ready"
    break
  fi
  echo "Waiting for Kafka... ($i/30)"
  sleep 2
done

echo "Step 4: Start Kafka Connect"
docker start connect
sleep 10

echo "Step 5: Verify connector status"
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq .

echo "Startup complete!"
```

---

### 3.5 Complete Shutdown Sequence

**Recommended shutdown order (reverse of startup):**
```bash
#!/bin/bash
# Graceful shutdown of Kafka ecosystem

echo "Step 1: Pause Kafka Connect connector"
curl -X PUT http://localhost:8083/connectors/confluent-xstream-source/pause
sleep 5

echo "Step 2: Stop Kafka Connect"
docker stop -t 30 connect

echo "Step 3: Stop Kafka brokers"
docker stop -t 30 kafka1 kafka2 kafka3

echo "Step 4: Stop Schema Registry"
docker stop -t 30 schema-registry

echo "Shutdown complete!"
```

---

## 4. MONITORING STACK OPERATIONS

### 4.1 Grafana Operations

**Check Grafana status:**
```bash
docker ps --filter "name=grafana"

# Access web UI
# URL: http://<kafka-vm-ip>:3000
# Default credentials: admin/admin (likely changed)
```

**Start/Stop/Restart:**
```bash
docker start grafana
docker stop grafana
docker restart grafana
```

**View logs:**
```bash
docker logs -f grafana
```

**Access Grafana dashboards:**
```bash
# Open in browser
open http://<kafka-vm-ip>:3000

# List dashboards via API
curl -s http://admin:admin@localhost:3000/api/search | jq .
```

---

### 4.2 Prometheus Operations

**Check Prometheus status:**
```bash
docker ps --filter "name=prometheus"

# Access web UI
# URL: http://localhost:9090
```

**Start/Stop/Restart:**
```bash
docker start prometheus
docker stop prometheus
docker restart prometheus
```

**View logs:**
```bash
docker logs -f prometheus
```

**Check scrape targets:**
```bash
# Via web UI
open http://localhost:9090/targets

# Via API
curl -s http://localhost:9090/api/v1/targets | jq .
```

**Query Prometheus metrics (examples):**
```bash
# Kafka broker JMX metrics
curl -s 'http://localhost:9090/api/v1/query?query=kafka_server_brokertopicmetrics_messagesinpersec' | jq .

# Kafka Connect task status
curl -s 'http://localhost:9090/api/v1/query?query=kafka_connect_connector_status' | jq .

# Consumer lag
curl -s 'http://localhost:9090/api/v1/query?query=kafka_consumergroup_lag' | jq .
```

---

### 4.3 Loki and Promtail Operations

**Check Loki status:**
```bash
docker ps --filter "name=loki"
```

**Start/Stop/Restart:**
```bash
docker start loki
docker stop loki
docker restart loki
```

**View Loki logs:**
```bash
docker logs -f loki
```

**Check Promtail status:**
```bash
docker ps --filter "name=promtail"
```

**Start/Stop/Restart Promtail:**
```bash
docker start promtail
docker stop promtail
docker restart promtail
```

**Query Loki logs (via CLI or Grafana):**
```bash
# Example: query Kafka Connect logs
# This is typically done via Grafana Explore UI
# Loki API endpoint: http://localhost:3100
```

---

### 4.4 Kafka Exporter Operations

**Check Kafka Exporter status:**
```bash
docker ps --filter "name=kafka-exporter"
```

**Start/Stop/Restart:**
```bash
docker start kafka-exporter
docker stop kafka-exporter
docker restart kafka-exporter
```

**View logs:**
```bash
docker logs -f kafka-exporter
```

**Check exported metrics:**
```bash
# Kafka Exporter exposes metrics on port 9308
curl -s http://localhost:9308/metrics | grep kafka

# Consumer lag metrics
curl -s http://localhost:9308/metrics | grep kafka_consumergroup_lag

# Topic partition metrics
curl -s http://localhost:9308/metrics | grep kafka_topic_partitions
```

---

### 4.5 Monitoring Stack Startup/Shutdown

**Start all monitoring containers:**
```bash
docker start prometheus grafana loki promtail kafka-exporter

# Verify
docker ps --filter "name=prometheus|grafana|loki|promtail|kafka-exporter"
```

**Stop all monitoring containers:**
```bash
docker stop prometheus grafana loki promtail kafka-exporter
```

---

## 5. HAMMERDB LOAD TEST OPERATIONS

### 5.1 HammerDB Installation and Setup

**Installation:**
```bash
# Download HammerDB (already installed in your environment)
cd /opt
wget https://github.com/TPC-Council/HammerDB/releases/download/v4.10/HammerDB-4.10-Linux.tar.gz
tar -xzf HammerDB-4.10-Linux.tar.gz
cd HammerDB-4.10

# Create custom MTX driver directory
mkdir -p /opt/HammerDB-4.10/custom/mtx
```

**Copy MTX custom driver:**
```bash
# Your custom driver: hammerdb-mtx-custom-driver.tcl
# Location: /opt/HammerDB-4.10/custom/mtx/mtx_driver.tcl
```

---

### 5.2 Running MTX Load Test

**Test execution flow:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    HAMMERDB EXECUTION FLOW                      │
└─────────────────────────────────────────────────────────────────┘

1. Launch HammerDB
   ├─► ./hammerdbcli (CLI mode)
   └─► OR ./hammerdbgui (GUI mode)

2. Load Custom Driver
   ├─► source /opt/HammerDB-4.10/custom/mtx/mtx_driver.tcl
   └─► Loads MTX INSERT workload

3. Configure Test Parameters
   ├─► Virtual Users: 48
   ├─► Iterations: 292,078 per VU
   ├─► Total rows: 48 × 292,078 = 14,019,744
   └─► Row size: ~4KB each

4. Build Schema (Optional - Already Done)
   ├─► Create MTX_TRANSACTION_ITEMS table
   ├─► Add supplemental logging
   └─► Baseline row count: 3,493,457

5. Run Workload
   ├─► Start 48 virtual users
   ├─► Each VU connects to Oracle RAC
   ├─► Executes INSERT loop
   └─► Duration: ~22 minutes

6. Monitor Progress
   ├─► Watch Oracle sessions: v$session
   ├─► Watch XStream capture: v$xstream_capture
   ├─► Watch Kafka topics: kafka-console-consumer
   └─► Watch Grafana dashboards

7. Collect Results
   ├─► Row count: SELECT COUNT(*) FROM MTX_TRANSACTION_ITEMS
   ├─► Redo generated: v$archived_log
   ├─► Throughput: total_rows / duration_seconds
   └─► XStream lag: v$xstream_outbound_server

8. Verify CDC Pipeline
   ├─► Kafka message count = Oracle row count
   ├─► No errors in Connect logs
   ├─► Consumer lag < 100ms
   └─► Test PASSED ✅
```

---

**Execute load test (CLI mode):**
```bash
cd /opt/HammerDB-4.10

# Start HammerDB CLI
./hammerdbcli

# Inside HammerDB CLI:
source /opt/HammerDB-4.10/custom/mtx/mtx_driver.tcl
puts "MTX driver loaded"

# Configure
set VIRTUAL_USERS 48
set ITERATIONS 292078
set ORACLE_SERVICE "<database-name>PDB_POC"
set ORACLE_USER "ordermgmt"
set ORACLE_PASSWORD "ordermgmt"

# Run test
run_mtx_test

# Exit
quit
```

---

**Monitor test in real-time:**
```bash
# Terminal 1: Watch Oracle sessions
watch -n 5 'sqlplus -S sys/<sys-password>@<database-name>PDB_POC as sysdba <<< "
  SELECT COUNT(*) AS active_sessions 
  FROM v\$session 
  WHERE username = '\''<schema-name>'\'' 
  AND status = '\''ACTIVE'\'';
"'

# Terminal 2: Watch row count
watch -n 10 'sqlplus -S <schema-user>/<schema-password>@<database-name>PDB_POC <<< "
  SELECT COUNT(*) FROM mtx_transaction_items;
"'

# Terminal 3: Watch Kafka topic message count
watch -n 10 'docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic <schema-name>.MTX_TRANSACTION_ITEMS \
  --time -1 | awk -F: '\''{sum += $3} END {print "Total messages:", sum}'\'
```

---

### 5.3 Post-Test Verification

**Verify row count:**
```bash
sqlplus <schema-user>/<schema-password>@<database-name>PDB_POC <<'EOF'
SELECT TO_CHAR(COUNT(*), '999,999,999,999') AS total_rows 
FROM mtx_transaction_items;
EXIT;
EOF

# Expected: 17,513,258 (baseline 3,493,457 + test 14,019,801)
```

**Verify Kafka messages:**
```bash
# Get total message count in Kafka topic
docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic <schema-name>.MTX_TRANSACTION_ITEMS \
  --time -1 | awk -F: '{sum += $3} END {print "Total messages:", sum}'

# Expected: 14,019,801 (should match Oracle insert count)
```

**Verify XStream statistics:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SELECT server_name,
       total_messages_sent,
       total_messages_sent - LAG(total_messages_sent) OVER (ORDER BY startup_time) AS messages_this_test
FROM v$xstream_outbound_server
WHERE server_name = '<xstream-outbound-name>';
EXIT;
EOF
```

---

## 6. HEALTH CHECKS

### 6.1 Complete System Health Check

```bash
#!/bin/bash
# comprehensive-health-check.sh

echo "========================================="
echo "  ORACLE XSTREAM CDC PIPELINE HEALTH"
echo "========================================="
echo ""

echo "1. ORACLE RAC CLUSTER"
echo "---------------------"
srvctl status database -d <database-name> 2>/dev/null || echo "  ⚠️  Oracle RAC not reachable"
echo ""

echo "2. XSTREAM CAPTURE"
echo "------------------"
sqlplus -S sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET FEEDBACK OFF
SET HEADING OFF
SELECT '  Status: ' || status || ' | State: ' || state 
FROM v$xstream_capture 
WHERE capture_name = '<xstream-outbound-name>';
EXIT;
EOF
echo ""

echo "3. DOCKER CONTAINERS"
echo "--------------------"
docker ps --format "table {{.Names}}\t{{.Status}}" --filter "name=kafka|connect|schema|prometheus|grafana|loki|promtail|exporter"
echo ""

echo "4. KAFKA CLUSTER"
echo "----------------"
KAFKA_STATUS=$(docker exec kafka1 kafka-broker-api-versions --bootstrap-server localhost:9092 2>&1)
if [ $? -eq 0 ]; then
  echo "  ✅ Kafka cluster ONLINE"
else
  echo "  ❌ Kafka cluster OFFLINE"
fi
echo ""

echo "5. KAFKA CONNECT"
echo "----------------"
CONNECT_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8083/)
if [ "$CONNECT_STATUS" = "200" ]; then
  echo "  ✅ Kafka Connect ONLINE"
  CONNECTOR_STATE=$(curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq -r '.connector.state')
  echo "  Connector state: $CONNECTOR_STATE"
else
  echo "  ❌ Kafka Connect OFFLINE"
fi
echo ""

echo "6. SCHEMA REGISTRY"
echo "------------------"
SR_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8081/)
if [ "$SR_STATUS" = "200" ]; then
  echo "  ✅ Schema Registry ONLINE"
else
  echo "  ❌ Schema Registry OFFLINE"
fi
echo ""

echo "7. GRAFANA"
echo "----------"
GRAFANA_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/health)
if [ "$GRAFANA_STATUS" = "200" ]; then
  echo "  ✅ Grafana ONLINE (http://<kafka-vm-ip>:3000)"
else
  echo "  ❌ Grafana OFFLINE"
fi
echo ""

echo "8. PROMETHEUS"
echo "-------------"
PROM_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:9090/-/healthy)
if [ "$PROM_STATUS" = "200" ]; then
  echo "  ✅ Prometheus ONLINE"
else
  echo "  ❌ Prometheus OFFLINE"
fi
echo ""

echo "========================================="
echo "  HEALTH CHECK COMPLETE"
echo "========================================="
```

---

### 6.2 Quick Status Commands

**One-liner health check:**
```bash
# Check all critical components
docker ps --filter "name=kafka1|connect" --format "{{.Names}}: {{.Status}}" && \
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq -r '.connector.state' | xargs echo "Connector:" && \
sqlplus -S sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<< "SELECT status FROM v\$xstream_capture WHERE capture_name='<xstream-outbound-name>';"
```

**Check for errors:**
```bash
# Search Docker logs for errors (last 30 minutes)
docker ps --format "{{.Names}}" | xargs -I {} sh -c 'echo "=== {} ==="; docker logs --since 30m {} 2>&1 | grep -i "error\|exception\|failed" | tail -5'
```

---

## 7. TROUBLESHOOTING

### 7.1 Common Issues and Solutions

#### **Issue: Kafka Connect cannot reach Oracle**

**Symptoms:**
```bash
docker logs connect | grep -i "connection refused\|timeout"
```

**Solutions:**
```bash
# 1. Verify Oracle listener is running
lsnrctl status

# 2. Test TNS connectivity from Connect container
docker exec connect ping -c 3 <oracle-hostname>

# 3. Test SQL*Net connection
docker exec connect sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<< "SELECT 'CONNECTED' FROM dual;"

# 4. Check tnsnames.ora inside Connect container
docker exec connect cat $ORACLE_HOME/network/admin/tnsnames.ora

# 5. Restart Connect
docker restart connect
```

---

#### **Issue: XStream capture not sending messages**

**Symptoms:**
```sql
SELECT state FROM v$xstream_outbound_server WHERE server_name='<xstream-outbound-name>';
-- Returns: WAITING FOR CLIENT
```

**Solutions:**
```bash
# 1. Check if Connect is paused
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq .

# 2. Resume connector
curl -X PUT http://localhost:8083/connectors/confluent-xstream-source/resume

# 3. Restart connector
curl -X POST http://localhost:8083/connectors/confluent-xstream-source/restart

# 4. Check Connect logs
docker logs --tail 200 connect | grep -i "xstream\|oracle"
```

---

#### **Issue: High consumer lag**

**Symptoms:**
```bash
# Lag > 1000 messages
docker exec kafka1 kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group xstream-connector-group --describe
```

**Solutions:**
```bash
# 1. Check Kafka Connect task count (scale up)
curl -s http://localhost:8083/connectors/confluent-xstream-source/config | jq '.["tasks.max"]'

# 2. Increase batch size
curl -X PUT http://localhost:8083/connectors/confluent-xstream-source/config \
  -H "Content-Type: application/json" \
  -d '{"batch.size": "10000", "poll.interval.ms": "500"}'

# 3. Check Kafka broker health
docker logs kafka1 | grep -i "error\|exception"

# 4. Restart lagging consumers
# (depends on your consumer application)
```

---

#### **Issue: Docker container won't start**

**Symptoms:**
```bash
docker start kafka1
# Error response from daemon: container cannot be started
```

**Solutions:**
```bash
# 1. Check container logs
docker logs kafka1

# 2. Check Docker daemon status
sudo systemctl status docker

# 3. Remove and recreate container (CAUTION: may lose data)
docker rm kafka1
# Re-run docker run command with original parameters

# 4. Check disk space
df -h
docker system df
```

---

#### **Issue: Grafana dashboard shows no data**

**Symptoms:**
- Grafana dashboards are empty
- "No data" message

**Solutions:**
```bash
# 1. Check Prometheus targets
open http://localhost:9090/targets
# All targets should be "UP"

# 2. Check Prometheus can scrape JMX
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health != "up")'

# 3. Restart Prometheus
docker restart prometheus

# 4. Check Grafana data source
# Grafana UI → Configuration → Data Sources → Prometheus
# Test connection

# 5. Re-import dashboard
# Grafana UI → Dashboards → Import → Use dashboard ID
```

---

### 7.2 Emergency Recovery

**Complete pipeline restart:**
```bash
#!/bin/bash
# emergency-restart.sh

echo "Stopping all components..."

# Stop Kafka Connect (stop CDC consumption)
docker stop connect

# Stop Kafka brokers
docker stop kafka1 kafka2 kafka3

# Stop Schema Registry
docker stop schema-registry

echo "Waiting 10 seconds..."
sleep 10

echo "Starting components..."

# Start in reverse order
docker start schema-registry
sleep 5

docker start kafka1 kafka2 kafka3
sleep 15

docker start connect
sleep 10

echo "Verifying startup..."
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq .

echo "Recovery complete!"
```

---

## 8. METRICS COLLECTION

### 8.1 Collect Test Evidence

**Run comprehensive metrics collection:**
```bash
# Your existing script
./collect-april16-test-metrics.sh

# Output saved to: mtx-30min-test-metrics-YYYYMMDD_HHMMSS/
```

---

### 8.2 Manual Metrics Collection

**Oracle redo generation:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LINESIZE 200
SELECT thread#,
       ROUND(SUM(blocks * block_size) / 1024 / 1024 / 1024, 2) AS gb,
       COUNT(*) AS log_switches,
       ROUND(SUM(blocks * block_size) / 1024 / 1024 / 
         ((MAX(completion_time) - MIN(completion_time)) * 24 * 60 * 60), 2) AS mb_per_sec
FROM v$archived_log
WHERE completion_time >= TO_TIMESTAMP('2026-04-16 00:29:00', 'YYYY-MM-DD HH24:MI:SS')
  AND completion_time <= TO_TIMESTAMP('2026-04-16 00:52:00', 'YYYY-MM-DD HH24:MI:SS')
GROUP BY thread#;
EXIT;
EOF
```

**XStream message count:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SELECT server_name,
       total_messages_sent,
       total_messages_sent / ((SYSDATE - startup_time) * 24 * 60 * 60) AS messages_per_sec
FROM v$xstream_outbound_server
WHERE server_name = '<xstream-outbound-name>';
EXIT;
EOF
```

**Kafka topic metrics:**
```bash
# Message count
docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic <schema-name>.MTX_TRANSACTION_ITEMS \
  --time -1 | awk -F: '{sum += $3} END {printf "Total messages: %'\''d\n", sum}'

# Topic size
docker exec kafka1 kafka-log-dirs --bootstrap-server localhost:9092 \
  --topic-list <schema-name>.MTX_TRANSACTION_ITEMS --describe | grep size
```

---

## 9. REFERENCE

### 9.1 Key URLs

- **Grafana Dashboard:** http://<kafka-vm-ip>:3000
- **Prometheus UI:** http://localhost:9090
- **Kafka Connect REST:** http://localhost:8083
- **Schema Registry:** http://localhost:8081
- **Kafka Exporter Metrics:** http://localhost:9308/metrics
- **Loki API:** http://localhost:3100

---

### 9.2 Important File Locations

**HammerDB:**
- Installation: `/opt/HammerDB-4.10`
- Custom driver: `/opt/HammerDB-4.10/custom/mtx/mtx_driver.tcl`

**Docker:**
- Container logs: `docker logs <container_name>`
- Docker volumes: `docker volume ls`

**Oracle:**
- Alert log: `$ORACLE_BASE/diag/rdbms/rac_xstr/<database-name>1/trace/alert_<database-name>1.log`
- Archive logs: `+FRA/<database-name>/ARCHIVELOG`

---

### 9.3 Quick Reference Commands

```bash
# Oracle status
srvctl status database -d <database-name>

# XStream status
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<< "SELECT status, state FROM v\$xstream_capture WHERE capture_name='<xstream-outbound-name>';"

# All Docker containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Kafka brokers
docker ps --filter "name=kafka"

# Kafka Connect
curl -s http://localhost:8083/connectors | jq .

# Connector status
curl -s http://localhost:8083/connectors/confluent-xstream-source/status | jq '.connector.state'

# Kafka topics
docker exec kafka1 kafka-topics --bootstrap-server localhost:9092 --list

# Grafana
open http://<kafka-vm-ip>:3000

# Full health check
./comprehensive-health-check.sh
```

---

## 10. ADDITIONAL OPERATIONS

### 10.1 Adding New MTX Tables to XStream

**Configure XStream to capture additional tables:**
```bash
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
-- Add new table to capture
BEGIN
  DBMS_XSTREAM_ADM.ADD_TABLE_RULES(
    table_name => '<schema-name>.MTX_NEW_TABLE',
    streams_type => 'CAPTURE',
    streams_name => '<xstream-outbound-name>',
    queue_name => '<xstream-outbound-name>_QUEUE',
    include_dml => TRUE,
    include_ddl => FALSE,
    source_database => NULL
  );
END;
/

-- Enable supplemental logging
ALTER TABLE <schema-name>.MTX_NEW_TABLE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;

EXIT;
EOF
```

**Update Kafka Connect to create new topic:**
```bash
# Connector will auto-create topic for new table
# Verify topic creation:
docker exec kafka1 kafka-topics --bootstrap-server localhost:9092 --list | grep MTX_NEW_TABLE
```

---

### 10.2 Backup and Recovery

**Backup XStream configuration:**
```bash
# Export XStream configuration
sqlplus sys/'<sys-password>'@<database-name>PDB_POC as sysdba <<'EOF'
SET LONG 100000
SET PAGESIZE 0
SELECT DBMS_METADATA.GET_DDL('CAPTURE', '<xstream-outbound-name>') FROM DUAL;
EXIT;
EOF > xstream_capture_ddl.sql
```

**Backup Kafka Connect configuration:**
```bash
# Export connector config
curl -s http://localhost:8083/connectors/confluent-xstream-source/config > connector-config-backup.json
```

---

**Document Version:** 2.0  
**Last Updated:** April 17, 2026  
**Environment:** Docker-based Kafka with Oracle 19c RAC  
**Test Results:** 14,019,801 rows | 1,063 rows/sec | 100% success
