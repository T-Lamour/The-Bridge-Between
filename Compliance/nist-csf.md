# NIST Cybersecurity Framework (CSF) Mapping

## Overview

The NIST Cybersecurity Framework (CSF) provides a structured approach for managing cybersecurity risk across five core functions:

* Identify
* Protect
* Detect
* Respond
* Recover

This document maps the SOC architecture in this project to the NIST CSF framework to demonstrate how open-source tools can implement a complete cybersecurity lifecycle.

This mapping is intended for **educational and portfolio purposes** and does not represent formal organisational compliance.

---

## SOC Architecture Summary

The SOC environment is built using the following components:

* Wazuh (SIEM / Detection)
* n8n (Automation / SOAR)
* DFIR IRIS (Incident Management)
* MISP (Threat Intelligence)
* pfSense + Suricata (Network Security)

These tools collectively implement end-to-end security monitoring and response capabilities.

---

## NIST CSF Implementation Overview

| Function | Description                               | SOC Implementation                                               |
| -------- | ----------------------------------------- | ---------------------------------------------------------------- |
| Identify | Understand assets, risks, and environment | Inventory of systems via Wazuh agents and VM environment mapping |
| Protect  | Safeguard systems and data                | pfSense firewall rules, Suricata IDS, system hardening           |
| Detect   | Identify cybersecurity events             | Wazuh SIEM rules, Suricata network alerts                        |
| Respond  | Take action on detected incidents         | n8n automation workflows, DFIR IRIS incident handling            |
| Recover  | Restore services and improve resilience   | Incident review, rule tuning, IOC enrichment via MISP            |

---

## 1. Identify (ID)

This function focuses on understanding the environment and associated risks.

### Implementation in SOC:

* Asset visibility through Wazuh agent deployment
* Network topology defined in VMware lab environment
* System roles documented across GitHub architecture section

### Outcome:

Provides foundational awareness of all monitored systems and their security relevance.

---

## 2. Protect (PR)

This function focuses on implementing safeguards to ensure service continuity and security.

### Implementation in SOC:

* pfSense firewall enforcing network segmentation
* Suricata IDS detecting malicious traffic patterns
* Host-level logging and integrity monitoring via Wazuh agents

### Outcome:

Reduces attack surface and limits unauthorised access to systems.

---

## 3. Detect (DE)

This function focuses on identifying security events and anomalies.

### Implementation in SOC:

* Wazuh SIEM rule-based detection engine
* Log aggregation from endpoints and systems
* Suricata network-based intrusion detection
* Alert generation for suspicious authentication or behaviour

### Outcome:

Enables early detection of malicious or abnormal activity.

---

## 4. Respond (RS)

This function focuses on containment and mitigation of security incidents.

### Implementation in SOC:

* n8n automation workflows for alert enrichment and decision-making
* DFIR IRIS for structured incident management
* Threat intelligence enrichment via MISP, VirusTotal, AbuseIPDB
* Potential automated response actions (e.g. IP blocking, account actions)

### Outcome:

Reduces time to respond and standardises incident handling.

---

## 5. Recover (RC)

This function focuses on restoring normal operations and improving resilience.

### Implementation in SOC:

* Incident review and post-analysis in DFIR IRIS
* Refinement of Wazuh detection rules based on findings
* Updates to MISP IOC database for future detection
* Workflow improvements in n8n automation pipelines

### Outcome:

Improves system resilience and detection quality over time.

---

## SOC Data Flow Mapping

```text id="nist-flow"
Protect Layer → pfSense + Suricata
        │
Detect Layer → Wazuh + Suricata Alerts
        │
Respond Layer → n8n + DFIR IRIS
        │
Threat Intelligence → MISP (enrichment across all stages)
        │
Recover → Rule tuning + IOC feedback loop
```

---

## Security Capabilities Demonstrated

This SOC implementation demonstrates:

* Continuous monitoring of system and network activity
* Automated detection and enrichment of security alerts
* Structured incident response workflows
* Threat intelligence-driven decision making
* Feedback loops for continuous improvement

---

## Key Strength of This Architecture

Unlike isolated security tools, this SOC demonstrates a **full lifecycle security model** aligned with NIST CSF:

* Detection is not isolated (Wazuh + Suricata)
* Response is automated (n8n + IRIS)
* Intelligence is reusable (MISP feedback loop)
* Recovery improves detection over time

---

## Limitations

This implementation is a **technical simulation of NIST CSF**, and does not include:

* Formal governance structures
* Organisational risk registers
* Business continuity planning
* Legal/compliance enforcement layers

The focus is on **technical SOC operations and automation workflows**.

---

## Summary

The SOC architecture in this project aligns strongly with the NIST Cybersecurity Framework by implementing all five core functions through open-source tools.

This demonstrates a complete security lifecycle:

**Identify → Protect → Detect → Respond → Recover**

resulting in a scalable, automated, and threat-informed SOC design.

---
