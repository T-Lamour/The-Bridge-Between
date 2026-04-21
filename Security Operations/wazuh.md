<p align="center">
  <img src="Images/Wazuh/logo.png" width="420"/>
</p>

## Overview

Wazuh is the core **SIEM** platform used in this SOC. It is responsible for collecting, analysing, and generating alerts from logs across endpoints and network devices.

Within this project, Wazuh acts as the **primary detection engine**, identifying suspicious or malicious activity in real time.

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

Wazuh sits at the centre of the SOC pipeline:

```id="wazuh-flow"
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

## Log Collection

Wazuh agents are deployed on monitored systems to forward logs to the Wazuh server.

<p align="center">
  <img src="Images/Wazuh/Wazuh discover.png" width="620"/>
</p>

### Data Sources Include:

* System logs (Linux / Windows)
* Authentication events (e.g. logins, failures)
* Application logs
* Network/security device logs

---

## Detection & Rules

Wazuh uses a combination of:

* Built-in detection rules
* Custom rules tailored to specific use cases

### Examples of detections:

* Brute force login attempts
* Successful logins from suspicious IP addresses
* Privilege escalation activity
* Malware indicators

Custom rules can be created to align with:

* Specific client environments
* Known threat patterns
* Threat intelligence feeds

---

## Integration with SOC Workflow

Wazuh integrates with other components as follows:

* **n8n** → receives alerts for enrichment and automation
* **DFIR IRIS** → receives incidents created from enriched alerts
* **MISP** → used for threat intelligence correlation (via workflows)

This enables a fully automated detection-to-response pipeline.

---

## Key Benefits

* Centralised visibility across all monitored systems
* Real-time threat detection
* Flexible and customisable rule engine
* Seamless integration with automation tools
* Open-source and cost-effective

---

## Limitations

* Requires tuning to reduce false positives
* Rule management can become complex at scale
* Advanced use cases may require custom development

---

## Summary

Wazuh forms the foundation of the SOC by providing:

* Visibility into system and network activity
* Detection of security threats
* Structured alerts for automated response

It enables the transition from raw log data to actionable security events.

---
