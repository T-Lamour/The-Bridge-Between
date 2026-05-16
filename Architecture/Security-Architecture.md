# Architecture Overview

A breakdown of the tools used in this project — what they do, why they were chosen, and how they fit together.

---

## Data Flow

The system processes security events through two parallel pipelines:

**Detection → Response pipeline**

1. Endpoints generate logs and security events
2. Wazuh ingests, analyses, and generates alerts
3. Alerts are forwarded to n8n via webhook
4. n8n enriches alerts using threat intelligence APIs (AbuseIPDB, OTX AlienVault, abuse.ch, URLScan.io)
5. n8n checks and updates MISP for IOC correlation
6. Enriched alerts are converted into structured cases in DFIR IRIS
7. n8n executes automated response actions (IP block via OPNsense API, account disable via Entra ID)
8. Analysts investigate, triage, and close cases in DFIR IRIS

**Infrastructure observability pipeline**

1. Prometheus scrapes metrics from every VM and LXC container (CPU, memory, disk, service health)
2. Grafana queries Prometheus and renders dashboards
3. Analysts monitor infrastructure health and SOC tool availability alongside security telemetry

---

## Wazuh — SIEM / XDR Layer

**Role:** Centralised log collection, detection, and alerting engine. Deployed as a Docker Compose stack (Indexer + Manager + Dashboard) on a dedicated Proxmox 1 VM.

**Key Capabilities:**

- Log ingestion from endpoints via lightweight agents
- Rule-based detection (Wazuh ruleset + custom rules)
- File integrity monitoring (FIM)
- Vulnerability detection
- Security Configuration Assessment (SCA)
- Syslog receiver for OPNsense / Suricata network events

**Function in Architecture:**

- Primary detection engine for the entire SOC
- Normalises and correlates log data from agents and syslog sources
- Generates enriched alerts forwarded to n8n via webhook
- Hosts the SOC analyst dashboard

> **Resource note:** The Wazuh Indexer (OpenSearch) requires a minimum of 16 GB RAM to function reliably under real alert volume. Below this it silently OOM-kills — the dashboard loads but shows no alerts.

---

## n8n — SOAR / Automation Layer

**Role:** Workflow automation and orchestration engine. Deployed as a Docker Compose stack on a Proxmox 2 LXC container.

**Key Capabilities:**

- Webhook ingestion from Wazuh
- Integration with threat intelligence APIs (AbuseIPDB, OTX AlienVault, abuse.ch MalwareBazaar, abuse.ch URLhaus, abuse.ch ThreatFox, URLScan.io)
- IOC lookup and ingestion against MISP
- Conditional branching and severity-based routing
- Automated response actions via external APIs
- Built-in encrypted credential store — no API keys are stored in workflow JSON or environment files

**Function in Architecture:**

- Orchestration layer that connects every other tool
- Enriches alerts with multi-source threat intelligence before any case is opened
- Automates triage: IP blocking (OPNsense), account disable (Entra ID), session revocation
- Creates structured cases in DFIR IRIS with full enrichment context attached

---

## DFIR IRIS — Incident Response Platform

**Role:** Case management and investigation platform. Deployed as a Docker Compose stack on a Proxmox 2 LXC container.

**Key Capabilities:**

- Structured incident creation and tracking
- Asset, IOC, and evidence management
- Timeline analysis
- Analyst collaboration and case notes

**Function in Architecture:**

- Incident management layer — every enriched alert lands here as a structured case
- Stores the full investigation record from triage through closure
- Provides the analyst workflow for investigation and response

---

## MISP — Threat Intelligence Platform

**Role:** Internal IOC database and threat intelligence hub. Deployed as a Docker Compose stack on a Proxmox 1 VM.

**Key Capabilities:**

- IOC storage (IPs, domains, hashes, URLs)
- External threat feed ingestion
- Event correlation across incidents
- REST API for programmatic lookups and ingestion

**Function in Architecture:**

- Internal threat intelligence store
- n8n checks MISP during enrichment and writes confirmed IOCs back after incident closure
- Builds institutional memory — every confirmed indicator improves future detections

---

## OPNsense — Network Security Layer

