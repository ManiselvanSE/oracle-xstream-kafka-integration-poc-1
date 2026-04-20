# JMX → Prometheus → Grafana Monitoring Guide (Part 2)
## Alerting, Troubleshooting, and Best Practices

**Continuation of JMX_PROMETHEUS_GRAFANA_MONITORING_GUIDE.md**

---

## 7. ALERTING IN PROMETHEUS (DETAILED)

### 7.1 How Prometheus Alerting Works

```
┌──────────────────────────────────────────────────────────────┐
│              PROMETHEUS ALERTING FLOW                         │
└──────────────────────────────────────────────────────────────┘

Step 1: RULE EVALUATION (Every evaluation_interval = 15s)
┌────────────────────────────────────────────────────────────┐
│  Prometheus Server                                         │
│  ├─ Loads alert.rules.yml at startup                       │
│  ├─ Every 15s:                                             │
│  │   ├─ Executes each alert rule expression (PromQL)      │
│  │   ├─ Example: jvm_memory_bytes_used / max > 0.8        │
│  │   └─ Checks if expression returns > 0 results          │
│  └─ If TRUE → Alert enters PENDING state                  │
└────────────────────────────────────────────────────────────┘
                        │
                        ▼
Step 2: PENDING STATE (Waiting for "for" duration)
┌────────────────────────────────────────────────────────────┐
│  Alert is in PENDING state                                 │
│  ├─ Condition is TRUE but not yet FIRING                   │
│  ├─ Prometheus starts timer based on "for" duration        │
│  │   Example: for: 2m                                      │
│  └─ If condition stays TRUE for full 2 minutes → FIRING    │
│                                                             │
│  If condition becomes FALSE during PENDING:                │
│  └─ Alert returns to INACTIVE (no notification sent)       │
└────────────────────────────────────────────────────────────┘
                        │
                        ▼
Step 3: FIRING STATE (Alert is active)
┌────────────────────────────────────────────────────────────┐
│  Alert enters FIRING state                                 │
│  ├─ Prometheus sends alert to Alertmanager                 │
│  ├─ Alert payload includes:                                │
│  │   ├─ Labels: {alertname, severity, instance, job}      │
│  │   ├─ Annotations: {summary, description}               │
│  │   ├─ StartsAt: timestamp when it started firing        │
│  │   └─ GeneratorURL: link to Prometheus expression       │
│  └─ Alertmanager receives alert (next step)                │
└────────────────────────────────────────────────────────────┘
                        │
                        ▼
Step 4: ALERTMANAGER PROCESSING
┌────────────────────────────────────────────────────────────┐
│  Alertmanager (localhost:9093)                             │
│  ├─ Receives alert from Prometheus                         │
│  ├─ DEDUPLICATION:                                         │
│  │   └─ If same alert from multiple Prometheus instances  │
│  │       → Send only one notification                      │
│  ├─ GROUPING:                                              │
│  │   └─ Group by labels (e.g., all alerts from job=myapp) │
│  │       → Send as single notification                     │
│  ├─ ROUTING:                                               │
│  │   └─ Match alert labels to routes in config            │
│  │       severity=critical → PagerDuty                     │
│  │       severity=warning  → Email                         │
│  ├─ SILENCING:                                             │
│  │   └─ Check if alert matches silence rule               │
│  │       (e.g., during maintenance window)                 │
│  └─ THROTTLING:                                            │
│      └─ Respect repeat_interval (don't spam)               │
└────────────────────────────────────────────────────────────┘
                        │
                        ▼
Step 5: NOTIFICATION DELIVERY
┌────────────────────────────────────────────────────────────┐
│  Send to configured receivers:                             │
│  ├─ Email (SMTP)                                           │
│  ├─ Slack (webhook)                                        │
│  ├─ PagerDuty (API)                                        │
│  ├─ Webhook (generic HTTP POST)                            │
│  └─ OpsGenie, VictorOps, etc.                              │
└────────────────────────────────────────────────────────────┘
                        │
                        ▼
Step 6: RESOLUTION (Alert clears)
┌────────────────────────────────────────────────────────────┐
│  When alert condition becomes FALSE:                       │
│  ├─ Prometheus sends "resolved" notification               │
│  ├─ Alertmanager forwards to same receivers                │
│  └─ Notification says "RESOLVED" + duration                │
└────────────────────────────────────────────────────────────┘
```

---

### 7.2 Create Alert Rules File

```bash
sudo vim /etc/prometheus/alert.rules.yml
```

