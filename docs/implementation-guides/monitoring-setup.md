# Complete Monitoring Pipeline: JMX → Prometheus → Grafana
## Production-Ready Guide for Java Applications

**Version:** 1.0  
**Environment:** Linux Standalone (Non-Kubernetes)  
**Application:** Java/Spring Boot  
**Created:** April 20, 2026

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 Component Interaction

```
┌─────────────────────────────────────────────────────────────────┐
│                    MONITORING ARCHITECTURE                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: METRICS GENERATION                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Java Application (Spring Boot / Kafka / etc)           │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  JVM Internals                                     │ │   │
│  │  │  - Heap Memory (Eden, Old Gen, Survivor)          │ │   │
│  │  │  - Non-Heap (Metaspace, Code Cache)               │ │   │
│  │  │  - Garbage Collectors (G1GC, ParallelGC, etc)     │ │   │
│  │  │  - Thread Pools                                    │ │   │
│  │  │  - Class Loading                                   │ │   │
│  │  │  - Application MBeans                              │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                           │                              │   │
│  │                           ▼                              │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  JMX (Java Management Extensions)                  │ │   │
│  │  │  - MBean Server (javax.management.MBeanServer)     │ │   │
│  │  │  - Exposed on port: 9999 (RMI)                     │ │   │
│  │  │  - Protocol: JMX RMI                               │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                           │                                      │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: METRICS TRANSLATION (JMX → Prometheus Format)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prometheus JMX Exporter (jmx_prometheus_javaagent.jar) │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  - Attaches to JVM as Java Agent                   │ │   │
│  │  │  - Reads MBeans via JMX locally (no network)       │ │   │
│  │  │  - Translates to Prometheus metrics format         │ │   │
│  │  │  - Exposes HTTP endpoint: :7071/metrics            │ │   │
│  │  │  - Config: jmx_exporter_config.yaml                │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  │  Example Translation:                                   │   │
│  │  JMX: java.lang:type=Memory → HeapMemoryUsage.used     │   │
│  │  Prometheus: jvm_memory_bytes_used{area="heap"}        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           │ HTTP GET /metrics                    │
│                           │ (scrape every 15s)                   │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: METRICS STORAGE & ALERTING                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Prometheus Server (localhost:9090)                     │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Scraper Engine:                                   │ │   │
│  │  │  - Pulls metrics from :7071/metrics every 15s      │ │   │
│  │  │  - Stores in TSDB (Time Series Database)           │ │   │
│  │  │  - Retention: 15 days (configurable)               │ │   │
│  │  │  - Compression: Delta encoding                     │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Alert Manager:                                    │ │   │
│  │  │  - Evaluates alert rules every 15s                 │ │   │
│  │  │  - Checks: jvm_memory_bytes_used / max > 0.8      │ │   │
│  │  │  - Fires alert if true for 2 minutes              │ │   │
│  │  │  - Sends to Alertmanager (deduplication)          │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Query Engine (PromQL):                           │ │   │
│  │  │  - Executes: rate(jvm_gc_pause_seconds_sum[5m])   │ │   │
│  │  │  - Aggregations, functions, math operations        │ │   │
│  │  │  - Serves data to Grafana via HTTP API             │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                           │                                      │
│                           │ HTTP API queries                     │
│                           │ (PromQL)                             │
└───────────────────────────┼──────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4: VISUALIZATION & ADVANCED ALERTING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Grafana (localhost:3000)                               │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Data Source:                                      │ │   │
│  │  │  - Connects to Prometheus API                      │ │   │
│  │  │  - URL: http://localhost:9090                      │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Dashboards:                                       │ │   │
│  │  │  - JVM Memory panel (gauge, graph)                 │ │   │
│  │  │  - GC activity (heatmap)                           │ │   │
│  │  │  - Thread count (time series)                      │ │   │
│  │  │  - Auto-refresh every 5 seconds                    │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Alerting (Grafana-managed):                      │ │   │
│  │  │  - Query: jvm_threads_current > 500                │ │   │
│  │  │  - Condition: above threshold for 3 minutes        │ │   │
│  │  │  - Notification: Slack, Email, PagerDuty          │ │   │
│  │  │  - Advantage: More flexible UI vs Prometheus       │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

           │
           ▼
    ┌──────────────┐
    │ Alertmanager │ ← Routes alerts from Prometheus
    │ (localhost:  │    Deduplication, grouping, silencing
    │  9093)       │    Sends to: Email, Slack, PagerDuty
    └──────────────┘
```

---

### 1.2 Full Data Flow (Step-by-Step)

**Step 1: Metrics Generation (Every JVM Cycle)**
```
JVM Runtime → Collects internal stats (heap, GC, threads)
            → Exposes via MBeans on MBean Server
            → MBeans are Java objects with standardized attributes
            → Example: java.lang:type=Memory has HeapMemoryUsage attribute
```

**Step 2: JMX Exposure (On Demand)**
```
JMX RMI Server → Listens on port 9999 (optional - not needed with agent)
                → Allows remote JMX clients (JConsole, VisualVM)
                → NOT used by Prometheus JMX Exporter (agent mode)
```

**Step 3: Metrics Translation (Continuous)**
```
JMX Exporter (Java Agent) → Runs inside JVM process
                           → Reads MBeans directly via local JMX
                           → Applies YAML config rules
                           → Translates to Prometheus format:
                             jvm_memory_bytes_used{area="heap"} 512000000
                           → Exposes HTTP endpoint :7071/metrics
```

**Step 4: Metrics Scraping (Every 15 seconds)**
```
Prometheus → HTTP GET http://app-server:7071/metrics
           → Parses response (text format)
           → Stores each metric with timestamp in TSDB
           → Metric becomes queryable: jvm_memory_bytes_used{job="myapp"}
```

**Step 5: Alert Evaluation (Every 15 seconds)**
```
Prometheus → Executes alert rules from alert.rules.yml
           → Example: jvm_memory_bytes_used / jvm_memory_bytes_max > 0.8
           → If true for 2 minutes → FIRING state
           → Sends alert to Alertmanager
```

