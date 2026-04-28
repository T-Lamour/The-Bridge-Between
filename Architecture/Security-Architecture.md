🧩 Architecture Overview

A simple breakdown of the tools used in this project — what they do, and why they matter.

Data Flow

The system processes events through the following pipeline:

Endpoints generate logs and security events
Wazuh ingests and analyses logs
Alerts are forwarded to n8n via webhook
n8n performs enrichment using threat intelligence sources
Enriched alerts are converted into incidents in DFIR IRIS
Analysts investigate and respond

## Wazuh - SIEM / XDR Layer

Role:
Centralised log collection, detection, and alerting engine.

Key Capabilities:

Log ingestion from endpoints (agents)
Rule-based detection (Wazuh ruleset)
File integrity monitoring (FIM)
Vulnerability detection
Security Configuration Assessment (SCA)

Function in Architecture:

Acts as the primary detection engine
Normalises and correlates log data
Generates alerts for downstream processing

## n8n - SOAR / Automation Layer

Role:
Workflow automation and orchestration engine.

Key Capabilities:

Webhook ingestion from Wazuh
Integration with external APIs (VirusTotal, AbuseIPDB, MISP)
Conditional logic and branching workflows
Automated response actions

Function in Architecture:

Acts as the orchestration layer
Enriches alerts with contextual threat intelligence
Automates triage and response workflows.

## DFIR IRIS - Incident Response Platform

Role:
Case management and investigation platform.

Key Capabilities:

Incident creation and tracking
Evidence management
Timeline analysis
Analyst collaboration

Function in Architecture:

Serves as the incident management layer
Stores enriched alerts as structured cases
Provides a workflow for investigation and response

## MISP - Threat Intelligence Platform

Role:
Threat intelligence storage and correlation.

Key Capabilities:

IOC storage (IPs, domains, hashes)
Threat sharing and feeds
Event correlation
Integration with enrichment workflows

Function in Architecture:

Provides internal threat intelligence
Enhances alert context and confidence scoring

## OPNsense - Network Security Layer

Role:
Firewall and network segmentation.

Key Capabilities:

Traffic filtering and control
VLAN/network segmentation (VLAN 10, 20, 30)
Firewall rules and NAT
Suricata IDS — real-time packet inspection
API-driven automated IP blocking
Logging of network activity forwarded to Wazuh

Function in Architecture:

Provides network-level security controls
Segments the lab into Management, SOC, and Victim VLANs
Acts as the automated enforcement point — n8n pushes block rules via the OPNsense API

🔗 Integration Points
| Source    | Destination | Method          | Purpose                           |
| --------- | ----------- | --------------- | --------------------------------- |
| Wazuh     | n8n         | Webhook         | Forward alerts for processing     |
| n8n       | MISP        | API             | IOC correlation / ingestion       |
| n8n       | DFIR IRIS   | API             | Case creation                     |
| OPNsense  | Wazuh       | Syslog (UDP 514)| Network and Suricata visibility   |
| n8n       | OPNsense    | API             | Automated IP blocking             |
