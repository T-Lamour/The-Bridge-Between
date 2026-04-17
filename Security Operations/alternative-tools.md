# Alternative Open-Source SOC Tools

## Overview

This document outlines alternative open-source tools that can be used to replace or extend components in this SOC architecture. The goal is to demonstrate flexibility in design and awareness of different security tooling ecosystems.

The current stack is modular, meaning each component can be swapped without breaking the overall SOC design.

---

## SIEM Alternatives

### Current: Wazuh

Wazuh is used for log ingestion, detection, and alerting.

### Alternatives:

#### ELK Stack (Elasticsearch, Logstash, Kibana)

* Provides powerful log aggregation and visualization
* Highly scalable for large environments
* Requires more manual configuration compared to Wazuh

**Use case fit:**

* Organisations needing advanced log analytics
* Custom detection pipelines

---

#### Graylog

* Centralised log management platform
* Easier setup than ELK
* Strong search and filtering capabilities

**Use case fit:**

* Mid-sized SOC environments
* Simpler SIEM replacement for Wazuh

---

## SOAR / Automation Alternatives

### Current: n8n

n8n is used for alert enrichment and automation workflows.

### Alternatives:

#### Shuffle (Open Source SOAR)

* Built specifically for security automation
* Native SOAR features (playbooks, cases, integrations)
* Designed for SOC environments

**Use case fit:**

* Enterprise-grade SOAR replacement
* More security-focused than n8n

---

#### TheHive Cortex (Automation Pairing)

* Cortex provides automated analysis of observables
* TheHive provides case management + automation hooks
* Strong SOC-native ecosystem

**Use case fit:**

* Full SOC automation stack alternative
* Tight integration between case + enrichment

---

## Endpoint / EDR Alternatives

### Current: Wazuh Agent + Logs

### Alternatives:

#### Velociraptor

* Advanced endpoint visibility and DFIR tool
* Real-time forensic data collection
* Powerful query language for endpoints

**Use case fit:**

* Incident response and threat hunting
* Advanced forensic investigations

---

#### Osquery

* SQL-like interface for endpoint monitoring
* Lightweight and flexible
* Good for asset visibility and detection queries

**Use case fit:**

* Endpoint monitoring at scale
* Security telemetry collection

---

## Firewall / Network Security Alternatives

### Current: pfSense + Suricata

### Alternatives:

#### OPNsense

* Fork of pfSense with modern UI and features
* Strong security focus
* Active development community

**Use case fit:**

* Drop-in replacement for pfSense
* Preferred in some enterprise environments

---

#### IPFire

* Lightweight Linux-based firewall
* Simple configuration and strong performance
* Built-in IDS capabilities

**Use case fit:**

* Smaller environments
* Lightweight network security setups

---

## Threat Intelligence Alternatives

### Current: MISP

### Alternatives:

#### OpenCTI

* Modern threat intelligence platform
* Graph-based relationships between indicators
* Strong API-first design

**Use case fit:**

* Advanced threat intelligence operations
* Large-scale IOC correlation

---

#### AlienVault OTX

* Community-driven threat intelligence feed
* Cloud-based IOC sharing platform
* Easy integration with SOC tools

**Use case fit:**

* Quick external intelligence enrichment
* Supplementary IOC source

---

## Overall Architecture Flexibility

This SOC is designed with modularity in mind:

* SIEM layer can be swapped (Wazuh → ELK / Graylog)
* SOAR layer can be upgraded (n8n → Shuffle / TheHive Cortex)
* Endpoint visibility can be enhanced (Wazuh → Velociraptor)
* Firewall layer can be replaced (pfSense → OPNsense)
* Threat intel can scale (MISP → OpenCTI)

This ensures the architecture can evolve from a **homelab SOC → enterprise-grade SOC design** without redesigning the entire system.

---

## Summary

The current stack represents a balanced, cost-effective SOC implementation, but multiple mature open-source alternatives exist that can extend or replace individual components depending on scale, budget, and operational requirements.

Understanding these alternatives demonstrates architectural awareness and adaptability in security engineering.

---
