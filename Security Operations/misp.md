# MISP

<p align="center">
  <img src="Images/MISP/logo.png" width="220"/>
</p>

## Overview

MISP is used as the **threat intelligence platform** within this SOC. Its primary role is to act as a **centralised repository of Indicators of Compromise (IOCs)** that can be referenced during alert processing and investigations.

In this project, MISP functions both as an **internal IOC database** populated from previous incidents, and as a **feed aggregator**, automatically pulling indicators from external threat intelligence sources.

---

## Key Responsibilities

* Store and manage Indicators of Compromise (IOCs)
* Aggregate indicators from external threat intelligence feeds
* Provide threat intelligence context during alert enrichment
* Enable correlation between alerts and known malicious indicators
* Support incident investigations with historical and external intelligence

---

## Architecture Role

MISP is integrated into the SOC as a **reference point for enrichment**, queried by n8n during alert processing:

```
     Wazuh SIEM
 (Alert Generation)
        │
        ▼
        n8n
 (Enrichment Engine)
        │
        ├──────────────► MISP (IOC Lookup)
        │
        ▼
    DFIR IRIS
 (Incident Creation)
```

---

## Infrastructure

MISP runs on a dedicated **VMware Workstation VM** as a **Docker container**, with its web interface used for feed management, IOC review, and event organisation.

---

## Threat Intelligence Feeds

MISP is configured to automatically pull indicators from a selection of external threat intelligence feeds. These feeds are updated on a scheduled basis, keeping the IOC database current without manual input.

Feed types include:

* Malicious IP address lists
* Known C2 (Command and Control) infrastructure
* Phishing and malware domain lists
* File hashes associated with known malware families

*Add screenshot here – MISP feed configuration view showing active feeds and last sync status*

This continuous feed ingestion ensures that even a first-time encounter with a known threat is flagged immediately during the enrichment phase.

---

## IOC Management

MISP stores a variety of indicators used during the enrichment phase.

### Supported IOC Types:

* IP addresses (source and destination)
* Domain names
* URLs
* File hashes (MD5, SHA256)
* Email addresses

### IOC Sources:

* **Threat intelligence feeds** – automatically ingested from external sources on a schedule
* **Previous incidents** – indicators discovered during investigations, added back into MISP
* **Manual analyst input** – ad-hoc entries for targeted or emerging threats

---

## Enrichment Workflow

MISP is queried during the **n8n enrichment phase** for every alert processed.

### Process:

1. Alert received from Wazuh
2. Indicators are extracted (source IP, domain, URL, file hash)
3. n8n queries MISP for matching IOCs via its API
4. MISP returns context such as:

   * Known malicious status
   * Associated threat event or actor name
   * Feed source and confidence level
   * Historical sightings count

*Add screenshot here – MISP event overview showing IOC matches and associated threat context*

---

## Correlation & Detection

MISP enables correlation between current alerts and previously identified or feed-provided threats.

### Example:

* A login alert contains a source IP address
* n8n queries MISP during the enrichment phase
* The IP matches a C2 infrastructure indicator tagged as high confidence
* Alert severity is escalated within n8n
* Automated response is triggered (IP block via OPNsense, account disable via Entra)

This allows the SOC to react immediately to **known threats** without waiting for manual analysis.

*Add screenshot here – MISP IOC detail view showing indicator value, tags, and related events*

---

## Feedback Loop

A key design feature of the MISP integration is the **intelligence feedback loop**:

1. Incident occurs and is investigated in DFIR IRIS
2. New indicators are identified during the investigation
3. Indicators are added to MISP via n8n automation or analyst input
4. Future alerts containing those indicators are enriched with this context

This continuously strengthens the SOC's ability to detect and respond to known threats over time.

---

## Integration with SOC Workflow

MISP integrates with:

* **n8n** → queried via REST API during enrichment workflows
* **DFIR IRIS** → provides intelligence context for active investigations
* **Wazuh** → indirectly improves detection accuracy through enriched alert context

---

## Key Benefits

* Centralised IOC repository combining feed data and incident-derived indicators
* Faster identification of known threats, including those from external feeds
* Improves alert accuracy and severity prioritisation through automated correlation
* Enables intelligence-driven response without manual lookup
* Builds long-term threat visibility that improves with every incident handled

---

## Limitations

* Feed quality directly affects the accuracy of enrichment results
* Indicators without context (untagged, unclassified) reduce the value of matches
* Not configured for external threat sharing in this project — used as an internal platform only

---

## Design Approach

MISP is used with a focused and practical approach:

* **Feed-augmented** – external feeds keep the IOC database current automatically
* **IOC-first usage** – prioritising indicators over complex threat modelling
* **Tight integration with automation** – queried during every enrichment workflow
* **Continuous improvement** – new indicators from incidents are fed back into MISP
* **Lightweight deployment** – avoiding unnecessary complexity for the environment size

---

## Summary

MISP provides the **threat intelligence layer** of the SOC by combining automatically ingested external feeds with internally derived incident indicators.

It enables:

* Immediate identification of known malicious indicators
* Improved alert enrichment and severity prioritisation
* An intelligence feedback loop that strengthens detection with every incident handled

---
