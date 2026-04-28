# ISO/IEC 27001:2022 Control Mapping

## Overview

ISO/IEC 27001:2022 is the current edition of the international standard for Information Security Management Systems (ISMS). This document maps the technical controls deployed in this SOC to the Annex A controls in the 2022 edition.

All Annex A references use **ISO 27001:2022 numbering**. The 2022 revision reorganised the control set from 114 controls (2013) into 93 controls across four categories: Organisational (A.5), People (A.6), Physical (A.7), and Technological (A.8).

This mapping is a **technical demonstration**, not a certified ISMS. Formal certification requires governance documentation, management commitment, scope definition, and auditor assessment — none of which are in scope for this project.

---

## Control Mapping

### A.5.7 — Threat Intelligence

**Requirement:** Collect and analyse information about threats relevant to the organisation.

**Implementation:**

* MISP ingests threat intelligence feeds (CIRCL OSINT, abuse.ch Feodo Tracker, URLhaus, MalwareBazaar) on a daily schedule
* n8n queries AbuseIPDB and VirusTotal for every alert's source IP, returning confidence scores and historical reports
* Confirmed IOCs from resolved incidents are written back to MISP, creating an internal threat database that improves over time

**Outcome:** Every enrichment decision is informed by both external feeds and internal incident history.

---

### A.5.15 — Access Control

**Requirement:** Rules to control physical and logical access to information and assets shall be established and implemented.

**Implementation:**

* OPNsense firewall rules enforce VLAN-level access control — VLAN 30 (Victim) cannot reach SOC tools on VLAN 20 except through permitted ports
* SOC tool web interfaces are accessible only from VLAN 10 (Management)
* Each tool uses separate credentials; no shared passwords across the stack
* API keys are scoped — MISP API key is restricted to the n8n VM IP (`10.10.10.20`)

---

### A.5.24 — Information Security Incident Management Planning and Preparation

**Requirement:** An incident response process shall be planned and prepared.

**Implementation:**

* n8n workflows define a repeatable, documented response process for each incident type (brute force, malicious file, suspicious login, C2 beaconing)
* DFIR IRIS provides structured case templates pre-populated with investigation task checklists
* The pipeline runs automatically — no manual triage required for the initial response phase

---

### A.5.25 — Assessment and Decision on Information Security Events

**Requirement:** Security events shall be assessed to determine whether they qualify as incidents.

**Implementation:**

* n8n applies a decision node after enrichment: AbuseIPDB confidence >50 OR VirusTotal detections >3 → confirmed incident
* Alerts that do not meet the threshold are discarded without creating a case, reducing analyst noise
* The threshold is configurable per workflow to accommodate tuning as alert patterns become clearer

---

### A.5.26 — Response to Information Security Incidents

**Requirement:** Incidents shall be responded to in accordance with documented procedures.

**Implementation:**

* Automated response executes within seconds: OPNsense IP block, Microsoft Entra account disable (where applicable), IRIS case creation with IOC and asset attached
* Response actions are logged in the IRIS case timeline for full audit trail
* Analyst assigns severity, updates case status, and closes after investigation

---

### A.5.27 — Learning from Information Security Incidents

**Requirement:** Knowledge gained from incidents shall be used to improve controls.

**Implementation:**

* Post-incident IOCs are written to MISP — future alerts involving the same indicator are enriched with known-bad context
* False positives identified during case review feed back into n8n threshold adjustment and Wazuh rule tuning
* IRIS case closure notes document what changed and why

---

### A.8.2 — Privileged Access Rights

**Requirement:** Privileged access rights shall be restricted and managed.

**Implementation:**

* Each SOC tool runs with a dedicated admin account; no tool shares credentials with another
* n8n API credentials are stored encrypted (AES-256 via N8N_ENCRYPTION_KEY) and never exposed in workflow JSON exports
* OPNsense API user (`n8n-api`) is scoped to firewall alias operations only

---

### A.8.8 — Management of Technical Vulnerabilities

**Requirement:** Information about technical vulnerabilities shall be obtained and acted upon.

**Implementation:**

* Wazuh vulnerability detection module scans installed packages on all enrolled agents against the NVD CVE database
* Results are visible in the Wazuh dashboard under Vulnerability Detector
* Docker image versions are pinned in Compose files — unplanned updates do not introduce untested changes

---

### A.8.15 — Logging

**Requirement:** Logs that record activities, exceptions, faults, and other relevant events shall be produced, stored, protected, and analysed.

**Implementation:**

