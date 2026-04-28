# NIST Cybersecurity Framework (CSF) Mapping

## Overview

The NIST Cybersecurity Framework (CSF) provides a structured approach for managing cybersecurity risk across five core functions. This document maps the tools and capabilities deployed in this SOC to specific CSF subcategories, demonstrating how open-source tooling covers the full security lifecycle.

This mapping is intended for **educational and portfolio purposes** and does not represent formal organisational compliance.

---

## SOC Stack

| Tool | Role |
|------|------|
| OPNsense + Suricata | Firewall, VLAN segmentation, network IDS |
| Wazuh | SIEM, log collection, FIM, vulnerability scanning |
| n8n | SOAR — enrichment, decision logic, automated response |
| MISP | Threat intelligence — IOC storage and feed ingestion |
| DFIR IRIS | Case management — incident tracking from detection to closure |

---

## CSF Function Mapping

| Function | Subcategory | Implementation |
|----------|-------------|----------------|
| **Identify** | ID.AM-1: Physical device inventory | Wazuh agent enrolment registers all monitored endpoints |
| | ID.AM-2: Software/services inventory | Wazuh vulnerability module + SCA scans |
| | ID.RA-3: Threats identified and documented | MISP feeds (CIRCL, abuse.ch) + use-case threat modelling |
| | ID.RA-5: Risks prioritised | Wazuh rule severity levels (1–15) drive enrichment thresholds |
| **Protect** | PR.AC-3: Remote access managed | OPNsense firewall rules restrict inter-VLAN access |
| | PR.AC-4: Access granted with least privilege | VLAN 30 (Victim) isolated; SOC tools only reachable from VLAN 10 |
| | PR.DS-5: Protections against data leaks | OPNsense blocks outbound traffic from unauthorised VLANs |
| | PR.IP-1: Baseline configuration maintained | Netplan static IPs; Docker Compose pinned image tags |
| | PR.PT-3: Principle of least functionality | Each VM runs a single-service Docker stack |
| | PR.PT-4: Network communications protected | VLAN segmentation; management plane isolated from victim lab |
| **Detect** | DE.AE-1: Network baseline established | Suricata ET Open signatures define expected traffic patterns |
| | DE.AE-2: Detected events analysed | Wazuh correlates Suricata + endpoint logs; severity assigned |
| | DE.AE-3: Event data aggregated | Wazuh Manager ingests agents + OPNsense syslog + Windows events |
| | DE.AE-5: Incident alert thresholds established | n8n decision node: AbuseIPDB >50 OR VirusTotal >3 triggers response |
| | DE.CM-1: Network monitored for attacks | Suricata on OPNsense — all interfaces, real-time inspection |
| | DE.CM-3: Personnel activity monitored | Wazuh agent covers authentication, process execution, FIM |
| | DE.CM-4: Malicious code detected | Wazuh FIM + VirusTotal hash lookup via n8n |
| | DE.CM-7: Unauthorised assets monitored | Wazuh agent enrolment; unenrolled hosts generate no telemetry |
| **Respond** | RS.RP-1: Response plan executed | n8n workflow executes automatically on confirmed alerts |
| | RS.AN-1: Notifications from detections investigated | DFIR IRIS case created per confirmed incident, assigned to analyst |
| | RS.AN-3: Forensics performed | DFIR IRIS timeline, IOC tab, and asset tab capture enriched context |
| | RS.MI-1: Incidents contained | n8n blocks source IP via OPNsense `blocklist_dynamic` alias |
| | RS.MI-2: Incidents mitigated | n8n disables compromised accounts via Microsoft Entra Graph API |
| | RS.CO-3: Information shared | MISP IOC events written after each confirmed incident |
| **Recover** | RC.RP-1: Recovery plan executed | Analyst closes IRIS case; MISP updated; Wazuh rules tuned |
| | RC.IM-1: Response strategies improved | Post-incident review in IRIS; n8n thresholds adjusted |
| | RC.IM-2: Recovery strategies improved | OPNsense blocklist reviewed; false positives removed |
| | RC.CO-3: Recovery communicated | IRIS case closure notes document actions taken and outcome |

---

## Function Detail

### 1. Identify (ID)

The SOC maintains asset awareness through Wazuh agent enrolment — every monitored endpoint is registered with the Wazuh Manager. The vulnerability detection module scans installed packages against the NVD CVE database. MISP threat intelligence feeds (CIRCL OSINT, abuse.ch) populate the known-bad IOC database before any incident occurs, providing a pre-built threat baseline.

---

### 2. Protect (PR)

OPNsense enforces network segmentation across three VLANs: Management (10.10.1.0/24), SOC tools (10.10.10.0/24), and the victim lab (10.10.20.0/24). Firewall rules prevent VLAN 30 from reaching SOC tooling except on the Wazuh agent enrolment port. Each VM runs a single-purpose Docker Compose stack with no unnecessary services. Static IPs are assigned via Netplan — no DHCP drift on the SOC subnet.

---

### 3. Detect (DE)

Detection operates at two layers simultaneously:

- **Network layer** — Suricata on OPNsense inspects all inter-VLAN and outbound traffic using the Emerging Threats Open ruleset, updated daily. Alerts are forwarded to Wazuh via syslog.
- **Endpoint layer** — Wazuh agents on monitored hosts collect authentication events, process execution, file integrity changes, and Windows Security logs.

Wazuh correlates events from both layers and fires rules. Level 7+ alerts are forwarded to n8n for enrichment.

---

### 4. Respond (RS)

n8n executes an automated response workflow on every confirmed-malicious alert. The workflow runs in under 15 seconds from alert receipt:

1. Enrichment — AbuseIPDB confidence score and VirusTotal detection count retrieved in parallel
2. Decision — if confidence exceeds threshold, response proceeds
3. MISP check — is this IOC already known?
4. IRIS case created — with enrichment data, IOC, and affected asset attached
5. OPNsense block — source IP added to `blocklist_dynamic` and reconfigured
6. (Where applicable) — Microsoft Entra account disabled via Graph API

Human analysts use DFIR IRIS to manage the ongoing investigation and close the case.

---

### 5. Recover (RC)

Recovery is built into the pipeline design. After each incident:

- IRIS case is reviewed and closed with outcome documentation
- New IOCs confirmed during investigation are added to MISP for future correlation
- If the alert was a false positive, the Wazuh rule or n8n threshold is adjusted
- OPNsense blocklist is reviewed to remove any incorrectly blocked IPs

This creates a feedback loop where each incident improves detection quality for the next one.

---

## Data Flow by CSF Layer

```
Identify   →  Wazuh agent inventory + MISP feed ingestion
Protect    →  OPNsense VLAN segmentation + firewall rules
Detect     →  Suricata (network) + Wazuh (endpoint) → alerts
Respond    →  n8n enrichment + OPNsense block + IRIS case
Recover    →  IRIS review + MISP update + rule tuning
```

---

## Limitations

This implementation covers the technical controls within the NIST CSF but does not include:

* Formal governance structures or risk registers
* Organisational policies or security awareness training programmes
* Business continuity or disaster recovery planning
* Legal and regulatory compliance enforcement

The focus is on **technical SOC operations**: detection, automated response, and continuous improvement through the incident feedback loop.

---