```yaml
# Prometheus Alert Rules
# Purpose: Define conditions that trigger notifications

groups:
  # =============================================
  # GROUP 1: JVM MEMORY ALERTS
  # =============================================
  - name: jvm_memory
    interval: 15s  # How often to evaluate these rules
    
    rules:
      # Alert 1: High Heap Usage
      - alert: HighHeapUsage
        expr: |
          (jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) > 0.80
        for: 2m
        labels:
          severity: warning
          component: jvm
          team: backend
        annotations:
          summary: "High heap memory usage on {{ $labels.instance }}"
          description: |
            Heap memory usage is {{ $value | humanizePercentage }}.
            Current: {{ $labels.instance }}
            Job: {{ $labels.job }}
            Threshold: 80%
          dashboard: "http://grafana:3000/d/jvm-dashboard"
          runbook: "https://wiki.example.com/runbooks/high-heap-usage"

      # Alert 2: Critical Heap Usage
      - alert: CriticalHeapUsage
        expr: |
          (jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) > 0.90
        for: 1m
        labels:
          severity: critical
          component: jvm
          team: backend
          pagerduty: "true"
        annotations:
          summary: "CRITICAL: Heap memory usage on {{ $labels.instance }}"
          description: |
            Heap memory usage is {{ $value | humanizePercentage }}.
            Application is at risk of OutOfMemoryError.
            Instance: {{ $labels.instance }}
            Job: {{ $labels.job }}
          action: "Consider restarting application or increasing heap size"

      # Alert 3: Heap Memory Leak Detected
      - alert: HeapMemoryLeak
        expr: |
          deriv(jvm_memory_bytes_used{area="heap"}[1h]) > 10000000
        for: 30m
        labels:
          severity: warning
          component: jvm
          team: backend
        annotations:
          summary: "Possible memory leak on {{ $labels.instance }}"
          description: |
            Heap memory is growing at {{ $value | humanize }}B/sec over last hour.
            This indicates a potential memory leak.
            Instance: {{ $labels.instance }}
          action: "Analyze heap dump, check for retained objects"

  # =============================================
  # GROUP 2: GARBAGE COLLECTION ALERTS
  # =============================================
  - name: jvm_gc
    interval: 15s
    
    rules:
      # Alert 4: High GC Time
      - alert: HighGCTime
        expr: |
          (rate(jvm_gc_collection_time_ms_total[5m]) / 1000) > 0.05
        for: 5m
        labels:
          severity: warning
          component: jvm
          team: backend
        annotations:
          summary: "High GC time on {{ $labels.instance }}"
          description: |
            Application spending {{ $value | humanizePercentage }} of time in GC.
            Target: <2%, Current: {{ $value | humanizePercentage }}
            GC Type: {{ $labels.gc }}
            Instance: {{ $labels.instance }}
          action: "Check heap size, analyze GC logs, consider tuning GC parameters"

      # Alert 5: Frequent GC Collections
      - alert: FrequentGC
        expr: |
          rate(jvm_gc_collection_count_total[5m]) > 5
        for: 5m
        labels:
          severity: warning
          component: jvm
        annotations:
          summary: "Frequent GC on {{ $labels.instance }}"
          description: |
            GC running {{ $value }} times per second.
            This indicates heap pressure.
            GC: {{ $labels.gc }}

  # =============================================
  # GROUP 3: THREAD ALERTS
  # =============================================
  - name: jvm_threads
    interval: 15s
    
    rules:
      # Alert 6: High Thread Count
      - alert: HighThreadCount
        expr: |
          jvm_threads_current > 500
        for: 3m
        labels:
          severity: warning
          component: jvm
        annotations:
          summary: "High thread count on {{ $labels.instance }}"
          description: |
            Thread count: {{ $value }}
            Normal range: 50-200 for typical apps
            Check for thread leaks or connection pool issues.

      # Alert 7: Thread Count Spike
      - alert: ThreadCountSpike
        expr: |
          delta(jvm_threads_current[5m]) > 100
        for: 2m
        labels:
          severity: warning
          component: jvm
        annotations:
          summary: "Thread count spike on {{ $labels.instance }}"
          description: |
            Thread count increased by {{ $value }} in last 5 minutes.
            This may indicate a sudden load increase or thread leak.

      # Alert 8: Approaching Thread Limit
      - alert: ApproachingThreadLimit
        expr: |
          (jvm_threads_current / jvm_threads_peak) > 0.90
        for: 5m
        labels:
          severity: critical
          component: jvm
        annotations:
          summary: "Thread count approaching peak on {{ $labels.instance }}"
          description: |
            Current threads: {{ $value | humanizePercentage }} of peak.
            May hit OS thread limit soon.

  # =============================================
  # GROUP 4: CPU ALERTS
  # =============================================
  - name: jvm_cpu
    interval: 15s
    
    rules:
      # Alert 9: High CPU Usage
      - alert: HighCPUUsage
        expr: |
          jvm_process_cpu_load > 0.80
        for: 5m
        labels:
          severity: warning
          component: jvm
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: |
            CPU usage: {{ $value | humanizePercentage }}
            Check for CPU-intensive operations or infinite loops.

  # =============================================
  # GROUP 5: FILE DESCRIPTOR ALERTS
  # =============================================
  - name: jvm_fd
    interval: 15s
    
    rules:
      # Alert 10: High File Descriptor Usage
      - alert: HighFileDescriptorUsage
        expr: |
          (jvm_os_open_file_descriptors / jvm_os_max_file_descriptors) > 0.80
        for: 5m
        labels:
          severity: warning
          component: jvm
        annotations:
          summary: "High file descriptor usage on {{ $labels.instance }}"
          description: |
            FD usage: {{ $value | humanizePercentage }}
            Open: {{ jvm_os_open_file_descriptors }}
            Max: {{ jvm_os_max_file_descriptors }}
          action: "Check for leaked connections, increase ulimit if needed"

  # =============================================
  # GROUP 6: APPLICATION AVAILABILITY
  # =============================================
  - name: availability
    interval: 15s
    
    rules:
      # Alert 11: Instance Down
      - alert: InstanceDown
        expr: |
          up == 0
        for: 1m
        labels:
          severity: critical
          component: infrastructure
          pagerduty: "true"
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: |
            Prometheus cannot scrape metrics from {{ $labels.instance }}.
            Job: {{ $labels.job }}
            Check if application is running, firewall rules, network connectivity.

      # Alert 12: Scrape Duration High
      - alert: ScrapeDurationHigh
        expr: |
          scrape_duration_seconds > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow metric scraping on {{ $labels.instance }}"
          description: |
            Scrape took {{ $value }}s (threshold: 10s).
            This may indicate application performance issues.
```

**Save and validate:**
```bash
# Validate alert rules
promtool check rules /etc/prometheus/alert.rules.yml

# Expected output:
# Checking /etc/prometheus/alert.rules.yml
#   SUCCESS: 6 groups, 12 rules found

# If errors, you'll see line numbers and issue description
```

---

### 7.3 Reload Prometheus Configuration

```bash
# Method 1: Send HUP signal
sudo systemctl reload prometheus

# Method 2: HTTP API (requires --web.enable-lifecycle flag)
curl -X POST http://localhost:9090/-/reload

# Method 3: Restart (less preferred)
sudo systemctl restart prometheus

# Verify rules loaded
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[].name'

# Expected output:
# "jvm_memory"
# "jvm_gc"
# "jvm_threads"
# "jvm_cpu"
# "jvm_fd"
# "availability"
```

---

### 7.4 View Alerts in Prometheus UI

```
URL: http://localhost:9090/alerts

You'll see:
- Alert name
- State: INACTIVE | PENDING | FIRING
- Labels
- Value
- Active since (for FIRING alerts)
```

---

### 7.5 Alertmanager Setup

**Install Alertmanager:**

```bash
# Download
cd /tmp
curl -LO https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

# Extract
tar -xzf alertmanager-0.27.0.linux-amd64.tar.gz

# Move binary
sudo cp alertmanager-0.27.0.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-0.27.0.linux-amd64/amtool /usr/local/bin/

# Create user
sudo useradd --no-create-home --shell /bin/false alertmanager

# Create directories
sudo mkdir -p /etc/alertmanager
sudo mkdir -p /var/lib/alertmanager

# Set ownership
sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager
sudo chown alertmanager:alertmanager /usr/local/bin/amtool
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager
```

---

**Configure Alertmanager:**

```bash
sudo vim /etc/alertmanager/alertmanager.yml
```

```yaml
# Alertmanager Configuration

# Global settings
global:
  # Email SMTP settings
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'your-app-password'
  smtp_require_tls: true
  
  # Slack webhook (global default)
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
  
  # Default receiver if no route matches
  resolve_timeout: 5m

# Templates for custom notifications
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  # Default receiver for all alerts
  receiver: 'default-email'
  
  # Group alerts by these labels
  group_by: ['alertname', 'cluster', 'service']
  
  # Wait before sending first notification (batch alerts together)
  group_wait: 30s
  
  # Wait before sending notification about new alerts in same group
  group_interval: 5m
  
  # How long to wait before re-sending notification
  repeat_interval: 4h
  
  # Child routes (specific handling)
  routes:
    # Route 1: Critical alerts to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true  # Also send to default receiver
      repeat_interval: 15m
    
    # Route 2: Warning alerts to Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_interval: 10m
      repeat_interval: 2h
    
    # Route 3: Backend team alerts
    - match:
        team: backend
      receiver: 'backend-team-email'
    
    # Route 4: Database alerts (different channel)
    - match:
        component: database
      receiver: 'dba-team'

# Inhibition rules (suppress alerts)
inhibit_rules:
  # If CriticalHeapUsage is firing, suppress HighHeapUsage
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance', 'alertname']
  
  # If instance is down, suppress all other alerts from same instance
  - source_match:
      alertname: 'InstanceDown'
    target_match_re:
      alertname: '.*'
    equal: ['instance']

# Receivers (notification endpoints)
receivers:
  # Default email receiver
  - name: 'default-email'
    email_configs:
      - to: 'ops-team@example.com'
        headers:
          Subject: '{{ .GroupLabels.alertname }} - {{ .GroupLabels.instance }}'
        html: |
          <h2>Alert: {{ .GroupLabels.alertname }}</h2>
          <b>Summary:</b> {{ .CommonAnnotations.summary }}<br>
          <b>Description:</b> {{ .CommonAnnotations.description }}<br>
          <b>Instance:</b> {{ .GroupLabels.instance }}<br>
          <b>Severity:</b> {{ .GroupLabels.severity }}<br>
          <b>Started:</b> {{ .StartsAt }}<br>
          <a href="{{ .CommonAnnotations.dashboard }}">View Dashboard</a>

  # Slack receiver for warnings
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warning'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          *Summary:* {{ .CommonAnnotations.summary }}
          *Instance:* {{ .GroupLabels.instance }}
          *Severity:* {{ .GroupLabels.severity }}
          *Description:* {{ .CommonAnnotations.description }}
        color: 'warning'
        send_resolved: true

  # PagerDuty for critical alerts
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          instance: '{{ .GroupLabels.instance }}'

  # Backend team email
  - name: 'backend-team-email'
    email_configs:
      - to: 'backend-team@example.com'
        send_resolved: true

  # DBA team (email + Slack)
  - name: 'dba-team'
    email_configs:
      - to: 'dba-team@example.com'
    slack_configs:
      - channel: '#dba-alerts'
        send_resolved: true
```

