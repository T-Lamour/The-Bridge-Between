# ISO/IEC 27001 Mapping

## Overview

ISO/IEC 27001 is an international standard for establishing, implementing, maintaining, and continuously improving an Information Security Management System (ISMS).

In this project, ISO 27001 is used as a **design and mapping framework**, not as a formal certification. It helps demonstrate how the SOC architecture aligns with recognised security controls and best practices.

---

## Purpose in This Project

The purpose of mapping ISO 27001 to this SOC environment is to:

* Demonstrate security control alignment
* Show structured risk-based thinking
* Validate that the SOC design follows industry standards
* Provide a compliance-aware security architecture

---

## High-Level ISMS Alignment

This SOC simulates key ISO 27001 principles:

* **Confidentiality** → Controlled access to systems and logs
* **Integrity** → Log protection and alert validation via SIEM
* **Availability** → Resilient monitoring and automation workflows
* **Accountability** → Centralised logging and case management

---

## Control Mapping to SOC Components

### 1. Logging & Monitoring (Annex A.8 / A.12)

**Implemented via:**

* Wazuh SIEM
* Suricata (pfSense IDS)
* System and endpoint log collection

**Function:**

* Continuous monitoring of security events
* Centralised log aggregation
* Detection of suspicious activity

---

### 2. Access Control (Annex A.5 / A.9)

**Implemented via:**

* pfSense firewall rules
* System-level user permissions
* (Planned) Identity response via Microsoft Entra

**Function:**

* Restricting network and system access
* Enforcing least privilege principles
* Preventing unauthorised access paths

---

### 3. Incident Management (Annex A.5.24 / A.5.25)

**Implemented via:**

* DFIR IRIS case management system
* n8n incident creation workflows

**Function:**

* Structured incident tracking
* Defined investigation workflow
* Documentation of response actions

---

### 4. Threat Intelligence (Annex A.5.7)

**Implemented via:**

* MISP (IOC database)
* VirusTotal / AbuseIPDB integrations (via n8n)

**Function:**

* Enrichment of security alerts
* Correlation of indicators of compromise
* Improved detection accuracy

---

### 5. Network Security (Annex A.8 / A.13)

**Implemented via:**

* pfSense firewall
* Suricata IDS/IPS

**Function:**

* Network segmentation and traffic control
* Detection of malicious network activity
* Blocking of known threats

---

### 6. Security Automation (Operational Control Support)

**Implemented via:**

* n8n workflows

**Function:**

* Automated alert enrichment
* Incident creation in IRIS
* Triggering of response actions (blocking, escalation, logging)

---

## SOC Architecture as a Control System

The SOC architecture directly supports ISO 27001 objectives:

```text id="iso-flow"
Detection → Wazuh + Suricata
     │
Enrichment → MISP + Threat Intel APIs
     │
Automation → n8n
     │
Incident Management → DFIR IRIS
     │
Network Enforcement → pfSense
```

---

## Risk-Based Design Approach

This SOC is designed around common security risks:

| Risk                  | Control Implementation     |
| --------------------- | -------------------------- |
| Credential compromise | Login detection via Wazuh  |
| Malware execution     | File / endpoint monitoring |
| Network intrusion     | Suricata IDS               |
| Lateral movement      | Log correlation + alerting |
| Data exposure         | Firewall segmentation      |

---

## Continuous Improvement (ISO Principle)

The architecture supports continuous improvement through:

* Feedback loops via incident investigations
* IOC enrichment added back into MISP
* Refinement of Wazuh detection rules
* Automation workflow updates in n8n

This mirrors the ISO 27001 requirement for ongoing system improvement.

---

## Limitations

This implementation is:

* A **conceptual mapping**, not a certified ISMS
* Focused on technical controls rather than full governance
* Missing formal policy and audit documentation layers

---

## Summary

This SOC environment demonstrates alignment with ISO/IEC 27001 by implementing core technical controls across detection, response, access control, and threat intelligence.

While not formally certified, the architecture reflects key principles of an Information Security Management System (ISMS) through a practical, automated security operations workflow.

---
