# 08 — Prometheus + Grafana — Infrastructure Metrics and SOC Dashboards

## Overview

This guide deploys Prometheus and Grafana as a Docker Compose stack on the monitoring LXC (`10.10.10.50`). Node Exporter is installed on every SOC VM and container to expose host-level metrics. By the end of this guide:

* Prometheus is scraping CPU, memory, disk, and service health from all five SOC hosts
* Grafana is running and connected to Prometheus as a data source
* The Node Exporter Full dashboard is imported and showing live infrastructure telemetry
* Alert rules are configured for the conditions most likely to indicate a SOC failure

This observability layer is entirely separate from Wazuh. Wazuh handles security telemetry. Prometheus handles infrastructure health — a distinction that matters when you need to diagnose whether an alert gap is a detection failure or an infrastructure failure.

---

## 1. Before You Begin

| Requirement | Where to Check |
| ----------- | -------------- |
| Monitoring LXC running on Proxmox 2 | Proxmox 2 UI — container status |
| Docker Engine installed on monitoring LXC | `docker compose version` |
| All five SOC hosts reachable from monitoring LXC | `ping 10.10.10.10` etc. |
| Port 9100 open on each target host (or ufw not yet locked down) | After running `09-hardening.md`, port 9100 must be open from `10.10.10.50` |

---

## 2. Architecture

Prometheus scrapes metrics by pulling from exporters on each target host. No agents push data to Prometheus — the flow is pull-based.

```text
monitoring LXC (10.10.10.50)
├── Prometheus  — scrapes :9100 on each host every 15s
│   └── Alertmanager — fires on defined alert rules
└── Grafana     — reads from Prometheus, renders dashboards

Each target host:
└── Node Exporter — exposes /metrics on :9100
```

| Component | Port | Purpose |
| --------- | ---- | ------- |
| Prometheus | 9090 | Metrics database + query engine |
| Grafana | 3000 | Dashboard visualisation |
| Alertmanager | 9093 | Alert routing and deduplication |
| Node Exporter (per host) | 9100 | Host-level metrics endpoint |

---

## 3. Install Node Exporter on All Target Hosts

Node Exporter runs as a systemd service on each host. It does not run in Docker — this keeps the metrics endpoint independent of Docker's health.

Run the following on **each of the five SOC hosts** (Wazuh, MISP, n8n, IRIS, and the monitoring LXC itself):

```bash
# Download Node Exporter — check https://github.com/prometheus/node_exporter/releases for the latest version
NODE_EXPORTER_VERSION="1.8.2"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
tar -xzf node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz
sudo mv node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/node_exporter

# Create a dedicated system user
sudo useradd -r -s /bin/false node_exporter

# Create systemd service
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter

# Verify the metrics endpoint is up
curl -s http://localhost:9100/metrics | head -5
```

The output should show Prometheus metric lines beginning with `#`. If it returns a connection error, check `systemctl status node_exporter` for the failure reason.

> If you have already run the Ansible hardening playbook from `07-ansible.md`, add a ufw rule for Node Exporter before running the commands above:
> ```bash
> sudo ufw allow from 10.10.10.50 to any port 9100 proto tcp
> ```

---

## 4. Prometheus + Grafana — Docker Compose Stack

SSH into the monitoring LXC (`10.10.10.50`).

Create the working directory:

```bash
sudo mkdir -p /opt/monitoring/{prometheus,grafana}
sudo chown -R $USER:$USER /opt/monitoring
```

### 4.1 Prometheus Configuration

Create `/opt/monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    lab: soc-bridge-between

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    static_configs:
      - targets:
          - 10.10.10.10:9100   # wazuh
          - 10.10.10.20:9100   # misp
          - 10.10.10.30:9100   # n8n
          - 10.10.10.40:9100   # iris
          - 10.10.10.50:9100   # monitoring (self)
        labels:
          environment: soc
    relabel_configs:
      - source_labels: [__address__]
        regex: '(10\.10\.10\.\d+):\d+'
        target_label: instance
        replacement: '${1}'
```

### 4.2 Alert Rules

Create `/opt/monitoring/prometheus/rules/soc-alerts.yml`:

```yaml
groups:
  - name: soc-infrastructure
    interval: 30s
    rules:
      - alert: HostDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "SOC host {{ $labels.instance }} is unreachable"
          description: "Prometheus cannot scrape {{ $labels.instance }}. Node Exporter may be down or the host is offline."

      - alert: HighCPU
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage has been above 90% for 10 minutes on {{ $labels.instance }}."

      - alert: LowMemory
        expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Available memory is below 10% on {{ $labels.instance }}. Wazuh Indexer OOM is the most likely cause."

      - alert: LowDisk
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk on {{ $labels.instance }}"
          description: "Root filesystem is below 15% free on {{ $labels.instance }}."

      - alert: ServiceDown
        expr: node_systemd_unit_state{state="active"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Systemd unit {{ $labels.name }} is not active on {{ $labels.instance }}"
```

### 4.3 Alertmanager Configuration