**Role:** Perimeter firewall, network segmentation, and IDS. Runs bare metal — direct NIC access is required for a firewall appliance.

**Key Capabilities:**

- Traffic filtering and NAT
- VLAN segmentation (VLAN 10 Management, VLAN 20 SOC, VLAN 30 Victim)
- Default-deny inter-VLAN policy with explicit allow rules
- Suricata IDS — real-time deep packet inspection with ET Open signatures
- API-driven automated IP blocking
- Syslog forwarding of network events and Suricata alerts to Wazuh

**Function in Architecture:**

- Network-level security control and segmentation layer
- Source of network visibility — all OPNsense and Suricata events flow into Wazuh
- Automated enforcement point — n8n pushes block rules via the OPNsense API on confirmed threats

---

## Ansible — Configuration Management / IaC Layer

**Role:** Agentless configuration management and infrastructure as code. Runs from a Proxmox 2 LXC container (or management laptop during active development).

**Key Capabilities:**

- SSH-based agentless execution — no agent installed on managed hosts
- Idempotent playbooks for provisioning, patching, and hardening
- Inventory-driven targeting of VMs and LXC containers
- Version-controlled infrastructure state

**Function in Architecture:**

- Keeps all VMs and containers in a known, reproducible configuration state
- Handles initial provisioning (Docker install, user setup, ufw rules) and ongoing patching
- Hardens the SOC stack to a consistent baseline — CIS-aligned where applicable
- Provides a repeatable rollback path alongside Proxmox snapshot automation

---

## Prometheus — Metrics Collection Layer

**Role:** Pull-based time-series metrics database. Deployed alongside Grafana as a Docker Compose stack on a Proxmox 2 LXC container.

**Key Capabilities:**

- Pull-based scraping of node exporters on every VM and LXC container
- Time-series storage of CPU, memory, disk I/O, network, and service health metrics
- Alerting rules for threshold breaches (high disk usage, service down, memory pressure)

**Function in Architecture:**

- Collects infrastructure telemetry from every host in the lab
- Data source for Grafana dashboards
- Provides early warning of resource exhaustion before it impacts SOC tool availability

---

## Grafana — Dashboards / Visualisation Layer

**Role:** Dashboard and visualisation platform. Deployed alongside Prometheus as a Docker Compose stack on a Proxmox 2 LXC container.

**Key Capabilities:**

- Pre-built and custom dashboards fed from Prometheus
- Infrastructure health panels: CPU, memory, disk, uptime per host
- SOC service availability monitoring
- Alert notification routing (email, Slack, webhook)

**Function in Architecture:**

- Single-pane visibility across the entire SOC infrastructure
- Surfaces resource issues before they cause silent tool failures
- Complements Wazuh's security dashboards with infrastructure health context

---

## Integration Points

| Source | Destination | Method | Purpose |
| ------ | ----------- | ------ | ------- |
| Wazuh | n8n | Webhook | Forward alerts for enrichment and orchestration |
| OPNsense / Suricata | Wazuh | Syslog UDP 514 | Network and IDS event visibility |
| n8n | AbuseIPDB | REST API | IP reputation scoring |
| n8n | OTX AlienVault | REST API | Broad IOC enrichment (IP, domain, hash, URL) |
| n8n | abuse.ch MalwareBazaar | REST API | File hash lookup on FIM alerts |
| n8n | abuse.ch URLhaus | REST API | Malicious URL / domain lookup |
| n8n | abuse.ch ThreatFox | REST API | General IOC lookup |
| n8n | URLScan.io | REST API | URL sandbox scan (optional) |
| n8n | MISP | REST API | IOC correlation lookup and ingestion |
| n8n | DFIR IRIS | REST API | Structured case creation with enrichment context |
| n8n | OPNsense | REST API | Automated IP block on confirmed threat |
| n8n | Entra ID / M365 | REST API | Account disable and session revocation |
| Prometheus | All VMs + LXCs | HTTP scrape (node exporter) | Infrastructure metrics collection |
| Grafana | Prometheus | PromQL | Dashboard queries and visualisation |
| Ansible | All VMs + LXCs | SSH | Provisioning, patching, and hardening |
