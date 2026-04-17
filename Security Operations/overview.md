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
* **pfSense** – Network firewall and traffic control

Each tool plays a specific role in the detection and response lifecycle.

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

     * Threat intelligence (e.g. MISP, VirusTotal, AbuseIPDB)
     * Contextual data (IP reputation, geolocation, etc.)

4. **Incident Creation**

   * Enriched alerts are forwarded to DFIR IRIS
   * Incidents are created and tracked for investigation

5. **Response Actions**

   * Automated or manual responses may include:

     * Disabling user accounts
     * Revoking sessions (Microsoft Entra)
     * Blocking IPs via pfSense
     * Updating threat intelligence in MISP

---

## Data Flow Summary

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
        ▼
    DFIR IRIS
 (Case Management)
        │
        ▼
 Response Actions
(pfSense / Entra / Defender)
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

## Next Steps

For detailed configuration and implementation, see:

* `wazuh.md`
* `n8n.md`
* `dfir-iris.md`
* `misp.md`
* `pfsense.md`

---