**Step 6: Alert Routing (Immediate)**
```
Alertmanager → Receives alert from Prometheus
             → Deduplicates (if multiple Prometheus instances)
             → Groups by labels (e.g., all app=myapp alerts together)
             → Routes based on config (severity=critical → PagerDuty)
             → Sends notification (email, Slack, webhook)
```

**Step 7: Visualization (On User Request)**
```
User opens Grafana Dashboard → Grafana queries Prometheus API
                             → PromQL: rate(jvm_gc_pause_seconds_sum[5m])
                             → Prometheus returns time series data
                             → Grafana renders graph/gauge/table
                             → Auto-refreshes every 5 seconds
```

**Step 8: Grafana Alerting (Every 1 minute - configurable)**
```
Grafana → Executes alert query independently
        → Example: avg(jvm_threads_current) > 500
        → If condition true for 3 evaluations → Alert fires
        → Sends via Grafana contact points (parallel to Prometheus alerts)
```

---

### 1.3 Key Differences: Prometheus Alerts vs Grafana Alerts

| Aspect | Prometheus Alerts | Grafana Alerts |
|--------|------------------|----------------|
| **Where Defined** | alert.rules.yml file | Grafana UI or provisioning |
| **Evaluation** | Prometheus server (every evaluation_interval) | Grafana server (configurable per alert) |
| **Query Language** | PromQL only | PromQL, LogQL, SQL, etc. |
| **Routing** | Via Alertmanager (separate component) | Built-in Grafana Alerting |
| **Use Case** | Infrastructure-level, critical alerts | Dashboard-driven, exploratory alerts |
| **Dependency** | Works without Grafana | Requires Grafana running |
| **Best For** | Production-critical alerts (SLA breaches) | Dev/staging, multi-datasource alerts |

**Recommendation:** Use BOTH
- Prometheus alerts for critical SLIs (latency, error rate, saturation)
- Grafana alerts for dashboard-specific thresholds and exploratory monitoring

---

## 2. JMX SETUP

### 2.1 What is JMX?

**Java Management Extensions (JMX)** is a Java technology that provides:
- **Instrumentation:** Expose application metrics as MBeans (Managed Beans)
- **Management:** Query and modify MBean attributes at runtime
- **Monitoring:** Track JVM internals (memory, GC, threads, class loading)

**Why JMX for Monitoring?**
1. **Built-in JVM Metrics:** No code changes needed for heap, GC, threads
2. **Standardized Interface:** All JVMs expose same core MBeans
3. **Real-Time:** Metrics reflect current JVM state
4. **Extensible:** Create custom MBeans for application-specific metrics

**Common MBeans:**
```
java.lang:type=Memory                    → Heap/non-heap usage
java.lang:type=GarbageCollector,name=*   → GC stats (collections, pause time)
java.lang:type=Threading                 → Thread count, deadlocks
java.lang:type=ClassLoading              → Loaded/unloaded classes
java.lang:type=Runtime                   → Uptime, JVM version
java.lang:type=OperatingSystem           → CPU, file descriptors
```

---

### 2.2 Enabling JMX in Java Application

**Option 1: Remote JMX (NOT recommended for Prometheus - use for JConsole/VisualVM)**

```bash
# JVM startup flags for remote JMX access
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.rmi.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Djava.rmi.server.hostname=192.168.1.100 \
     -jar myapp.jar
```

**Explanation:**
- `-Dcom.sun.management.jmxremote` → Enable JMX agent
- `-Dcom.sun.management.jmxremote.port=9999` → JMX RMI registry port
- `-Dcom.sun.management.jmxremote.rmi.port=9999` → RMI server port (same as registry to avoid firewall issues)
- `-Dcom.sun.management.jmxremote.authenticate=false` → No username/password (NOT production-safe)
- `-Dcom.sun.management.jmxremote.ssl=false` → No SSL encryption (NOT production-safe)
- `-Djava.rmi.server.hostname=192.168.1.100` → Advertised hostname for RMI callbacks

**Verification:**
```bash
# From another machine with JDK
jconsole 192.168.1.100:9999

# OR use command-line JMX client
echo "get -b java.lang:type=Memory HeapMemoryUsage" | java -jar jmxterm.jar -l 192.168.1.100:9999
```

---

**Option 2: JMX with Authentication (Production)**

**Create password file:**
```bash
# Create password file
cat > /opt/myapp/jmxremote.password <<EOF
monitorRole  password123
controlRole  password456
EOF

chmod 600 /opt/myapp/jmxremote.password
chown myapp:myapp /opt/myapp/jmxremote.password
```

**Create access file:**
```bash
cat > /opt/myapp/jmxremote.access <<EOF
monitorRole  readonly
controlRole  readwrite
EOF

chmod 644 /opt/myapp/jmxremote.access
```

**Start with authentication:**
```bash
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.rmi.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=true \
     -Dcom.sun.management.jmxremote.password.file=/opt/myapp/jmxremote.password \
     -Dcom.sun.management.jmxremote.access.file=/opt/myapp/jmxremote.access \
     -Dcom.sun.management.jmxremote.ssl=false \
     -Djava.rmi.server.hostname=192.168.1.100 \
     -jar myapp.jar
```

**Connect with authentication:**
```bash
jconsole -J-Djmx.remote.credentials=monitorRole,password123 192.168.1.100:9999
```

---

**Option 3: JMX with SSL (Maximum Security)**

**Generate SSL keystore:**
```bash
# Generate self-signed certificate
keytool -genkeypair -alias jmxssl \
        -keyalg RSA -keysize 2048 \
        -validity 365 \
        -keystore /opt/myapp/jmx-keystore.jks \
        -storepass changeit \
        -dname "CN=myapp.example.com, OU=Monitoring, O=MyCompany, L=City, ST=State, C=US"

# Export certificate
keytool -exportcert -alias jmxssl \
        -keystore /opt/myapp/jmx-keystore.jks \
        -storepass changeit \
        -file /opt/myapp/jmx-cert.pem
```