**Validate configuration:**
```bash
amtool check-config /etc/alertmanager/alertmanager.yml

# Expected:
# Checking '/etc/alertmanager/alertmanager.yml'  SUCCESS
# Found:
#  - global config
#  - route
#  - 0 inhibit rules
#  - 5 receivers
#  - 0 templates
```

---

**Create systemd service:**

```bash
sudo vim /etc/systemd/system/alertmanager.service
```

```ini
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/alerting/alertmanager/
After=network.target

[Service]
Type=simple
User=alertmanager
Group=alertmanager

ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager/ \
  --web.listen-address=0.0.0.0:9093 \
  --cluster.listen-address= \
  --log.level=info

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**Start Alertmanager:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager

# Access Web UI
open http://localhost:9093
```

---

### 7.6 Testing Alerts

**Method 1: Trigger alert manually**
```bash
# Temporarily lower threshold in alert.rules.yml
# Change: > 0.80 to > 0.01
sudo vim /etc/prometheus/alert.rules.yml

# Reload Prometheus
sudo systemctl reload prometheus

# Wait 2-3 minutes, check Prometheus UI
open http://localhost:9090/alerts

# Should see alerts in FIRING state
```

**Method 2: Use amtool to send test alert**
```bash
# Send test alert to Alertmanager
amtool alert add test_alert \
  alertname=TestAlert \
  severity=warning \
  instance=test-instance \
  --annotation=summary="This is a test alert"

# Check Alertmanager UI
open http://localhost:9093/#/alerts
```

**Method 3: Cause actual high heap usage**
```bash
# In Java application, allocate large objects
# This will trigger HighHeapUsage alert naturally
```

---

## 8. ALERTING IN GRAFANA

### 8.1 Prometheus Alerts vs Grafana Alerts

| Feature | Prometheus Alerts | Grafana Alerts |
|---------|------------------|----------------|
| **Definition** | YAML files (alert.rules.yml) | Grafana UI or provisioning |
| **Query Language** | PromQL only | PromQL, SQL, LogQL, etc. |
| **Evaluation** | Prometheus server | Grafana server |
| **Multi-datasource** | No (Prometheus only) | Yes (combine Prometheus + MySQL) |
| **Visualization** | Basic (Prometheus UI) | Rich (charts in alert notifications) |
| **Notification** | Via Alertmanager | Direct from Grafana |
| **State Management** | Prometheus TSDB | Grafana database |
| **Best For** | Infrastructure SLIs | Dashboard-specific thresholds |

**When to use:**
- **Prometheus:** Production-critical alerts (SLA breaches, infrastructure failures)
- **Grafana:** Exploratory alerts, dashboard-driven monitoring, multi-datasource correlation

---

### 8.2 Grafana Alerting Architecture (Unified Alerting)

Grafana 8+ uses **Unified Alerting** (replaces legacy alerting):

```
┌──────────────────────────────────────────────────────────┐
│  Grafana Unified Alerting                                │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Alert Rules                                       │  │
│  │  - Query: PromQL, SQL, LogQL                       │  │
│  │  - Condition: threshold, range, no data            │  │
│  │  - Evaluation interval: 1m, 5m, etc.               │  │
│  │  - Folder: organize by team/service                │  │
│  └───────────────────┬────────────────────────────────┘  │
│                      │                                    │
│                      ▼                                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Notification Policies                             │  │
│  │  - Match labels (severity, team)                   │  │
│  │  - Group by labels                                 │  │
│  │  - Timing: group_wait, group_interval, repeat      │  │
│  └───────────────────┬────────────────────────────────┘  │
│                      │                                    │
│                      ▼                                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Contact Points                                    │  │
│  │  - Email                                           │  │
│  │  - Slack                                           │  │
│  │  - PagerDuty                                       │  │
│  │  - Webhook                                         │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

### 8.3 Step-by-Step: Create Grafana Alert

**Example: Alert on High CPU Usage**

**Step 1: Navigate to Alerting**
1. Left sidebar → 🔔 Alerting → Alert rules
2. Click "New alert rule"

**Step 2: Define Query**
- **Rule name:** `High CPU Usage - MyApp`
- **Data source:** `Prometheus`
- **Query A:**
  ```promql
  jvm_process_cpu_load{job="myapp"} * 100
  ```
- **Alias:** `CPU Usage %`

**Step 3: Set Conditions**
- **Condition:** `WHEN last() OF query(A) IS ABOVE 80`
- Explanation: Trigger when CPU > 80%

**Step 4: Configure Evaluation**
- **Folder:** `JVM Alerts` (create if doesn't exist)
- **Evaluation group:** `jvm-metrics` (or create new)
- **Evaluation interval:** `1m` (check every minute)
- **Pending period:** `3m` (must be true for 3 minutes before firing)

**Step 5: Add Annotations**
- **Summary:** `High CPU usage on {{ $labels.instance }}`
- **Description:**
  ```
  CPU usage is {{ $values.A }}%
  Instance: {{ $labels.instance }}
  Job: {{ $labels.job }}
  
  Threshold: 80%
  ```
- **Runbook URL:** `https://wiki.example.com/runbooks/high-cpu`
- **Dashboard UID:** (select your JVM dashboard)
- **Panel ID:** (select CPU panel)

**Step 6: Add Labels**
- `severity` = `warning`
- `team` = `backend`
- `component` = `jvm`

**Step 7: Preview Alert**
- Click "Preview" to see if query returns data
- Check if condition evaluates correctly

**Step 8: Save**
- Click "Save rule and exit"

---

### 8.4 Configure Contact Points

**Step 1: Navigate to Contact Points**
1. Alerting → Contact points
2. Click "New contact point"

**Email Contact Point:**
```
Name: ops-team-email
Integration: Email
Addresses: ops-team@example.com, oncall@example.com
Subject: [Grafana Alert] {{ .GroupLabels.alertname }}
Message: (leave default or customize)
```