Create `/opt/monitoring/prometheus/alertmanager.yml`:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: default
    # Configure a webhook, email, or Slack receiver here.
    # Example for n8n webhook (route SOC alerts back through your SOAR):
    # webhook_configs:
    #   - url: 'http://10.10.10.30:5678/webhook/soc-infra-alerts'
    #     send_resolved: true
```

Alertmanager ships with no receivers configured by default — it will deduplicate and group alerts but not dispatch them until you add a receiver. The n8n webhook option above routes infrastructure alerts back through the SOAR pipeline, which keeps all incident data in DFIR IRIS.

### 4.4 Docker Compose File

Create `/opt/monitoring/docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./prometheus/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=changeme
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://10.10.10.50:3000
      - GF_ALERTING_ENABLED=true
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
```

> Change `GF_SECURITY_ADMIN_PASSWORD` before starting the stack. The default `changeme` is a placeholder.

Start the stack:

```bash
cd /opt/monitoring
docker compose up -d
docker compose ps
```

All three containers (`prometheus`, `alertmanager`, `grafana`) must show status `Up`.

---

## 5. Configure Grafana

### 5.1 Add the Prometheus Data Source

Open Grafana at `http://10.10.10.50:3000`. Log in with `admin` and the password you set in the Compose file.

Navigate to **Connections > Data sources > Add data source > Prometheus**.

| Field | Value |
| ----- | ----- |
| Name | Prometheus |
| URL | `http://prometheus:9090` |
| Scrape interval | 15s |

Click **Save & test**. The result must be `Data source is working` — if it fails, confirm the `prometheus` container is running and the URL uses the container name, not an IP.

### 5.2 Import Dashboards

Grafana has a public dashboard library. Import the following by ID from **Dashboards > Import**:

| Dashboard | ID | Purpose |
| --------- | -- | ------- |
| Node Exporter Full | 1860 | Full host-level telemetry (CPU, memory, disk, network, load) |
| Node Exporter Quickstart | 13978 | Compact single-host overview |
| Prometheus Stats | 2 | Prometheus self-monitoring |

For each: click **Import**, enter the ID, select the `Prometheus` data source, and click **Import** again.

The Node Exporter Full dashboard (1860) will immediately show all five SOC hosts in the `instance` dropdown if Node Exporter was installed correctly on each.

### 5.3 Create a SOC Overview Dashboard

Create a new dashboard with a row for each host. Useful panels:

| Panel Type | Metric | Purpose |
| ---------- | ------ | ------- |
| Stat | `up{job="node"}` | Quick green/red host status |
| Time series | `rate(node_cpu_seconds_total{mode!="idle"}[5m])` | CPU usage over time |
| Gauge | `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100` | Memory available % |
| Gauge | `node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100` | Disk free % |
| Stat | `node_load1` | 1-minute load average |

Set the Wazuh host panel to display a warning annotation when memory drops below 20% — the Wazuh Indexer's OOM pattern starts well before it kills the process.

---

## 6. Verify Scraping

Check that Prometheus is receiving metrics from all targets:

1. Open `http://10.10.10.50:9090/targets`
2. All five `node` targets must show **State: UP** with a recent **Last Scrape** timestamp
3. Any target showing `DOWN` or `UNKNOWN` indicates Node Exporter is not reachable — check the firewall rules and Node Exporter service status on that host

Run a test query in Prometheus to confirm data is present:

```promql
up{job="node"}
```

Should return five results, all with value `1`.

---

## 7. Verification Summary

| Check | Expected Result |
| ----- | --------------- |
| `docker compose ps` on monitoring LXC | prometheus, alertmanager, grafana all `Up` |
| `http://10.10.10.50:9090/targets` | All five node targets `UP` |
| `http://10.10.10.50:9090/alerts` | Alert rules loaded (state pending or inactive if thresholds not met) |
| Grafana data source test | `Data source is working` |
| Node Exporter Full dashboard | All five hosts visible in instance dropdown |
| `curl http://10.10.10.10:9100/metrics` from monitoring LXC | Prometheus metric lines returned |

---

## 8. Final Checklist

- [ ] Node Exporter installed and running on all five SOC hosts
- [ ] Node Exporter responding on `:9100` — `curl http://<ip>:9100/metrics` returns metric lines
- [ ] ufw rule allowing monitoring LXC (`10.10.10.50`) to reach `:9100` on each target
- [ ] `/opt/monitoring/docker-compose.yml` created
- [ ] Grafana admin password changed from `changeme`
- [ ] `docker compose up -d` run — all three containers running
- [ ] Prometheus targets page shows all five hosts as `UP`
- [ ] Alert rules loaded — visible at `/alerts`
- [ ] Alertmanager reachable at `:9093`
- [ ] Grafana data source `Prometheus` added and test passes
- [ ] Node Exporter Full dashboard (ID 1860) imported
- [ ] All five hosts visible in Grafana dashboard instance dropdown
- [ ] Snapshot taken of monitoring LXC after confirming the stack is healthy

---