**Start with SSL:**
```bash
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9999 \
     -Dcom.sun.management.jmxremote.rmi.port=9999 \
     -Dcom.sun.management.jmxremote.authenticate=true \
     -Dcom.sun.management.jmxremote.password.file=/opt/myapp/jmxremote.password \
     -Dcom.sun.management.jmxremote.access.file=/opt/myapp/jmxremote.access \
     -Dcom.sun.management.jmxremote.ssl=true \
     -Dcom.sun.management.jmxremote.ssl.need.client.auth=false \
     -Djavax.net.ssl.keyStore=/opt/myapp/jmx-keystore.jks \
     -Djavax.net.ssl.keyStorePassword=changeit \
     -Djava.rmi.server.hostname=192.168.1.100 \
     -jar myapp.jar
```

---

**Option 4: Local JMX Only (RECOMMENDED for Prometheus JMX Exporter)**

When using Prometheus JMX Exporter as a Java Agent, **you don't need remote JMX at all**:

```bash
# No JMX flags needed - exporter reads MBeans locally
java -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/jmx_exporter/config.yaml \
     -jar myapp.jar
```

**Why this is better:**
- ✅ No network ports for JMX (only HTTP :7071)
- ✅ No authentication complexity
- ✅ JMX Exporter reads MBeans via local JVM interface
- ✅ Simpler firewall rules
- ✅ Lower attack surface

---

### 2.3 Security Considerations

**1. Remote JMX Risks:**
- **Unauthenticated access** → Anyone can read heap dumps, trigger GC, modify MBeans
- **No SSL** → Credentials sent in plaintext
- **RMI vulnerabilities** → Known exploits (deserialization attacks)

**2. Production Best Practices:**
```bash
# NEVER use in production:
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

# ALWAYS use:
- Authentication (password file)
- SSL encryption
- Firewall rules (restrict JMX port to monitoring servers only)
- Or better: Use JMX Exporter agent (no remote JMX needed)
```

**3. Firewall Configuration:**
```bash
# Allow JMX only from Prometheus server
iptables -A INPUT -p tcp --dport 9999 -s 192.168.1.50 -j ACCEPT
iptables -A INPUT -p tcp --dport 9999 -j DROP

# Allow JMX Exporter HTTP (safe - read-only)
iptables -A INPUT -p tcp --dport 7071 -s 192.168.1.50 -j ACCEPT
iptables -A INPUT -p tcp --dport 7071 -j DROP
```

**4. Network Segmentation:**
- Place monitoring infrastructure in separate VLAN
- Use VPN for accessing JMX remotely
- Never expose JMX ports to public internet

---

## 3. JMX EXPORTER SETUP

### 3.1 What is Prometheus JMX Exporter?

**Prometheus JMX Exporter** is a collector that:
- Scrapes JMX MBeans from a Java application
- Translates JMX metrics to Prometheus format
- Exposes metrics via HTTP endpoint (/metrics)
- Written in Java, maintained by Prometheus community

**GitHub:** https://github.com/prometheus/jmx_exporter  
**Latest Version:** 0.20.0 (as of 2026)

**Why use it?**
- Prometheus cannot scrape JMX directly (different protocols)
- JMX uses RMI (binary, complex), Prometheus uses HTTP (text, simple)
- Exporter bridges the gap

---

### 3.2 Java Agent vs Standalone Mode

| Aspect | Java Agent Mode | Standalone Mode |
|--------|----------------|-----------------|
| **How it runs** | Inside target JVM (as -javaagent) | Separate JVM process |
| **JMX connection** | Local (no network) | Remote (via RMI port) |
| **Port usage** | HTTP only (:7071) | HTTP (:7071) + JMX (:9999) |
| **Performance** | Lower overhead | Higher overhead (RMI) |
| **Security** | No JMX auth needed | Requires JMX auth/SSL |
| **Use case** | **RECOMMENDED** for own apps | For 3rd-party apps you can't modify |
| **Example** | Your Spring Boot app | External Kafka broker |

**Recommendation:** Always use **Java Agent mode** for applications you control.

---

### 3.3 Step-by-Step Setup (Java Agent Mode)

**Step 1: Download JMX Exporter**

```bash
# Create directory
sudo mkdir -p /opt/jmx_exporter
cd /opt/jmx_exporter

# Download latest version
sudo curl -L https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar \
     -o jmx_prometheus_javaagent-0.20.0.jar

# Verify download
ls -lh jmx_prometheus_javaagent-0.20.0.jar
# Expected: ~600 KB file

# Set permissions
sudo chmod 644 jmx_prometheus_javaagent-0.20.0.jar
```

---

**Step 2: Create Configuration File**

```bash
sudo vim /opt/jmx_exporter/config.yaml
```

**config.yaml content:**