**Slack Contact Point:**
```
Name: slack-alerts
Integration: Slack
Webhook URL: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
Username: Grafana Alerts
Channel: #alerts
Title: {{ .GroupLabels.alertname }}
Text:
  *Summary:* {{ .CommonAnnotations.summary }}
  *Instance:* {{ .GroupLabels.instance }}
  *Value:* {{ .Values.A }}%
  *Severity:* {{ .GroupLabels.severity }}
```

**Test Contact Point:**
- Click "Test" button
- Should receive test notification

---

### 8.5 Configure Notification Policies

**Step 1: Navigate to Notification Policies**
1. Alerting → Notification policies
2. Edit default policy or create new

**Example Policy:**
```
Default notification policy:
├─ Contact point: ops-team-email
├─ Group by: alertname, instance
├─ Timing:
│   ├─ Group wait: 30s
│   ├─ Group interval: 5m
│   └─ Repeat interval: 4h
│
├─ Nested policy 1: Critical alerts
│   ├─ Match labels: severity = critical
│   ├─ Contact point: pagerduty
│   ├─ Continue: Yes (also notify default)
│   └─ Repeat interval: 15m
│
└─ Nested policy 2: Backend team
    ├─ Match labels: team = backend
    ├─ Contact point: slack-backend
    └─ Group interval: 10m
```

**Create nested policy:**
1. Click "New nested policy"
2. **Label matchers:**
   - `severity` = `critical`
3. **Contact point:** `pagerduty`
4. **Continue matching:** ☑️ (send to both PagerDuty and default)
5. **Timing overrides:**
   - Repeat interval: `15m`
6. Save

---

### 8.6 Complete Example: Multi-Condition Alert

**Use Case:** Alert if heap usage > 80% AND GC time > 5%

**Step 1: Create alert rule**
```
Rule name: JVM Under Pressure
Folder: JVM Alerts
Evaluation group: jvm-health
Interval: 1m
```

**Step 2: Define queries**

**Query A (Heap Usage):**
```promql
(jvm_memory_bytes_used{area="heap", job="myapp"} / jvm_memory_bytes_max{area="heap", job="myapp"}) * 100
```

**Query B (GC Time %):**
```promql
(rate(jvm_gc_collection_time_ms_total{job="myapp"}[5m]) / 1000) * 100
```

**Query C (Reduce - Math Expression):**
```
$A > 80 && $B > 5
```

**Step 3: Condition**
```
WHEN last() OF query(C) IS ABOVE 0
```
Explanation: Expression `$A > 80 && $B > 5` returns 1 if true, 0 if false

**Step 4: Configure**
```
Pending: 2m
Labels:
  - severity: critical
  - component: jvm
  - team: backend
Annotations:
  - summary: JVM under pressure on {{ $labels.instance }}
  - description: |
      Heap: {{ $values.A }}%
      GC Time: {{ $values.B }}%
      Both metrics exceeded thresholds simultaneously.
```

---

### 8.7 Silencing Alerts

**Temporary silence (e.g., during maintenance):**

1. Alerting → Silences
2. Click "New silence"
3. **Matchers:**
   - `instance` = `192.168.1.100:7071`
   - `severity` = `warning`
4. **Duration:** `2h`
5. **Comment:** "Scheduled maintenance - heap analysis"
6. Create

**All alerts matching those labels won't notify for 2 hours.**

---

### 8.8 Mute Timings

**Mute alerts during specific hours (e.g., weekends):**

1. Alerting → Notification policies → Mute timings
2. Click "New mute timing"
3. **Name:** `weekends`
4. **Time intervals:**
   - Days of week: Saturday, Sunday
   - Time range: 00:00 - 23:59
5. Save

**Apply to notification policy:**
1. Edit notification policy
2. **Mute timings:** Select `weekends`
3. Save

**Now alerts won't notify on weekends (but still fire/resolve).**

---

## 9. END-TO-END FLOW EXPLANATION

### 9.1 Complete Timeline (15-Second Cycle)

