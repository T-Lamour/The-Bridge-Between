# SOC Operations Overview

## Purpose

This section outlines how the Security Operations Centre (SOC) within this project functions. It demonstrates how security events are collected, enriched, analysed, and responded to using an automated and scalable pipeline.

The goal is to simulate a lightweight but effective SOC capability suitable for small businesses or individuals, using open-source and cost-efficient tooling.

---

## Core Components

The SOC is built around the following tools:

* **Wazuh** – Security Information and Event Management (SIEM)
* **n8n** – Workflow automation and orchestration (SOAR-like capabilities)
* **DFIR IRIS** – Incident response and case management
* **MISP** – Threat intelligence platform
* **OPNsense** – Network firewall and traffic control
* **Ansible** – Agentless configuration management and IaC
* **Prometheus** – Infrastructure metrics collection
* **Grafana** – Dashboard and visualisation layer

Each tool plays a specific role in the detection and response lifecycle.

---

## Infrastructure

The SOC runs across two Proxmox hosts and a bare-metal OPNsense firewall. Heavier workloads (Wazuh, MISP) run as full VMs on Proxmox 1. Lighter services (n8n, IRIS, Grafana/Prometheus, Ansible) run as LXC containers on Proxmox 2. All tools are deployed as Docker Compose stacks.

| Component | Host | Type |
|-----------|------|------|
| Wazuh | Proxmox 1 | VM (Ubuntu 22.04) |
| MISP | Proxmox 1 | VM (Ubuntu 22.04) |
| n8n | Proxmox 2 | LXC (Ubuntu 22.04) |
| DFIR IRIS | Proxmox 2 | LXC (Ubuntu 22.04) |
| Grafana + Prometheus | Proxmox 2 | LXC (Ubuntu 22.04) |
| Ansible | Proxmox 2 | LXC (Ubuntu 22.04) |
| OPNsense | Bare metal | Native install |

---

## Monitored Endpoints

Wazuh agents are deployed across the following systems:

* Windows 10 (workstation)
* Windows Server
* Linux systems

OPNsense forwards Suricata alerts and firewall events to Wazuh via syslog (UDP 514), extending log collection to the network layer.

---

## High-Level Workflow

The SOC operates using the following pipeline:

1. **Log Collection**

   * Endpoints and network devices generate logs
   * Logs are forwarded to Wazuh for ingestion and analysis

2. **Detection & Alerting**

   * Wazuh applies rules and decoders to identify suspicious activity
   * Alerts are generated based on predefined detection logic

3. **Automation & Enrichment**

   * Alerts are sent to n8n via webhook
   * n8n enriches data using:

     * Threat intelligence (MISP, VirusTotal, AbuseIPDB)
     * Contextual data (IP reputation, geolocation, etc.)

4. **Incident Creation**

   * Enriched alerts are forwarded to DFIR IRIS
   * Incidents are created and tracked for investigation

5. **Response Actions**

   * Automated or manual responses may include:

     * Disabling user accounts
     * Revoking sessions (Microsoft Entra)
     * Blocking IPs via OPNsense
     * Updating threat intelligence in MISP

---

## Data Flow

```
Endpoint / Network Device
        │
        ▼
     Wazuh SIEM
 (Detection & Alerting)
        │
        ▼
        n8n
 (Enrichment & Automation)
        │
        ├──────────► MISP (IOC Lookup)
        │
        ├──────────► DFIR IRIS (Incident Creation)
        │
        ▼
 Response Actions
(OPNsense / Entra / Defender)
```

---

## Key Capabilities

* Centralised log collection and monitoring
* Real-time alerting based on detection rules
* Automated enrichment using threat intelligence
* Incident tracking and case management
* Automated response to reduce mean time to respond (MTTR)

---

## Design Principles

* **Automation First** – Reduce manual effort through workflows
* **Modular Architecture** – Each component can be replaced or scaled independently
* **Cost Efficiency** – Designed for small environments without enterprise budgets
* **Real-World Applicability** – Simulates workflows used in modern SOC environments

---

## Component Documentation

For detailed configuration and implementation, see:

* [wazuh.md](wazuh.md) – SIEM, log collection, and detection rules
* [n8n.md](n8n.md) – Automation workflows and SOAR capabilities
* [iris.md](iris.md) – Incident response and case management
* [misp.md](misp.md) – Threat intelligence and IOC management
* [opnsense.md](opnsense.md) – Firewall and network control
* [alternative-tools.md](alternative-tools.md) – Alternative open-source tools for each layer

---