```yaml
---
# Prometheus JMX Exporter Configuration
# Purpose: Export JVM metrics (memory, GC, threads) to Prometheus

# Global settings
startDelaySeconds: 0
ssl: false
lowercaseOutputName: true
lowercaseOutputLabelNames: true

# WhiteList: Only export these MBeans (improves performance)
whitelistObjectNames:
  - "java.lang:type=Memory"
  - "java.lang:type=GarbageCollector,name=*"
  - "java.lang:type=Threading"
  - "java.lang:type=ClassLoading"
  - "java.lang:type=OperatingSystem"
  - "java.lang:type=Runtime"
  - "java.nio:type=BufferPool,name=*"

# Rules: Transform JMX attributes to Prometheus metrics
rules:
  # ============================================
  # 1. MEMORY METRICS (Heap and Non-Heap)
  # ============================================
  - pattern: 'java.lang<type=Memory><HeapMemoryUsage>(\w+)'
    name: jvm_memory_bytes_$1
    type: GAUGE
    labels:
      area: "heap"
    help: "JVM heap memory usage in bytes"
    
  - pattern: 'java.lang<type=Memory><NonHeapMemoryUsage>(\w+)'
    name: jvm_memory_bytes_$1
    type: GAUGE
    labels:
      area: "nonheap"
    help: "JVM non-heap memory usage in bytes"

  # ============================================
  # 2. GARBAGE COLLECTION METRICS
  # ============================================
  - pattern: 'java.lang<type=GarbageCollector, name=([\w\s]+)><>CollectionCount'
    name: jvm_gc_collection_count_total
    type: COUNTER
    labels:
      gc: "$1"
    help: "Total number of GC collections"

  - pattern: 'java.lang<type=GarbageCollector, name=([\w\s]+)><>CollectionTime'
    name: jvm_gc_collection_time_ms_total
    type: COUNTER
    labels:
      gc: "$1"
    help: "Total GC collection time in milliseconds"

  # ============================================
  # 3. THREAD METRICS
  # ============================================
  - pattern: 'java.lang<type=Threading><>ThreadCount'
    name: jvm_threads_current
    type: GAUGE
    help: "Current number of live threads"

  - pattern: 'java.lang<type=Threading><>DaemonThreadCount'
    name: jvm_threads_daemon
    type: GAUGE
    help: "Current number of daemon threads"

  - pattern: 'java.lang<type=Threading><>PeakThreadCount'
    name: jvm_threads_peak
    type: GAUGE
    help: "Peak number of live threads since JVM start"

  - pattern: 'java.lang<type=Threading><>TotalStartedThreadCount'
    name: jvm_threads_started_total
    type: COUNTER
    help: "Total number of threads started since JVM start"

  # ============================================
  # 4. CLASS LOADING METRICS
  # ============================================
  - pattern: 'java.lang<type=ClassLoading><>LoadedClassCount'
    name: jvm_classes_currently_loaded
    type: GAUGE
    help: "Number of classes currently loaded"

  - pattern: 'java.lang<type=ClassLoading><>TotalLoadedClassCount'
    name: jvm_classes_loaded_total
    type: COUNTER
    help: "Total number of classes loaded since JVM start"

  - pattern: 'java.lang<type=ClassLoading><>UnloadedClassCount'
    name: jvm_classes_unloaded_total
    type: COUNTER
    help: "Total number of classes unloaded since JVM start"

  # ============================================
  # 5. RUNTIME METRICS
  # ============================================
  - pattern: 'java.lang<type=Runtime><>Uptime'
    name: jvm_runtime_uptime_ms
    type: COUNTER
    help: "JVM uptime in milliseconds"

  # ============================================
  # 6. OPERATING SYSTEM METRICS
  # ============================================
  - pattern: 'java.lang<type=OperatingSystem><>ProcessCpuLoad'
    name: jvm_process_cpu_load
    type: GAUGE
    help: "Recent CPU usage for the JVM process (0.0 to 1.0)"

  - pattern: 'java.lang<type=OperatingSystem><>SystemCpuLoad'
    name: jvm_system_cpu_load
    type: GAUGE
    help: "Recent CPU usage for the whole system (0.0 to 1.0)"

  - pattern: 'java.lang<type=OperatingSystem><>OpenFileDescriptorCount'
    name: jvm_os_open_file_descriptors
    type: GAUGE
    help: "Number of open file descriptors"

  - pattern: 'java.lang<type=OperatingSystem><>MaxFileDescriptorCount'
    name: jvm_os_max_file_descriptors
    type: GAUGE
    help: "Maximum number of file descriptors"

  # ============================================
  # 7. BUFFER POOL METRICS (NIO)
  # ============================================
  - pattern: 'java.nio<type=BufferPool, name=([\w\s]+)><>Count'
    name: jvm_buffer_pool_count
    type: GAUGE
    labels:
      pool: "$1"
    help: "Number of buffers in the pool"

  - pattern: 'java.nio<type=BufferPool, name=([\w\s]+)><>MemoryUsed'
    name: jvm_buffer_pool_bytes_used
    type: GAUGE
    labels:
      pool: "$1"
    help: "Bytes used by buffer pool"

  - pattern: 'java.nio<type=BufferPool, name=([\w\s]+)><>TotalCapacity'
    name: jvm_buffer_pool_bytes_capacity
    type: GAUGE
    labels:
      pool: "$1"
    help: "Total capacity of buffer pool in bytes"
```

**Save and set permissions:**
```bash
sudo chmod 644 /opt/jmx_exporter/config.yaml
```

---

**Step 3: Modify Application Startup Script**

**Example 1: Standalone JAR**

```bash
# Before (no monitoring):
java -Xmx2g -Xms2g -jar /opt/myapp/myapp.jar

# After (with JMX Exporter):
java -Xmx2g -Xms2g \
     -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/jmx_exporter/config.yaml \
     -jar /opt/myapp/myapp.jar
```

**Explanation:**
- `-javaagent:<path-to-jar>=<port>:<config-file>`
- `7071` → HTTP port where metrics are exposed
- `/opt/jmx_exporter/config.yaml` → Configuration file path

---

**Example 2: Systemd Service**

```bash
sudo vim /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Java Application
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp

# Environment variables
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
Environment="JAVA_OPTS=-Xmx2g -Xms2g -XX:+UseG1GC"
Environment="JMX_EXPORTER_JAR=/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar"
Environment="JMX_EXPORTER_CONFIG=/opt/jmx_exporter/config.yaml"
Environment="JMX_EXPORTER_PORT=7071"

# Startup command
ExecStart=/bin/bash -c '${JAVA_HOME}/bin/java ${JAVA_OPTS} \
  -javaagent:${JMX_EXPORTER_JAR}=${JMX_EXPORTER_PORT}:${JMX_EXPORTER_CONFIG} \
  -jar /opt/myapp/myapp.jar'

# Restart policy
Restart=on-failure
RestartSec=10

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

**Reload and restart:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
sudo systemctl status myapp
```

---

**Example 3: Kafka (Common Use Case)**

If you're monitoring Kafka Connect (from your earlier setup):

