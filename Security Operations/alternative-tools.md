# Alternative Open-Source SOC Tools

## Overview

This document outlines alternative open-source tools that can replace or extend individual components in this SOC stack. The architecture is intentionally modular — each layer can be swapped without breaking the overall pipeline design.

Each section covers: what the alternative does differently, when it makes more sense than the current choice, and what the trade-offs are.

---

## Firewall / Network Security

### Current: OPNsense + Suricata

OPNsense handles VLAN segmentation, firewall enforcement, and hosts Suricata for network IDS. The REST API enables n8n to push automated block rules without manual intervention.

### Alternative: pfSense + Suricata

pfSense is the upstream project OPNsense forked from in 2015. It remains widely deployed and has a larger body of community guides.

| | OPNsense | pfSense |
|--|----------|---------|
| Release cadence | Weekly | Monthly |
| REST API | Native, well-structured | Third-party package (FauxAPI) required |
| UI | Modern, menu-driven | Legacy layout |
| Suricata support | First-party plugin | Available, less integrated |
| Community | Active, security-focused | Larger but split since fork |

**When to prefer pfSense:** Existing deployment, team familiarity, or specific package availability. For a greenfield SOC lab, OPNsense is the better starting point — the API alone makes automated response significantly easier to build.

---

### Alternative: IPFire

A lightweight Linux-based firewall distribution with a minimal footprint.

* Simpler configuration than OPNsense
* Built-in IDS (Suricata-based)
* Lacks OPNsense's REST API — automated response requires workarounds

**When to use:** Resource-constrained environments where OPNsense's memory footprint is a problem. Not suitable as a direct replacement if automated IP blocking via API is required.

---

## SIEM / Detection

### Current: Wazuh

Wazuh is an all-in-one agent-based SIEM covering log ingestion, rule-based detection, FIM, vulnerability scanning, and SCA. A single Docker Compose stack runs the Manager, Indexer (OpenSearch), and Dashboard.

### Alternative: ELK Stack (Elasticsearch, Logstash, Kibana)

The ELK stack is the most widely deployed open-source log analytics platform.

| | Wazuh | ELK Stack |
|--|-------|-----------|
| Detection rules | Pre-built ruleset (3000+ rules) | Custom only — must write your own |
| Agent coverage | Wazuh agent handles collection + detection | Beats agents for collection, no built-in detection |
| Deployment complexity | Medium — single Compose stack | High — multiple components, significant tuning |
| Scalability | Limited single-node | Designed for large-scale horizontal scaling |
| FIM / SCA | Built-in | Not included |

**When to use:** Organisations with large log volumes needing custom analytics pipelines, or those already invested in the Elastic ecosystem. For a security-first homelab where detection matters more than analytics, Wazuh wins.

---

### Alternative: Graylog

A log management platform with strong search, stream routing, and alerting.

* Easier setup than ELK
* Better out-of-the-box alerting than raw Elasticsearch
* No agent-based detection — requires a separate IDS layer
* Alerting integrates well with n8n via webhook

**When to use:** When log search and correlation are the priority over pre-built detection rules. Can sit alongside Suricata and forward alerts to n8n without Wazuh in the middle.

---

## SOAR / Automation

### Current: n8n

n8n is a self-hosted workflow automation engine with a visual editor. It handles Wazuh webhook ingestion, parallel enrichment via VirusTotal and AbuseIPDB, decision logic, IRIS case creation, and OPNsense IP blocking — all without writing code.

### Alternative: Shuffle

Shuffle is purpose-built for security automation, modelled on the commercial SOAR product UX.

| | n8n | Shuffle |
|--|-----|---------|
| Security-native | No — general automation | Yes — built for SOC workflows |
| Pre-built integrations | 400+ general | 300+ security-focused |
| Code requirement | None | None |
| Deployment | Docker Compose, simple | Docker Compose, simple |
| Community | Large, active | Smaller, security-focused |
| API flexibility | Full HTTP Request node | More opinionated integration model |

**When to use:** If you want pre-built security playbooks and tighter SOC tooling integration out of the box. n8n is more flexible for custom logic; Shuffle gives you security context faster.

---

### Alternative: TheHive + Cortex

TheHive is a case management platform that also supports automation via Cortex, its companion observable analysis engine.

* Cortex runs analysers (VirusTotal, AbuseIPDB, etc.) automatically when observables are added to a case
* TheHive and DFIR IRIS overlap significantly — both handle case management
* Replacing both n8n and IRIS with TheHive + Cortex is a valid architecture for teams preferring a tighter case-first workflow