* Wazuh agents collect authentication, process, file, and system logs from all enrolled endpoints
* OPNsense forwards firewall and Suricata logs to Wazuh via syslog (UDP 514)
* All logs are centralised in the Wazuh Indexer (OpenSearch) with configurable retention
* n8n execution logs record every workflow run, including input payloads and response outputs

---

### A.8.16 — Monitoring Activities

**Requirement:** Networks, systems, and applications shall be monitored for anomalous behaviour.

**Implementation:**

* Suricata monitors all network interfaces on OPNsense in real time using the Emerging Threats Open ruleset
* Wazuh agents monitor authentication events, process execution, and file integrity continuously
* n8n receives Wazuh alerts via webhook — enrichment and decision logic run automatically on every level 7+ alert
* The pipeline produces an IRIS case for every confirmed incident without requiring analyst polling

---

### A.8.20 — Networks Security

**Requirement:** Networks shall be secured, managed, and controlled to protect information in systems and applications.

**Implementation:**

* Three isolated VLANs separate the management plane, SOC tools, and victim lab
* OPNsense enforces inter-VLAN firewall rules — lateral movement between zones requires explicit permit rules
* The `blocklist_dynamic` alias provides a real-time IP block list updated automatically by n8n during incident response

---

### A.8.22 — Segregation of Networks

**Requirement:** Groups of information services, users, and information systems shall be segregated in networks.

**Implementation:**

* VLAN 10 (10.10.1.0/24) — Management only; hypervisor and OPNsense console access
* VLAN 20 (10.10.10.0/24) — SOC tools; Wazuh, n8n, MISP, DFIR IRIS
* VLAN 30 (10.10.20.0/24) — Victim lab; monitored endpoints, no access to SOC tooling except Wazuh agent port 1514
* All routing passes through OPNsense — no direct VLAN-to-VLAN path exists outside of permitted firewall rules

---

### A.8.23 — Web Filtering

**Requirement:** Access to external websites shall be managed to reduce exposure to malicious content.

**Implementation:**

* OPNsense firewall rules control outbound internet access per VLAN
* VLAN 30 (Victim) outbound is restricted — used in use-case scenarios to demonstrate C2 detection before traffic escapes
* DNS-based domain blocking via the `blocklist_domains` alias blocks known malicious domains network-wide

---

## Control Coverage Summary

| Control | Area | Implemented By |
|---------|------|----------------|
| A.5.7 | Threat Intelligence | MISP + AbuseIPDB + VirusTotal (via n8n) |
| A.5.15 | Access Control | OPNsense firewall rules + credential scoping |
| A.5.24 | Incident Planning | n8n workflows + IRIS templates |
| A.5.25 | Incident Assessment | n8n decision node (enrichment thresholds) |
| A.5.26 | Incident Response | n8n automation + DFIR IRIS case management |
| A.5.27 | Learning from Incidents | MISP feedback loop + rule tuning |
| A.8.2 | Privileged Access | Scoped API keys + encrypted credential storage |
| A.8.8 | Vulnerability Management | Wazuh vulnerability detection module |
| A.8.15 | Logging | Wazuh centralised log collection |
| A.8.16 | Monitoring | Suricata + Wazuh continuous monitoring |
| A.8.20 | Network Security | OPNsense + VLAN segmentation + blocklist |
| A.8.22 | Network Segregation | Three-VLAN architecture |
| A.8.23 | Web Filtering | OPNsense domain blocklist alias |

---

## Risk-Based Design

The SOC architecture was designed around the attack scenarios most relevant to SMB environments:

| Risk | Likelihood | Control |
|------|-----------|---------|
| Credential brute force | High | Wazuh rule 60106 → n8n auto-block |
| Account compromise (M365) | High | Entra sign-in monitoring → account disable |
| Malicious file download | Medium | Wazuh FIM + VirusTotal hash → host isolation |
| C2 beaconing | Medium | Suricata ET rules → OPNsense block + DNS sinkhole |
| Lateral movement | Medium | VLAN segmentation + log correlation |
| Unpatched vulnerabilities | High | Wazuh vulnerability detection → analyst review |

---

## Limitations

This mapping covers the technical control layer only. A complete ISO 27001 ISMS also requires:

* Scope definition and Statement of Applicability (SoA)
* Information security policies and documented procedures
* Risk assessment and risk treatment plan
* Management review and internal audit programme
* Evidence of continual improvement

This project demonstrates **technical control implementation** — the controls that underpin the operational layer of an ISMS — not the governance and policy layer above it.

---