```bash
# Edit Kafka Connect startup script
vim /opt/kafka/bin/connect-distributed.sh

# Find KAFKA_OPTS line and add:
export KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/jmx_exporter/kafka-connect-config.yaml"

# Then start Kafka Connect
./bin/connect-distributed.sh config/connect-distributed.properties
```

---

**Step 4: Verify JMX Exporter is Working**

```bash
# Check if port is listening
ss -tlnp | grep 7071
# Expected: LISTEN on 0.0.0.0:7071

# Fetch metrics via HTTP
curl http://localhost:7071/metrics

# Expected output (sample):
# HELP jvm_memory_bytes_used JVM heap memory usage in bytes
# TYPE jvm_memory_bytes_used gauge
# jvm_memory_bytes_used{area="heap"} 5.24288E8
# jvm_memory_bytes_used{area="nonheap"} 1.2345678E8
# 
# HELP jvm_gc_collection_count_total Total number of GC collections
# TYPE jvm_gc_collection_count_total counter
# jvm_gc_collection_count_total{gc="G1 Young Generation"} 15.0
# jvm_gc_collection_count_total{gc="G1 Old Generation"} 0.0
#
# HELP jvm_threads_current Current number of live threads
# TYPE jvm_threads_current gauge
# jvm_threads_current 42.0
```

**If you see metrics, JMX Exporter is working! ✅**

---

**Step 5: Test from Remote Machine**

```bash
# From Prometheus server
curl http://<app-server-ip>:7071/metrics | grep jvm_memory

# Expected: Should see metrics (if firewall allows)
```

---

### 3.4 Standalone Mode Setup (For Completeness)

**When to use:** Monitoring 3rd-party Java apps where you can't add `-javaagent` flag.

**Step 1: Download standalone JAR**
```bash
cd /opt/jmx_exporter
sudo curl -L https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_httpserver/0.20.0/jmx_prometheus_httpserver-0.20.0.jar \
     -o jmx_prometheus_httpserver-0.20.0.jar
```

**Step 2: Create config (same as agent mode)**
```yaml
# /opt/jmx_exporter/standalone-config.yaml
hostPort: 192.168.1.100:9999  # JMX RMI endpoint
ssl: false
lowercaseOutputName: true

# ... same rules as agent mode ...
```

**Step 3: Run standalone exporter**
```bash
java -jar /opt/jmx_exporter/jmx_prometheus_httpserver-0.20.0.jar \
     7071 \
     /opt/jmx_exporter/standalone-config.yaml
```

**Now metrics available at:** `http://localhost:7071/metrics`

**Note:** Target application must have remote JMX enabled (see section 2.2).

---

## 4. PROMETHEUS SETUP

### 4.1 Installation (Linux)

**Method 1: Binary Installation (Recommended)**

```bash
# Create Prometheus user
sudo useradd --no-create-home --shell /bin/false prometheus

# Download Prometheus
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.50.1/prometheus-2.50.1.linux-amd64.tar.gz

# Extract
tar -xzf prometheus-2.50.1.linux-amd64.tar.gz

# Move binaries
sudo cp prometheus-2.50.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.50.1.linux-amd64/promtool /usr/local/bin/

# Set ownership
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

# Create directories
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus

# Move console files
sudo cp -r prometheus-2.50.1.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-2.50.1.linux-amd64/console_libraries /etc/prometheus/

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

# Cleanup
rm -rf prometheus-2.50.1.linux-amd64*

# Verify installation
prometheus --version
# Expected: prometheus, version 2.50.1
```

---

**Method 2: Package Manager (Ubuntu/Debian)**

```bash
# Add Prometheus APT repository
sudo apt-get update
sudo apt-get install -y software-properties-common
sudo add-apt-repository ppa:prometheus-team/prometheus

# Install
sudo apt-get update
sudo apt-get install -y prometheus

# Prometheus installed at:
# Binary: /usr/bin/prometheus
# Config: /etc/prometheus/prometheus.yml
# Data: /var/lib/prometheus
```

---

### 4.2 Directory Structure

```
/etc/prometheus/
├── prometheus.yml          # Main configuration file
├── alert.rules.yml         # Alert rules (we'll create this)
├── consoles/               # Built-in console templates
└── console_libraries/      # Console libraries

/var/lib/prometheus/
└── data/                   # Time-series database (TSDB)
    ├── chunks_head/        # Recent data (in memory)
    ├── wal/                # Write-ahead log
    └── 01234567890ABCDEF/  # Block directories (2-hour chunks)

/usr/local/bin/
├── prometheus              # Main server binary
└── promtool                # Validation tool
```

---

### 4.3 Prometheus Configuration (prometheus.yml)

```bash
sudo vim /etc/prometheus/prometheus.yml
```

```yaml
# Prometheus Configuration
# Purpose: Scrape JMX metrics from Java applications

# Global settings
global:
  # How frequently to scrape targets
  scrape_interval: 15s
  
  # How frequently to evaluate alert rules
  evaluation_interval: 15s
  
  # Timeout for scraping
  scrape_timeout: 10s
  
  # External labels (useful for federation)
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

# Alerting configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093'  # Alertmanager address

# Alert rule files
rule_files:
  - '/etc/prometheus/alert.rules.yml'

# Scrape configurations
scrape_configs:
  # ============================================
  # Job 1: Prometheus itself
  # ============================================
  - job_name: 'prometheus'
    static_configs:
      - targets:
          - 'localhost:9090'
        labels:
          app: 'prometheus'
          env: 'production'

  # ============================================
  # Job 2: Java Application (Spring Boot / Custom App)
  # ============================================
  - job_name: 'myapp'
    
    # Scrape interval override (optional)
    scrape_interval: 15s
    scrape_timeout: 10s
    
    # Metrics path (default is /metrics)
    metrics_path: '/metrics'
    
    # Static targets (list of servers)
    static_configs:
      - targets:
          - '192.168.1.100:7071'  # App server 1
          - '192.168.1.101:7071'  # App server 2
        labels:
          app: 'myapp'
          env: 'production'
          tier: 'backend'
    
    # Metric relabeling (optional - modify labels after scraping)
    metric_relabel_configs:
      # Drop metrics you don't need (save storage)
      - source_labels: [__name__]
        regex: 'jvm_buffer_pool_.*'
        action: drop

  # ============================================
  # Job 3: Kafka Connect (JMX Exporter)
  # ============================================
  - job_name: 'kafka-connect'
    static_configs:
      - targets:
          - '192.168.1.200:7071'
        labels:
          app: 'kafka-connect'
          env: 'production'

  # ============================================
  # Job 4: Multiple instances with file-based discovery
  # ============================================
  - job_name: 'microservices'
    
    # File-based service discovery (auto-reload on file change)
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/microservices-*.json'
        refresh_interval: 30s
    
    # Relabel to extract instance name from file
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
      - source_labels: [__meta_filepath]
        target_label: source_file
```