**When to use:** When case management and enrichment should be coupled in one product rather than orchestrated across n8n + IRIS.

---

## Endpoint Visibility / EDR

### Current: Wazuh Agent

The Wazuh agent covers log collection, FIM, vulnerability detection, and active response from a single deployment.

### Alternative: Velociraptor

Velociraptor is a DFIR and threat hunting platform with powerful endpoint query capabilities.

* VQL (Velociraptor Query Language) enables targeted, real-time forensic data collection
* Better for incident response investigations than continuous monitoring
* No built-in detection rules — requires analyst-written hunts
* Complements Wazuh rather than replacing it: use Wazuh for ongoing detection, deploy Velociraptor when an incident requires deep endpoint investigation

**When to use:** During active incidents where you need to query filesystem artefacts, process trees, registry keys, or memory across multiple endpoints simultaneously.

---

### Alternative: Osquery

SQL-based endpoint telemetry collection.

* Lightweight agent with minimal performance impact
* Query endpoints the way you query a database
* No native alerting — requires a log shipper (e.g., Filebeat) to send data to Wazuh or ELK
* Best used alongside Wazuh to extend endpoint visibility without replacing it

**When to use:** When you need asset inventory or custom detection queries that go beyond Wazuh's default ruleset.

---

## Threat Intelligence

### Current: MISP

MISP is a self-hosted IOC database and sharing platform. In this SOC it serves two roles: n8n queries it before enrichment (known bad?), and n8n writes confirmed IOCs back to it after incidents close (feedback loop).

### Alternative: OpenCTI

OpenCTI is a newer threat intelligence platform with a graph-based data model built on STIX 2.1.

| | MISP | OpenCTI |
|--|------|---------|
| Data model | Flat events and attributes | Graph — entities, relationships, observables |
| Feed ecosystem | Large, mature | Growing |
| API | REST | GraphQL |
| Deployment complexity | Medium | High — separate connectors for each feed |
| IOC lookup speed | Fast | Fast |

**When to use:** When relationships between threat actors, campaigns, and indicators matter as much as the IOCs themselves. For a detection-focused SOC where you just need fast IOC lookups and feed ingestion, MISP is the simpler choice.

---

### Alternative: AlienVault OTX (Open Threat Exchange)

A cloud-hosted community threat intelligence platform with a free API.

* No self-hosting required — call the API from n8n directly
* Community-contributed IOCs across IPs, domains, and file hashes
* Less control than MISP — you can't push your own IOCs back in the same way
* Useful as a supplementary enrichment source alongside MISP, not a full replacement

**When to use:** As an additional enrichment API in n8n, queried in parallel with AbuseIPDB and VirusTotal. Adds breadth without replacing MISP's internal feedback loop.

---

## Case Management

### Current: DFIR IRIS

DFIR IRIS structures every confirmed incident into a case with IOC lists, affected assets, a timeline, and task tracking. n8n creates cases via API; analysts manage them through closure.

### Alternative: TheHive

TheHive is a mature open-source case management platform with tight Cortex integration.

| | DFIR IRIS | TheHive |
|--|-----------|---------|
| API-driven case creation | Yes | Yes |
| Built-in enrichment | No (relies on n8n) | Yes (via Cortex) |
| Timeline | Yes | Yes |
| IOC management | Yes | Yes |
| Deployment complexity | Low | Medium (TheHive + Cortex + Cassandra) |

**When to use:** When you want enrichment to happen inside the case management tool rather than upstream in n8n. The TheHive + Cortex pairing effectively absorbs part of what n8n does in this architecture.

---

## Architecture Flexibility Summary

| Layer | Current | Primary Alternative | Swap Difficulty |
|-------|---------|---------------------|-----------------|
| Firewall | OPNsense + Suricata | pfSense + Suricata | Low |
| SIEM | Wazuh | ELK Stack | High |
| SOAR | n8n | Shuffle | Medium |
| Endpoint | Wazuh Agent | Velociraptor (supplementary) | Low |
| Threat Intel | MISP | OpenCTI | Medium |
| Case Management | DFIR IRIS | TheHive | Medium |

The stack is designed so any single layer can be replaced without touching the others. The integration contracts — webhooks from Wazuh, REST APIs to MISP and IRIS, API calls to OPNsense — are the only coupling points.

---
