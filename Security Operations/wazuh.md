# Wazuh

<p align="center">
  <img src="Images/Wazuh/logo.png" width="420"/>
</p>

## Overview

Wazuh is the core **SIEM** platform used in this SOC. It is responsible for collecting, analysing, and generating alerts from logs across endpoints and network devices.

Within this project, Wazuh acts as the **primary detection engine**, identifying suspicious or malicious activity in real time and forwarding structured alerts downstream for enrichment and response.

<p align="center">
  <img src="Images/Wazuh/Wazuh overview.png" width="420"/>
</p>

---

## Key Responsibilities

* Log collection from endpoints and systems
* Event correlation and rule-based detection
* File integrity monitoring (FIM)
* Vulnerability detection
* Security alert generation

---

## Architecture Role

Wazuh sits at the start of the SOC pipeline, ingesting data from all monitored systems:

```
Endpoints / Devices
        │
        ▼
     Wazuh SIEM
 (Log Ingestion & Detection)
        │
        ▼
        n8n
 (Automation & Enrichment)
```

All incoming security data flows through Wazuh before being processed further.

---

## Infrastructure

Wazuh runs on a dedicated **VMware Workstation VM**, deployed using **Docker**. The deployment includes:

* **Wazuh Manager** – receives and processes logs from all agents
* **Wazuh Indexer** – stores and indexes alert and log data
* **Wazuh Dashboard** – web interface for log analysis, rule management, and alerting

---

## Monitored Endpoints

Wazuh agents are deployed across the following systems:

| Endpoint | OS | Agent Status |
|----------|----|--------------|
| Workstation | Windows 10 | Active |
| Server | Windows Server | Active |
| Lab System | Linux | Active |
| Firewall | OPNsense | Planned |

OPNsense integration is planned to extend log collection to network-level events, including firewall rule matches and blocked connection attempts.

---

## Log Collection

Wazuh agents are deployed on monitored systems to forward logs to the Wazuh Manager in real time.

<p align="center">
  <img src="Images/Wazuh/Wazuh discover.png" width="620"/>
</p>

### Data Sources:

* **Windows 10 / Windows Server** – Security event logs, authentication events, process creation (Sysmon)
* **Linux** – Syslog, auth.log, audit logs
* **Application logs** – Web servers and services where applicable
* **Network events** – Planned via OPNsense agent integration

---

## Detection & Rules

Wazuh uses a combination of built-in and custom detection rules to identify threats across all monitored systems.

### Active Detection Examples:

* Brute force login attempts (SSH, RDP, Windows logon)
* Successful logins from suspicious IP addresses
* Privilege escalation activity
* Malware indicators and known bad hashes
* File integrity changes on monitored paths
* New local user account creation
* Lateral movement patterns

### Custom Rule Development

Custom rules have been written to detect environment-specific threats, including:

* Repeated failed logins followed by a success (credential stuffing pattern)
* Logins occurring outside of expected hours
* Unusual process execution on Windows endpoints

Rules are written in Wazuh's XML-based rule format and stored alongside agent configuration in this project.

---

## File Integrity Monitoring

FIM is configured on critical directories across all monitored endpoints:

* **Windows** – System32, user profile directories, startup locations
* **Linux** – `/etc`, `/bin`, `/usr/bin`, home directories

Any unauthorised modification triggers an alert that is forwarded through the pipeline for enrichment and response.

---

## Alert Forwarding to n8n

When Wazuh generates an alert, it is forwarded to **n8n** via a webhook integration. The alert payload includes:

* Rule ID and description
* Severity level (Wazuh rule level, 1–15)
* Source IP address and agent hostname
* Timestamp and raw log data

This triggers the enrichment and automated response workflow in n8n.

---

## Integration with SOC Workflow

* **n8n** → receives alerts via webhook for enrichment and automated decision-making
* **DFIR IRIS** → receives structured incidents created from enriched alerts
* **MISP** → IOC correlation during the enrichment phase (handled via n8n)

---

## Key Benefits

* Centralised visibility across all monitored systems
* Real-time threat detection with low latency
* Flexible and customisable rule engine
* Seamless integration with automation tools
* Open-source and cost-effective

---

## Limitations

* Requires rule tuning to reduce false positives in the environment
* Rule management can become complex at scale
* Advanced use cases may require custom rule or decoder development

---

## Summary

Wazuh forms the foundation of the SOC by providing:

* Visibility into system and network activity across Windows 10, Windows Server, and Linux endpoints
* Detection of security threats using built-in and custom rules
* Structured alert payloads forwarded to n8n for automated enrichment and response

It enables the transition from raw log data to actionable security events.

---