---

**Example: File-based Service Discovery**

```bash
# Create targets directory
sudo mkdir -p /etc/prometheus/targets

# Create target file
sudo vim /etc/prometheus/targets/microservices-prod.json
```

```json
[
  {
    "targets": ["app1.example.com:7071", "app2.example.com:7071"],
    "labels": {
      "job": "order-service",
      "env": "production"
    }
  },
  {
    "targets": ["app3.example.com:7071"],
    "labels": {
      "job": "payment-service",
      "env": "production"
    }
  }
]
```

**Advantage:** Add/remove targets without restarting Prometheus (file auto-reloads).

---

### 4.4 Configuration Concepts Explained

**scrape_interval (default: 1m)**
- How often Prometheus pulls metrics from targets
- Trade-off:
  - **Short interval (15s):** Higher resolution, catches spikes, more storage/CPU
  - **Long interval (60s):** Less load, misses short-lived spikes
- **Recommendation:** 15s for critical apps, 30-60s for less critical

**evaluation_interval (default: 1m)**
- How often Prometheus evaluates alert rules
- Should be ≤ scrape_interval (otherwise alerts lag behind data)
- **Recommendation:** Same as scrape_interval (15s)

**scrape_timeout (default: 10s)**
- Max time to wait for target response
- Should be < scrape_interval (to avoid overlap)
- **Recommendation:** 10s for 15s interval, 30s for 60s interval

---

### 4.5 Validate Configuration

```bash
# Check syntax
promtool check config /etc/prometheus/prometheus.yml

# Expected output:
# Checking /etc/prometheus/prometheus.yml
#   SUCCESS: 2 rule files found
#   SUCCESS: /etc/prometheus/prometheus.yml is valid prometheus config file syntax

# If errors:
# Error parsing config file: yaml: line 45: could not find expected ':'
```

---

### 4.6 Create Systemd Service

```bash
sudo vim /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus Time Series Database
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus

# Prometheus command
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle \
  --log.level=info

# Reload config without restart
ExecReload=/bin/kill -HUP $MAINPID

# Restart policy
Restart=on-failure
RestartSec=5s

# File limits
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**Parameter explanation:**
- `--storage.tsdb.retention.time=15d` → Keep metrics for 15 days
- `--storage.tsdb.retention.size=50GB` → Max disk space for metrics
- `--web.enable-lifecycle` → Allow config reload via HTTP POST /-/reload
- `--log.level=info` → Logging verbosity (debug, info, warn, error)

---

### 4.7 Start Prometheus

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable on boot
sudo systemctl enable prometheus

# Start service
sudo systemctl start prometheus

# Check status
sudo systemctl status prometheus

# Expected output:
# ● prometheus.service - Prometheus Time Series Database
#    Loaded: loaded (/etc/systemd/system/prometheus.service; enabled)
#    Active: active (running) since Sun 2026-04-20 10:00:00 UTC; 5s ago
#  Main PID: 12345 (prometheus)
#     Tasks: 12 (limit: 4915)
#    Memory: 150.0M
#       CPU: 2.5s

# View logs
sudo journalctl -u prometheus -f

# Look for:
# level=info ts=2026-04-20T10:00:00.123Z caller=main.go:500 msg="Server is ready to receive web requests."
```

---

### 4.8 Access Prometheus Web UI

```bash
# Open in browser
http://<prometheus-server-ip>:9090

# Or locally
http://localhost:9090
```

**Web UI Features:**
1. **Graph tab:** Run PromQL queries, visualize time series
2. **Alerts tab:** View active alerts and their state
3. **Status → Targets:** Check scrape targets (UP/DOWN)
4. **Status → Configuration:** View loaded config
5. **Status → Rules:** View alert rules
6. **Status → Service Discovery:** View discovered targets

---

### 4.9 Verify Targets are Being Scraped

```bash
# Via Web UI:
# http://localhost:9090/targets
# Should show your jobs (myapp, kafka-connect, etc.)
# State column should show "UP" (green)

# Via API:
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health}'

# Expected output:
# {
#   "job": "myapp",
#   "instance": "192.168.1.100:7071",
#   "health": "up"
# }
```

**If target shows DOWN:**
- Check if JMX Exporter port is accessible: `curl http://192.168.1.100:7071/metrics`
- Check firewall rules
- Check Prometheus logs: `journalctl -u prometheus | grep "error.*scrape"`

---

### 4.10 Test PromQL Queries

```bash
# Via Web UI:
# http://localhost:9090/graph
# Enter query in "Expression" field, click "Execute"

# Via API:
curl -G http://localhost:9090/api/v1/query \
     --data-urlencode 'query=jvm_memory_bytes_used{area="heap"}' | jq .

# Expected response:
# {
#   "status": "success",
#   "data": {
#     "resultType": "vector",
#     "result": [
#       {
#         "metric": {
#           "__name__": "jvm_memory_bytes_used",
#           "area": "heap",
#           "instance": "192.168.1.100:7071",
#           "job": "myapp"
#         },
#         "value": [1713607200, "524288000"]
#       }
#     ]
#   }
# }
```

