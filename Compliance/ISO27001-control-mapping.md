# ISO 27001 Compliance Mapping

## Overview

This document outlines how the lab aligns with selected controls from **ISO/IEC 27001:2022**.

The purpose of this mapping is to demonstrate how commonly used open-source security tools can support security controls aligned with ISO 27001 best practices.

The environment simulates a **SOC** capable of monitoring, detecting, and responding to security incidents.

This mapping is intended for **educational and portfolio purposes** and does not represent a formally certified ISO 27001 ISMS.

---

# Project Architecture

The lab environment consists of the following security infrastructure:

| Component | Role |
|----------|------|
| VMware Workstation Pro | Virtualisation platform hosting infrastructure |
| pfSense | Firewall and network security |
| Kali Linux | Administrative Workstation |
| Wazuh | SIEM, XDR, Vulnerability Detection |
| DFIR IRIS | Incident response case management |
| n8n | SOAR |
| MISP | Theat Intelligence Platform and IOC Database |


### Security Workflow

Endpoint
│
▼
Wazuh SIEM
(Log collection & detection)
│
▼
n8n
(Automation & enrichment)
│
├── Threat intelligence lookup (MISP)
│
▼
DFIR IRIS
(Incident case creation)
│
▼
SOC Investigation



Threat intelligence lookups are performed against **MISP**, which acts as the internal IOC database used for enrichment of alerts and investigations.

---

# ISO 27001 Clauses

ISO 27001 defines requirements for establishing and maintaining an **Information Security Management System (ISMS)**.

While this project does not implement a full organisational ISMS, several operational concepts from the standard are demonstrated.

| Clause | Description | Implementation in Project |
|------|-------------|---------------------------|
| 6.1 | Risk management | Security considerations influence monitoring coverage, firewall rules, and alert workflows |
| 7.2 | Competence | Skills developed through building and operating the SOC environment |
| 7.5 | Documented information | Project documentation maintained in GitHub |
| 8.1 | Operational control | Security monitoring processes implemented through SIEM and automation |
| 9.1 | Monitoring and measurement | Alerts generated and investigated using Wazuh and IRIS |
| 10.2 | Continual improvement | Detection rules and automation workflows refined based on investigations |

---

# Annex A Control Mapping

ISO 27001 Annex A defines recommended security controls.  
The following controls are demonstrated through the architecture of this project.

---

## A.5 – Organisational Controls

| Control | Control Name | Implementation |
|-------|---------------|---------------|
| A.5.7 | Threat intelligence | MISP used as a threat intelligence platform for storing and querying indicators of compromise (IOCs) |
| A.5.24 | Information security incident management | Security incidents tracked and investigated using DFIR IRIS |

---

## A.6 – People Controls

| Control | Control Name | Implementation |
|-------|---------------|---------------|
| A.6.3 | Information security awareness | Continuous learning and practical lab operation improve security awareness and operational skills |

---

## A.7 – Physical Controls

| Control | Control Name | Implementation |
|-------|---------------|---------------|
| A.7.1 | Physical security perimeters | Infrastructure hosted within a controlled local environment |
| A.7.9 | Security of assets off-premises | Virtual machines hosted and managed within VMware |

---

## A.8 – Technological Controls

| Control | Control Name | Implementation |
|-------|---------------|---------------|
| A.8.9 | Configuration management | Baseline configurations defined for firewall, SIEM, and automation systems |
| A.8.15 | Logging | Security logs collected and centralised through Wazuh Indexer d|
| A.8.16 | Monitoring activities | Wazuh base and custom rules detect suspicious activity and generate alerts |
| A.8.20 | Network security | pfSense firewall enforces network segmentation and traffic filtering |
| A.8.23 | Web filtering | Firewall policies restrict unauthorised outbound traffic |
| A.8.28 | Secure coding | Automation workflows reviewed and tested before deployment |

---

# Threat Intelligence Integration

**MISP (Malware Information Sharing Platform)** is deployed as the internal threat intelligence repository and provides:

- storage of indicators of compromise (IOCs)
- threat intelligence correlation
- enrichment of security alerts
- support for incident investigations

Automation workflows within **n8n** can query MISP to determine whether an IP address, domain, or file hash has been previously identified as malicious.

This enrichment improves analyst visibility and supports faster incident triage.

---

# Security Capabilities Demonstrated

This environment demonstrates several practical SOC security capabilities aligned with ISO 27001 operational controls.

### Centralised Logging

Logs from monitored systems are collected and analysed using **Wazuh SIEM**.

### Security Monitoring

Detection rules identify suspicious activity such as:

- authentication anomalies
- suspicious network behaviour
- integrity monitoring alerts


### Incident Management

Incidents are tracked using **DFIR IRIS**, allowing structured investigation and documentation.

### Security Automation

**n8n workflows** automate security tasks including:

- alert enrichment
- threat intelligence lookups
- incident case creation

### Network Security

The **pfSense firewall** provides network segmentation, access control, and traffic filtering.

---

# Limitations

This project demonstrates **technical security controls aligned with ISO 27001 practices**, but it does not implement a full organisational ISMS.

Elements not implemented include:

- formal governance structures
- organisational risk management processes
- management review cycles

The focus of this project is **technical security operations and SOC engineering**.

---

# Future Improvements

Potential improvements include:

- Endpoint network isolation automatition
- automated IOC ingestion into MISP
- Wazuh active response & MISP integration to automatically 
- expanded threat intelligence correlation
- automated containment workflows
- improved alert correlation across multiple sources