```
T = 0s: METRICS GENERATION
────────────────────────────────────────────────────────────────
JVM Runtime:
├─ Allocates 100 MB in heap (Eden space)
├─ Thread pool creates 5 new threads
├─ GC runs (Young Generation)
└─ MBean Server updates attributes:
    ├─ java.lang:type=Memory → HeapMemoryUsage.used = 600 MB
    ├─ java.lang:type=Threading → ThreadCount = 150
    └─ java.lang:type=GC,name=G1YoungGen → CollectionCount++

T = 0.001s: JMX EXPORTER READS MBEANS
────────────────────────────────────────────────────────────────
JMX Exporter (Java Agent):
├─ Queries MBean Server (local, no network)
├─ Applies YAML config rules
├─ Translates to Prometheus format:
│   jvm_memory_bytes_used{area="heap"} 629145600
│   jvm_threads_current 150
│   jvm_gc_collection_count_total{gc="G1 Young Generation"} 245
└─ Caches metrics in memory (ready for scrape)

T = 5s: PROMETHEUS SCRAPES
────────────────────────────────────────────────────────────────
Prometheus Server:
├─ HTTP GET http://192.168.1.100:7071/metrics
├─ Receives text response (3000 lines, 50 KB)
├─ Parses each metric line
├─ Stores in TSDB:
│   Metric: jvm_memory_bytes_used
│   Labels: {area="heap", instance="192.168.1.100:7071", job="myapp"}
│   Value: 629145600
│   Timestamp: 1713607205
├─ Scrape duration: 245ms
└─ scrape_samples_scraped{job="myapp"} 3000

T = 15s: ALERT RULE EVALUATION
────────────────────────────────────────────────────────────────
Prometheus Alert Engine:
├─ Loads alert.rules.yml (6 groups, 12 rules)
├─ For each rule:
│   ├─ Execute PromQL expression
│   ├─ Example: (jvm_memory_bytes_used / jvm_memory_bytes_max) > 0.8
│   ├─ Query TSDB for latest values
│   ├─ Result: 0.786 (78.6% - below threshold)
│   └─ Alert state: INACTIVE
├─ Another rule:
│   ├─ Expression: jvm_threads_current > 500
│   ├─ Result: 150 (below threshold)
│   └─ Alert state: INACTIVE
└─ Total evaluation time: 12ms

T = 20s: PROMETHEUS SCRAPES AGAIN
────────────────────────────────────────────────────────────────
(Repeat of T=5s cycle)
New data point stored for each metric.

T = 30s: ALERT RULE EVALUATION AGAIN
────────────────────────────────────────────────────────────────
Prometheus Alert Engine:
├─ Re-evaluates all rules
├─ Rule: HighHeapUsage
│   ├─ Expression: (jvm_memory_bytes_used / jvm_memory_bytes_max) > 0.8
│   ├─ Result: 0.82 (82% - ABOVE threshold!)
│   └─ Alert state: INACTIVE → PENDING (starts timer)
└─ Timer started: PENDING for 0s (need 2m as per "for" duration)

T = 45s: ANOTHER EVALUATION
────────────────────────────────────────────────────────────────
├─ Heap still at 82%
└─ Alert state: PENDING for 15s (need 2m total)

T = 60s, 75s, 90s, 105s, 120s: CONTINUED EVALUATIONS
────────────────────────────────────────────────────────────────
├─ Heap remains > 80% for all cycles
└─ Alert PENDING duration accumulates

T = 150s (2 minutes 30 seconds): ALERT FIRES
────────────────────────────────────────────────────────────────
Prometheus Alert Engine:
├─ HighHeapUsage has been true for > 2 minutes
├─ Alert state: PENDING → FIRING
├─ Sends alert to Alertmanager:
│   POST http://localhost:9093/api/v1/alerts
│   Body:
│   [
│     {
│       "labels": {
│         "alertname": "HighHeapUsage",
│         "severity": "warning",
│         "instance": "192.168.1.100:7071",
│         "job": "myapp"
│       },
│       "annotations": {
│         "summary": "High heap memory usage on 192.168.1.100:7071",
│         "description": "Heap memory usage is 82%..."
│       },
│       "startsAt": "2026-04-20T10:02:30Z",
│       "endsAt": "0001-01-01T00:00:00Z",
│       "generatorURL": "http://localhost:9090/graph?g0.expr=..."
│     }
│   ]
└─ HTTP 200 OK received from Alertmanager

T = 150.1s: ALERTMANAGER PROCESSES ALERT
────────────────────────────────────────────────────────────────
Alertmanager:
├─ Receives alert
├─ Deduplication: checks if same alert already exists (no)
├─ Grouping: groups by alertname + instance
│   Group key: {alertname="HighHeapUsage", instance="192.168.1.100:7071"}
├─ group_wait timer starts: wait 30s before sending (batch similar alerts)
└─ Alert stored in memory

T = 180s (30s group_wait passed): SEND NOTIFICATION
────────────────────────────────────────────────────────────────
Alertmanager:
├─ group_wait timer expired
├─ Routing: checks route tree
│   ├─ Labels: severity=warning, team=backend
│   ├─ Matches route: "slack-warnings"
│   └─ Receiver: slack-warnings
├─ Sends to Slack:
│   POST https://hooks.slack.com/services/...
│   Body:
│   {
│     "text": "*High heap memory usage on 192.168.1.100:7071*\nHeap: 82%...",
│     "attachments": [...]
│   }
├─ HTTP 200 OK received from Slack
└─ Notification log updated: sent to slack-warnings at T=180s

T = 180.5s: USER RECEIVES NOTIFICATION
────────────────────────────────────────────────────────────────
Slack channel #alerts-warning:
├─ Message appears: "High heap memory usage..."
├─ User clicks "View Dashboard" link
└─ Opens Grafana dashboard showing heap graph

T = 200s: GRAFANA QUERY (User Action)
────────────────────────────────────────────────────────────────
User opens Grafana dashboard:
├─ Grafana queries Prometheus:
│   POST http://localhost:9090/api/v1/query_range
│   Body:
│   {
│     "query": "jvm_memory_bytes_used{area='heap', job='myapp'}",
│     "start": 1713606600,
│     "end": 1713607200,
│     "step": 15
│   }
├─ Prometheus returns 40 data points (10 minutes at 15s intervals)
├─ Grafana renders graph:
│   └─ Shows heap climbing from 70% → 82% over last 10 minutes
└─ User sees visualization

T = 200s (concurrent): GRAFANA ALERT EVALUATION
────────────────────────────────────────────────────────────────
Grafana Unified Alerting:
├─ Evaluates "High CPU Usage" alert rule (separate from Prometheus)
├─ Queries Prometheus:
│   jvm_process_cpu_load{job="myapp"} * 100
├─ Result: 45% (below 80% threshold)
└─ Grafana alert state: Normal

T = 300s: USER RESTARTS APPLICATION
────────────────────────────────────────────────────────────────
Sysadmin:
├─ systemctl restart myapp
├─ JVM starts with fresh heap (20 MB used)
└─ Application reinitializes

T = 320s: PROMETHEUS SCRAPES POST-RESTART
────────────────────────────────────────────────────────────────
Prometheus:
├─ Scrapes metrics
├─ jvm_memory_bytes_used{area="heap"} = 20971520 (20 MB)
└─ Stores new data point

T = 330s: ALERT RESOLVES
────────────────────────────────────────────────────────────────
Prometheus Alert Engine:
├─ Evaluates HighHeapUsage rule
├─ Expression: 20MB / 2GB = 0.01 (1% - below 80%)
├─ Alert state: FIRING → INACTIVE (resolved)
├─ Sends resolution to Alertmanager:
│   [
│     {
│       "labels": {...},
│       "endsAt": "2026-04-20T10:05:30Z"  # <- now populated
│     }
│   ]
└─ Alertmanager receives resolution

T = 360s: RESOLVED NOTIFICATION SENT
────────────────────────────────────────────────────────────────
Alertmanager:
├─ Checks if send_resolved: true (it is for Slack)
├─ Sends to Slack:
│   {
│     "text": "[RESOLVED] High heap memory usage on 192.168.1.100:7071",
│     "color": "good"
│   }
└─ Slack shows green "[RESOLVED]" message

T = 360.5s: USER SEES RESOLUTION
────────────────────────────────────────────────────────────────
Slack:
├─ "[RESOLVED] High heap memory usage..."
├─ Duration: 3m 30s
└─ User confirms issue fixed
```

---

### 9.2 Timing Concepts Summary

**scrape_interval (15s):**
- How often Prometheus pulls metrics
- Determines metric resolution
- Trade-off: shorter = more accurate but more load/storage

**evaluation_interval (15s):**
- How often Prometheus evaluates alert rules
- Should be ≤ scrape_interval
- Affects how quickly alerts detect issues