**Common test queries:**
```promql
# Current heap usage
jvm_memory_bytes_used{area="heap"}

# Heap usage percentage
(jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) * 100

# GC rate (collections per second)
rate(jvm_gc_collection_count_total[5m])

# Thread count
jvm_threads_current
```

---

## 5. GRAFANA SETUP

### 5.1 Installation (Linux)

**Method 1: APT Repository (Ubuntu/Debian)**

```bash
# Install dependencies
sudo apt-get install -y apt-transport-https software-properties-common wget

# Add Grafana GPG key
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Add repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Update and install
sudo apt-get update
sudo apt-get install -y grafana

# Start service
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# Check status
sudo systemctl status grafana-server
```

---

**Method 2: Binary Installation (Any Linux)**

```bash
# Download Grafana
cd /tmp
wget https://dl.grafana.com/oss/release/grafana-10.4.1.linux-amd64.tar.gz

# Extract
tar -zxvf grafana-10.4.1.linux-amd64.tar.gz

# Move to /opt
sudo mv grafana-10.4.1 /opt/grafana

# Create user
sudo useradd --no-create-home --shell /bin/false grafana

# Set permissions
sudo chown -R grafana:grafana /opt/grafana

# Create systemd service
sudo vim /etc/systemd/system/grafana.service
```

```ini
[Unit]
Description=Grafana
Documentation=https://grafana.com/docs/
After=network.target

[Service]
Type=simple
User=grafana
Group=grafana
WorkingDirectory=/opt/grafana
ExecStart=/opt/grafana/bin/grafana-server \
  --config=/opt/grafana/conf/defaults.ini \
  --homepath=/opt/grafana
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
# Start Grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana
sudo systemctl start grafana
```

---

### 5.2 Initial Login and Setup

**Access Grafana:**
```
URL: http://<grafana-server-ip>:3000
Default username: admin
Default password: admin
```

**First login:**
1. Browser opens login page
2. Enter `admin` / `admin`
3. Prompted to change password → Set new password
4. Welcome page appears

---

### 5.3 Add Prometheus as Data Source

**Via Web UI:**

1. **Navigate:** Left sidebar → ⚙️ Configuration → Data Sources
2. **Click:** "Add data source"
3. **Select:** "Prometheus"
4. **Configure:**
   - Name: `Prometheus`
   - URL: `http://localhost:9090` (or `http://<prometheus-ip>:9090`)
   - Access: `Server` (Grafana backend makes requests)
   - Scrape interval: `15s` (matches Prometheus config)
   - HTTP Method: `POST` (recommended for large queries)
5. **Scroll down → Click:** "Save & Test"
6. **Expected:** ✅ "Data source is working"

---

**Via Configuration File (Provisioning)**

```bash
# Create provisioning directory
sudo mkdir -p /etc/grafana/provisioning/datasources

# Create datasource config
sudo vim /etc/grafana/provisioning/datasources/prometheus.yaml
```

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: true
    jsonData:
      httpMethod: POST
      timeInterval: 15s
```

```bash
# Restart Grafana
sudo systemctl restart grafana-server
```

**Verify:**
- Go to Configuration → Data Sources
- "Prometheus" should appear
- Click "Test" → Should show success

---

### 5.4 Test Connection

**Method 1: Explore Tab**
1. Left sidebar → 🧭 Explore
2. Data source dropdown → Select "Prometheus"
3. Enter query: `up`
4. Click "Run query"
5. Should show all Prometheus targets with `value=1` (UP)

**Method 2: Dashboard Panel**
1. Create new dashboard → Add panel
2. Query: `jvm_memory_bytes_used{area="heap"}`
3. Click "Run query"
4. Should see time series graph

---

## 6. GRAFANA DASHBOARDS (DETAILED)

### 6.1 Creating Your First Dashboard

**Step 1: Create Dashboard**
1. Left sidebar → ➕ Create → Dashboard
2. Click "Add visualization"
3. Select data source: "Prometheus"

**Step 2: Add JVM Memory Panel**

**Panel Title:** "JVM Heap Memory Usage"

**Query:**
```promql
jvm_memory_bytes_used{area="heap", job="myapp"}
```

**Visualization:**
- Type: **Time series** (line graph)
- Unit: **bytes** (in Display settings)
- Thresholds:
  - Yellow: 1.5 GB (warning)
  - Red: 1.8 GB (critical)

**Panel Settings:**
- Title: "Heap Memory Usage"
- Description: "Current heap memory usage (used bytes)"
- Legend: `{{instance}}` (show instance name)

**Click:** Apply (top right)

---

### 6.2 Complete Dashboard: JVM Monitoring

I'll provide a full dashboard JSON in the next section. First, let's create each panel type.

---

**PANEL 1: Heap Usage Percentage (Gauge)**

**PromQL:**
```promql
(jvm_memory_bytes_used{area="heap", job="myapp"} / jvm_memory_bytes_max{area="heap", job="myapp"}) * 100
```

**Visualization:** Gauge  
**Unit:** Percent (0-100)  
**Thresholds:**
- Green: 0-70%
- Yellow: 70-85%
- Red: 85-100%

**Use case:** Quick visual indicator of heap pressure

---

**PANEL 2: Heap Memory Over Time (Time Series Graph)**

**PromQL:**
```promql
# Used memory
jvm_memory_bytes_used{area="heap", job="myapp"}

# Max memory (for reference line)
jvm_memory_bytes_max{area="heap", job="myapp"}

# Committed memory
jvm_memory_bytes_committed{area="heap", job="myapp"}
```

**Visualization:** Time series (multi-line)  
**Unit:** bytes (SI)  
**Legend:**
- `Used: {{instance}}`
- `Max: {{instance}}`
- `Committed: {{instance}}`

**Colors:**
- Used: Blue
- Max: Red (dashed line)
- Committed: Green

---

**PANEL 3: GC Collections per Second (Time Series)**

**PromQL:**
```promql
# Rate of GC collections
rate(jvm_gc_collection_count_total{job="myapp"}[5m])
```

**Visualization:** Time series (stacked area)  
**Unit:** ops/sec  
**Legend:** `{{gc}} on {{instance}}`

**Explanation:**
- `rate()` calculates per-second rate over 5-minute window
- Shows how frequently GC runs
- High rate → Heap pressure or memory leak

---

**PANEL 4: GC Pause Time Percentage (Stat)**

**PromQL:**
```promql
# Time spent in GC as percentage of total time
(rate(jvm_gc_collection_time_ms_total{job="myapp"}[5m]) / 1000) * 100
```

**Visualization:** Stat (big number)  
**Unit:** Percent (0-100)  
**Thresholds:**
- Green: 0-2% (healthy)
- Yellow: 2-5% (concerning)
- Red: >5% (critical - app spending too much time in GC)

**Explanation:**
- Dividing by 1000 converts milliseconds to seconds
- *100 converts to percentage
- Target: <1% is excellent, <2% is good, >5% needs investigation

---

**PANEL 5: Thread Count (Time Series)**

**PromQL:**
```promql
# Current threads
jvm_threads_current{job="myapp"}

# Daemon threads
jvm_threads_daemon{job="myapp"}

# Peak threads
jvm_threads_peak{job="myapp"}
```

**Visualization:** Time series  
**Unit:** none (count)  
**Legend:**
- `Current: {{instance}}`
- `Daemon: {{instance}}`
- `Peak: {{instance}}`

---

**PANEL 6: CPU Usage (Gauge)**

**PromQL:**
```promql
# Process CPU usage (0.0 to 1.0 scale)
jvm_process_cpu_load{job="myapp"} * 100
```

**Visualization:** Gauge  
**Unit:** Percent (0-100)  
**Thresholds:**
- Green: 0-60%
- Yellow: 60-80%
- Red: 80-100%

---

**PANEL 7: Class Loading (Stat Panel)**

**PromQL:**
```promql
# Currently loaded classes
jvm_classes_currently_loaded{job="myapp"}
```

**Visualization:** Stat  
**Unit:** short (formatted number)  
**Color:** Value-based

---

**PANEL 8: File Descriptors (Time Series)**

**PromQL:**
```promql
# Open file descriptors
jvm_os_open_file_descriptors{job="myapp"}

# Max file descriptors
jvm_os_max_file_descriptors{job="myapp"}
```

**Visualization:** Time series  
**Unit:** none  
**Legend:**
- `Open: {{instance}}`
- `Max: {{instance}}` (dashed red line)

**Alert threshold:** Open > 80% of Max

---

### 6.3 Import Community Dashboards

**Popular JVM Dashboards:**

**Dashboard ID 4701:** "JVM (Micrometer)"
- Comprehensive JVM metrics
- Heap, non-heap, GC, threads
- Works with JMX Exporter

**Import Steps:**
1. Left sidebar → ➕ Create → Import
2. Enter dashboard ID: `4701`
3. Click "Load"
4. Select Prometheus data source
5. Click "Import"

**Other recommended IDs:**
- `8563` - Spring Boot 2.1 Statistics
- `11159` - Kafka Overview (if monitoring Kafka)
- `12239` - JVM Dashboard (detailed)

---

### 6.4 Example Dashboard JSON (Complete JVM Dashboard)

Save this as `jvm-dashboard.json`:

```json
{
  "dashboard": {
    "title": "JVM Monitoring Dashboard",
    "tags": ["java", "jvm", "prometheus"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Heap Usage %",
        "type": "gauge",
        "targets": [
          {
            "expr": "(jvm_memory_bytes_used{area=\"heap\", job=\"myapp\"} / jvm_memory_bytes_max{area=\"heap\", job=\"myapp\"}) * 100",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 70, "color": "yellow"},
                {"value": 85, "color": "red"}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Heap Memory Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "jvm_memory_bytes_used{area=\"heap\", job=\"myapp\"}",
            "legendFormat": "Used: {{instance}}",
            "refId": "A"
          },
          {
            "expr": "jvm_memory_bytes_max{area=\"heap\", job=\"myapp\"}",
            "legendFormat": "Max: {{instance}}",
            "refId": "B"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "bytes"
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 6, "y": 0}
      },
      {
        "id": 3,
        "title": "GC Collections/sec",
        "type": "timeseries",
        "targets": [
          {
            "expr": "rate(jvm_gc_collection_count_total{job=\"myapp\"}[5m])",
            "legendFormat": "{{gc}} on {{instance}}",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "ops"
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      }
    ]
  }
}
```

**Import:**
1. Dashboards → Import → Upload JSON file
2. Select `jvm-dashboard.json`
3. Choose Prometheus data source
4. Import

---

### 6.5 Advanced PromQL Queries for Dashboards

**Memory Leak Detection:**
```promql
# Heap usage trend (if increasing linearly, possible leak)
deriv(jvm_memory_bytes_used{area="heap"}[1h])
```
If result > 0 consistently → Memory leak

**GC Pressure Score:**
```promql
# Combined metric: GC frequency + pause time
(rate(jvm_gc_collection_count_total[5m]) * 10) + 
(rate(jvm_gc_collection_time_ms_total[5m]) / 1000)
```
Higher score → More GC pressure

**Thread Pool Saturation:**
```promql
# Threads approaching peak (80% of peak is concerning)
(jvm_threads_current / jvm_threads_peak) * 100
```
Alert if > 80%

**Metaspace Usage Trend:**
```promql
# Non-heap (Metaspace in Java 8+) growth rate
deriv(jvm_memory_bytes_used{area="nonheap"}[30m])
```
Positive trend → Classes being loaded faster than unloaded

---

## Continuing with sections 7-12...

Would you like me to continue with:
- Section 7: Prometheus Alerting (alert rules, Alertmanager)
- Section 8: Grafana Alerting
- Section 9: End-to-End Flow
- Sections 10-12: Verification, Troubleshooting, Best Practices

Or should I save this and create a second document for the remaining sections?