**for duration (2m in HighHeapUsage example):**
- How long condition must be true before FIRING
- Prevents flapping (brief spikes don't alert)
- Balance: too short = noise, too long = delayed response

**group_wait (30s):**
- Alertmanager waits this long before first notification
- Allows batching similar alerts into one notification
- Example: 5 pods failing at once → 1 notification instead of 5

**group_interval (5m):**
- After first notification, wait this long before notifying about NEW alerts in same group
- Prevents spam when multiple related alerts fire in sequence

**repeat_interval (4h):**
- How often to re-send notification if alert still firing
- Acts as reminder
- Example: Alert at 10:00, remind at 14:00, 18:00, etc.

---

## 10. VERIFICATION STEPS

### 10.1 Verify JMX is Exposed

**If using remote JMX (port 9999):**
```bash
# Check port is listening
ss -tlnp | grep 9999
# Expected: LISTEN on 0.0.0.0:9999

# Connect with JConsole
jconsole <hostname>:9999

# OR use jmxterm
echo "domains" | java -jar jmxterm.jar -l <hostname>:9999
# Expected: List of domains including java.lang
```

**If using JMX Exporter (recommended):**
```bash
# JMX is local-only, no port check needed
# Verify via exporter HTTP endpoint (next section)
```

---

### 10.2 Verify JMX Exporter is Working

```bash
# Check HTTP port is listening
ss -tlnp | grep 7071
# Expected: LISTEN on 0.0.0.0:7071

# Fetch metrics
curl http://localhost:7071/metrics

# Should see output like:
# HELP jvm_memory_bytes_used JVM heap memory usage in bytes
# TYPE jvm_memory_bytes_used gauge
# jvm_memory_bytes_used{area="heap"} 5.24288E8

# Count metrics
curl -s http://localhost:7071/metrics | grep "^jvm_" | wc -l
# Expected: 50-200 metrics (depends on config)

# Check specific metric
curl -s http://localhost:7071/metrics | grep jvm_memory_bytes_used
# Expected: Multiple lines with different labels (heap, nonheap)

# Verify from remote machine
curl http://<app-server-ip>:7071/metrics | head -20
```

**If no metrics:**
- Check JVM started with `-javaagent` flag: `ps aux | grep javaagent`
- Check exporter logs in application startup logs
- Verify config file path is correct
- Validate YAML syntax: `yamllint /opt/jmx_exporter/config.yaml`

---

### 10.3 Verify Prometheus Targets are UP

**Via Web UI:**
```
1. Open http://localhost:9090/targets
2. Find your job (e.g., "myapp")
3. Check "State" column:
   - UP (green) ✅ = Working
   - DOWN (red) ❌ = Failed
   - UNKNOWN (gray) = Never scraped yet
```

**Via API:**
```bash
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job == "myapp")'

# Expected output:
# {
#   "discoveredLabels": {...},
#   "labels": {
#     "instance": "192.168.1.100:7071",
#     "job": "myapp"
#   },
#   "scrapePool": "myapp",
#   "scrapeUrl": "http://192.168.1.100:7071/metrics",
#   "globalUrl": "http://192.168.1.100:7071/metrics",
#   "lastError": "",
#   "lastScrape": "2026-04-20T10:30:15.123Z",
#   "lastScrapeDuration": 0.245,
#   "health": "up",  # <-- KEY: should be "up"
#   "scrapeInterval": "15s",
#   "scrapeTimeout": "10s"
# }
```

**If target shows DOWN:**
```bash
# 1. Check "lastError" field in API response
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job == "myapp") | .lastError'

# Common errors:
# "dial tcp 192.168.1.100:7071: connect: connection refused"
#   → JMX Exporter not running or wrong port

# "dial tcp 192.168.1.100:7071: i/o timeout"
#   → Firewall blocking, network issue

# "server returned HTTP status 404 Not Found"
#   → Wrong metrics_path in prometheus.yml

# 2. Test manually from Prometheus server
curl http://192.168.1.100:7071/metrics
# If this works but Prometheus shows DOWN, check prometheus.yml syntax

# 3. Check Prometheus logs
sudo journalctl -u prometheus | grep "error.*scrape"
```

---

### 10.4 Verify Metrics in Prometheus

**Check if metrics are being stored:**
```bash
# Query Prometheus API
curl -G http://localhost:9090/api/v1/query \
     --data-urlencode 'query=jvm_memory_bytes_used{job="myapp"}' | jq .

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
#         "value": [1713607800, "524288000"]  # [timestamp, value]
#       }
#     ]
#   }
# }
```

**If result array is empty:**
```bash
# 1. Check if metric name is correct
curl -s http://192.168.1.100:7071/metrics | grep jvm_memory

# 2. Check if Prometheus is scraping
curl -s http://localhost:9090/api/v1/query \
     --data-urlencode 'query=up{job="myapp"}' | jq .
# If up=0, target is down

# 3. Check for dropped metrics (metric_relabel_configs)
# Review prometheus.yml for drop rules

# 4. Query without labels (broader search)
curl -G http://localhost:9090/api/v1/query \
     --data-urlencode 'query=jvm_memory_bytes_used' | jq .
```

---

### 10.5 Verify Grafana Dashboards Show Data

**Dashboard showing "No Data":**

**Step 1: Check data source connection**
```
Grafana UI:
├─ Configuration → Data Sources → Prometheus
├─ Click "Test"
└─ Should show: "Data source is working"
```

**Step 2: Test query in Explore**
```
Grafana UI:
├─ Explore tab
├─ Data source: Prometheus
├─ Query: jvm_memory_bytes_used{job="myapp"}
├─ Run query
└─ Should show time series data
```

**Step 3: Check panel query**
```
Dashboard:
├─ Edit panel
├─ Check "Query inspector"
├─ Look for:
│   ├─ Query: (should show your PromQL)
│   ├─ Response: (should show data array)
│   └─ Errors: (any error messages)
```

**Step 4: Check time range**
```
Dashboard:
├─ Time range selector (top right)
├─ Set to "Last 15 minutes"
└─ Refresh dashboard
```

**If still no data:**
- Verify metric exists in Prometheus (step 10.4)
- Check if query has correct labels: `{job="myapp"}` (case-sensitive!)
- Try simpler query: `up` (should always work)
- Check Grafana logs: `sudo journalctl -u grafana-server | grep error`

---

### 10.6 Verify Alerts are Working

**Check Prometheus alerts:**
```
Prometheus UI:
├─ http://localhost:9090/alerts
├─ Should see your alert rules listed
├─ States:
│   ├─ INACTIVE (green) = Condition false
│   ├─ PENDING (yellow) = Condition true, waiting for "for" duration
│   └─ FIRING (red) = Condition true for > "for" duration
```

**Trigger test alert:**
```bash
# Manually set low threshold to trigger alert
sudo vim /etc/prometheus/alert.rules.yml

# Change:
expr: jvm_memory_bytes_used > 0.80
# To:
expr: jvm_memory_bytes_used > 0.01  # Will always be true

# Reload Prometheus
curl -X POST http://localhost:9090/-/reload

# Wait 2-3 minutes (for "for" duration)
# Check alerts page - should show FIRING

# Revert change after test
```

**Check Alertmanager received alert:**
```
Alertmanager UI:
├─ http://localhost:9093/#/alerts
├─ Should see FIRING alerts
├─ Check:
│   ├─ Labels match alert rule
│   ├─ Annotations populated
│   └─ Receiver shown
```

**Check notification was sent:**
```bash
# Check Alertmanager logs
sudo journalctl -u alertmanager | grep "notification sent"

# For email:
# Check email inbox (may be in spam folder)

# For Slack:
# Check Slack channel

# For webhook:
# Check target server logs
```

---

## 11. COMMON ISSUES & TROUBLESHOOTING

### 11.1 JMX Port Not Accessible

**Symptom:**
```bash
telnet <hostname> 9999
# Output: Connection refused
```

**Causes & Fixes:**

**1. JMX not enabled**
```bash
# Check JVM startup flags
ps aux | grep java | grep jmxremote

# If not present, JMX is not enabled
# Add flags (see section 2.2)
```

**2. Wrong hostname**
```bash
# JVM advertising wrong hostname for RMI callbacks
# Fix: Add -Djava.rmi.server.hostname=<public-ip>

# Example:
java -Djava.rmi.server.hostname=192.168.1.100 \
     -Dcom.sun.management.jmxremote.port=9999 \
     ...
```

**3. Firewall blocking**
```bash
# Check if port is listening
ss -tlnp | grep 9999

# If listening but still can't connect externally, check firewall
sudo iptables -L -n | grep 9999

# Open port
sudo iptables -A INPUT -p tcp --dport 9999 -j ACCEPT
sudo iptables-save

# For firewalld
sudo firewall-cmd --add-port=9999/tcp --permanent
sudo firewall-cmd --reload
```

**4. SELinux blocking**
```bash
# Check SELinux status
getenforce
# If "Enforcing", try:
sudo setenforce 0  # Temporary disable

# If this fixes it, add permanent rule:
sudo semanage port -a -t http_port_t -p tcp 9999
```

---

### 11.2 JMX Exporter Not Exposing Metrics

**Symptom:**
```bash
curl http://localhost:7071/metrics
# Output: curl: (7) Failed to connect to localhost port 7071: Connection refused
```

**Causes & Fixes:**

**1. Exporter not loaded**
```bash
# Check if -javaagent flag is present
ps aux | grep javaagent

# If missing, add to startup script:
java -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/jmx_exporter/config.yaml \
     -jar myapp.jar
```

**2. Wrong port**
```bash
# Check which port exporter is using
ss -tlnp | grep java
# Look for LISTEN on :7071

# If different port, update prometheus.yml scrape config
```

**3. Config file not found**
```bash
# Check application logs
journalctl -u myapp | grep jmx_exporter

# Common error:
# "Config file not found: /opt/jmx_exporter/config.yaml"

# Fix path or create missing file
```

**4. YAML syntax error**
```bash
# Validate YAML
yamllint /opt/jmx_exporter/config.yaml

# OR try loading in Python
python3 -c "import yaml; yaml.safe_load(open('/opt/jmx_exporter/config.yaml'))"

# Common issues:
# - Incorrect indentation
# - Missing quotes around regex patterns
# - Invalid characters
```

**5. JAR file corrupt**
```bash
# Verify JAR integrity
jar -tf /opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar | head

# If errors, re-download
cd /opt/jmx_exporter
sudo rm jmx_prometheus_javaagent-0.20.0.jar
sudo wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
```

---

### 11.3 Prometheus Target DOWN

**Symptom:**
```
Prometheus UI → Targets → State: DOWN (red)
```

**Diagnosis:**
```bash
# Check "Last Error" in Prometheus UI
# Common errors and fixes:

# Error: "dial tcp: lookup <hostname>: no such host"
# Fix: DNS issue - use IP instead of hostname in prometheus.yml

# Error: "dial tcp <ip>:7071: connect: connection refused"
# Fix: JMX Exporter not running (see 11.2)

# Error: "dial tcp <ip>:7071: i/o timeout"
# Fix: Firewall or network issue

# Error: "server returned HTTP status 404 Not Found"
# Fix: Wrong metrics_path in prometheus.yml (should be /metrics)

# Error: "context deadline exceeded"
# Fix: scrape_timeout too short or target too slow
#      Increase timeout in prometheus.yml
```

**Verify manually:**
```bash
# From Prometheus server, test scrape
curl -v http://<target-ip>:7071/metrics

# If this works but Prometheus still shows DOWN:
# 1. Check prometheus.yml syntax
promtool check config /etc/prometheus/prometheus.yml

# 2. Reload Prometheus
sudo systemctl reload prometheus

# 3. Check Prometheus logs
sudo journalctl -u prometheus | tail -50
```

---

### 11.4 Missing Metrics in Grafana

**Symptom:**
Dashboard panel shows "No data" but Prometheus has the metric.

**Diagnosis:**

**1. Query syntax error**
```
Grafana panel:
├─ Edit panel
├─ Query inspector → Query
├─ Check for typos:
│   ├─ Metric name (case-sensitive!)
│   ├─ Label names
│   └─ Label values
```

**2. Time range issue**
```
Grafana dashboard:
├─ Check time range (top right)
├─ Try "Last 5 minutes"
├─ If data appears, issue was time range
```

**3. Data source not selected**
```
Panel editor:
├─ Check "Data source" dropdown
└─ Should be "Prometheus"
```

**4. Metric dropped by Prometheus**
```bash
# Check if metric exists in Prometheus
curl -G http://localhost:9090/api/v1/label/__name__/values | jq . | grep jvm_memory

# If missing, check prometheus.yml metric_relabel_configs
# You may be dropping it accidentally
```

**5. Wrong aggregation**
```promql
# This might return no data if no results match:
avg(jvm_memory_bytes_used{instance="wrong-instance"})

# Try without aggregation first:
jvm_memory_bytes_used

# Then add filters one by one
```

---

### 11.5 Alert Not Firing

**Symptom:**
Condition is clearly true but alert stays INACTIVE.

**Diagnosis:**

**1. Check alert rule syntax**
```bash
# Validate rules file
promtool check rules /etc/prometheus/alert.rules.yml

# If errors, fix and reload Prometheus
```

**2. Test expression manually**
```
Prometheus UI:
├─ Graph tab
├─ Enter alert expression:
│   (jvm_memory_bytes_used{area="heap"} / jvm_memory_bytes_max{area="heap"}) > 0.8
├─ Execute
├─ Check if query returns results
│   └─ If empty, no instances match condition
```

**3. Check "for" duration**
```yaml
# Alert won't fire until condition true for this long
for: 2m

# If you just started test, wait full duration
# Monitor alert state: should go INACTIVE → PENDING → FIRING
```

**4. Check evaluation_interval**
```yaml
# In prometheus.yml
global:
  evaluation_interval: 15s

# If too long, alert evaluation is slow
# Reduce to 15s or less for faster response
```

**5. Alert rule not loaded**
```bash
# Check if rule file is listed in prometheus.yml
grep rule_files /etc/prometheus/prometheus.yml

# Expected:
# rule_files:
#   - '/etc/prometheus/alert.rules.yml'

# Verify Prometheus loaded it
curl -s http://localhost:9090/api/v1/rules | jq '.data.groups[].name'

# If your group name not listed, Prometheus didn't load the file
```

**6. Labels don't match**
```yaml
# Alert rule has:
expr: jvm_memory_bytes_used{job="myapp"} ...

# But your metrics have:
job="my-app"  # Note the hyphen

# Fix: Update alert rule or relabel in prometheus.yml
```

---

## 12. BEST PRACTICES

### 12.1 Metric Naming Conventions

Follow Prometheus naming standards:

**Format:** `<namespace>_<subsystem>_<unit>_<type>`

**Examples:**
```
✅ Good:
jvm_memory_bytes_used          # Clear unit (bytes)
jvm_gc_collection_seconds_total  # Clear unit (seconds), counter suffix
http_request_duration_seconds  # Standard naming

❌ Bad:
memory                         # Too vague, no unit
jvm_mem_mb                     # Abbreviations unclear
gc_time                        # Missing unit
```

**Unit suffixes:**
```
_bytes          → Size in bytes
_seconds        → Duration in seconds
_ratio          → Value 0.0-1.0
_percent        → Value 0-100
_total          → Counter (ever-increasing)
_count          → Gauge (can decrease)
```

**Type suffixes:**
```
_total          → Counter
_sum, _count    → Histogram/Summary components
(no suffix)     → Gauge
```

---

### 12.2 Label Usage

**DO:**
- ✅ Use labels for dimensions: `{instance, job, region, env}`
- ✅ Keep cardinality low (<100 unique values per label)
- ✅ Use consistent label names across metrics
- ✅ Label values should be bounded (not user IDs, request IDs)

**DON'T:**
- ❌ Don't use labels for high-cardinality data:
  ```
  # BAD: user_id can have millions of values
  http_requests_total{user_id="12345"}
  
  # GOOD: Use labels for bounded dimensions
  http_requests_total{endpoint="/api/users", method="GET"}
  ```
- ❌ Don't change label values over time for same metric
- ❌ Don't use labels for metric values (put value in metric itself)

**Example:**
```promql
# ❌ BAD: Temperature as label
server_status{temperature="72"}

# ✅ GOOD: Temperature as separate metric
server_temperature_celsius 72
```

---

### 12.3 Dashboard Design Tips

**1. Organize by audience:**
```
├─ Executive Dashboard
│   └─ High-level KPIs (availability, error rate)
├─ SRE Dashboard
│   └─ Detailed metrics (latency percentiles, saturation)
└─ Developer Dashboard
    └─ Application internals (heap, GC, threads)
```

**2. Use consistent colors:**
```
- Blue: Normal metrics (heap used, CPU)
- Green: Good thresholds, success rates
- Yellow: Warning thresholds
- Red: Critical thresholds, errors
```

**3. Panel types by use case:**
```
Gauge       → Current value with threshold (heap %, CPU %)
Time series → Trends over time (memory growth, request rate)
Stat        → Single big number (uptime, total requests)
Table       → Multi-instance comparison
Heatmap     → Latency distribution
```

**4. Set appropriate time ranges:**
```
Real-time monitoring:  Last 15 minutes, 5s refresh
Incident investigation: Last 1-6 hours
Capacity planning:     Last 30 days, no refresh
```

**5. Use variables for flexibility:**
```
Variables:
├─ $instance (select which server to view)
├─ $job (select which application)
└─ $interval (auto-adjust based on time range)

Query:
jvm_memory_bytes_used{instance=~"$instance", job="$job"}
```

---

### 12.4 Alert Tuning (Avoid Noise)

**1. Use "for" duration to avoid flapping:**
```yaml
# ❌ BAD: Alerts on every spike
- alert: HighHeap
  expr: jvm_memory_bytes_used > threshold
  for: 0s  # Fires immediately

# ✅ GOOD: Only alerts if sustained
- alert: HighHeap
  expr: jvm_memory_bytes_used > threshold
  for: 5m  # Must be true for 5 minutes
```

**2. Use rate() for counters:**
```yaml
# ❌ BAD: Raw counter (always increasing)
expr: jvm_gc_collection_count_total > 1000

# ✅ GOOD: Rate of change
expr: rate(jvm_gc_collection_count_total[5m]) > 5  # 5 GCs/sec
```

**3. Alert on symptoms, not causes:**
```yaml
# ❌ BAD: High heap may or may not be a problem
- alert: HighHeap
  expr: heap_used > 1GB

# ✅ GOOD: Alert on actual user impact
- alert: SlowResponse
  expr: http_request_duration_seconds > 1.0
```

**4. Group related alerts:**
```yaml
# Use Alertmanager grouping to batch related alerts
route:
  group_by: ['cluster', 'service']
  # All alerts from same service sent together
```

**5. Use severity levels:**
```yaml
Labels:
  severity: critical  → PagerDuty (wake up oncall)
  severity: warning   → Slack (review during business hours)
  severity: info      → Log only
```

---

### 12.5 Scaling Prometheus and Grafana

**Prometheus:**

**1. Retention tuning:**
```bash
# Balance storage vs. historical data
--storage.tsdb.retention.time=15d  # Default
--storage.tsdb.retention.size=50GB  # Auto-delete old blocks
```

**2. Scrape optimization:**
```yaml
# Reduce scrape frequency for less critical targets
scrape_configs:
  - job_name: 'critical-app'
    scrape_interval: 15s
  - job_name: 'batch-job'
    scrape_interval: 60s  # Less frequent
```

**3. Metric dropping:**
```yaml
# Drop unused metrics to save storage
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'jmx_scrape_.*'  # Internal exporter metrics
    action: drop
```

**4. Federation (multi-Prometheus):**
```yaml
# Aggregate metrics from multiple Prometheus servers
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 60s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="production"}'
    static_configs:
      - targets:
          - 'prom-1:9090'
          - 'prom-2:9090'
```

**Grafana:**

**1. Use datasource proxying:**
```
Access mode: Server (default)
# Grafana backend makes requests, not browser
# Better for security, caching
```

**2. Dashboard permissions:**
```
Folders:
├─ Production (view-only for most users)
├─ Development (edit access for devs)
└─ Personal (private dashboards)
```

**3. Playlist for NOC displays:**
```
Dashboards → Playlists → Create
├─ Add dashboards to rotate
├─ Interval: 30s per dashboard
└─ Display on wall-mounted screens
```

**4. Provisioning (Infrastructure as Code):**
```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: 'JVM Monitoring'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

---

### 12.6 Security Best Practices

**1. JMX Exporter:**
- ✅ Use Java Agent mode (no remote JMX needed)
- ✅ Bind HTTP to localhost if Prometheus is on same host
- ✅ Use firewall to restrict exporter port access

**2. Prometheus:**
- ✅ Enable authentication (basic auth via reverse proxy)
- ✅ Use HTTPS (via nginx reverse proxy)
- ✅ Restrict filesystem permissions (prometheus user only)

**3. Grafana:**
- ✅ Change default admin password immediately
- ✅ Enable HTTPS in grafana.ini
- ✅ Use LDAP/OAuth for user authentication
- ✅ Set up role-based access control (RBAC)

**4. Alertmanager:**
- ✅ Use secret management for webhook tokens (Vault, AWS Secrets Manager)
- ✅ Enable authentication for Alertmanager UI
- ✅ Encrypt sensitive annotations (don't include passwords in alerts)

---

### 12.7 Documentation

**Runbook links in alerts:**
```yaml
annotations:
  runbook: "https://wiki.example.com/runbooks/high-heap"
  # Link to step-by-step resolution guide
```

**Dashboard descriptions:**
```
Panel description:
"This shows JVM heap usage as percentage of max heap.
Threshold: >80% triggers warning, >90% triggers critical.
Expected range: 60-75% under normal load."
```

**Grafana variables documentation:**
```
Variable description:
$instance: Select which application server to view.
Use 'All' to see aggregate across all instances.
```

---

## SUMMARY

**Complete monitoring stack:**
```
JVM → JMX → JMX Exporter → Prometheus → Grafana + Alertmanager
     (built-in) (translator) (storage)   (visualization + alerting)
```

**Key files:**
- `/opt/jmx_exporter/config.yaml` - JMX metric translation rules
- `/etc/prometheus/prometheus.yml` - Scrape targets and global settings
- `/etc/prometheus/alert.rules.yml` - Alert definitions
- `/etc/alertmanager/alertmanager.yml` - Notification routing
- Grafana dashboards - Visualizations and dashboard-specific alerts

**Ports:**
- 7071 - JMX Exporter HTTP endpoint
- 9090 - Prometheus web UI and API
- 9093 - Alertmanager web UI
- 3000 - Grafana web UI

**Next steps:**
1. Set up monitoring for your first Java application
2. Import community Grafana dashboards (ID 4701, 8563)
3. Configure Alertmanager for your notification channels
4. Create runbooks for common alerts
5. Tune alert thresholds based on baseline metrics
6. Set up regular review of alert noise (weekly)

---

**End of Guide**
