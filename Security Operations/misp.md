<p align="center">
  <img src="Images/MISP/logo.png" width="220"/>
</p>

## Overview

MISP is used as the **threat intelligence platform** within this SOC. Its primary role is to act as a **centralised repository of Indicators of Compromise (IOCs)** that can be referenced during alert processing and investigations.

In this project, MISP is not used as a full threat-sharing platform, but rather as an **internal IOC database** to support detection, enrichment, and response.

---

## Key Responsibilities

* Store and manage Indicators of Compromise (IOCs)
* Provide threat intelligence context during alert enrichment
* Enable correlation between alerts and known malicious indicators
* Support incident investigations with historical intelligence

---

## Architecture Role

MISP is integrated into the SOC as a **reference point for enrichment**:

```id="misp-flow"
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

## IOC Management

MISP stores a variety of indicators that can be referenced during alert processing.

### Supported IOC Types:

* IP addresses
* Domain names
* URLs
* File hashes (MD5, SHA256)
* Email addresses

These indicators can originate from:

* Previous incidents
* Threat intelligence feeds
* Manual analyst input

---

## Enrichment Workflow

MISP is queried during the **n8n enrichment phase**.

### Process:

1. Alert received from Wazuh
2. Extract indicators (e.g. source IP, URL, hash)
3. Query MISP for matching IOCs
4. Return context such as:

   * Known malicious status
   * Associated threat information
   * Historical sightings

---

## Correlation & Detection

MISP enables correlation between:

* Current alerts
* Previously identified threats

### Example:

* A login alert contains an IP address
* n8n checks MISP
* IP is already tagged as malicious
* Alert severity is increased
* Automated response is triggered

This allows the SOC to react faster to **known threats**.

---

## Integration with SOC Workflow

MISP integrates with:

* **n8n** → queried during enrichment workflows
* **Wazuh** → indirectly supports detection through enriched context
* **DFIR IRIS** → provides intelligence for investigations

Additionally, new IOCs identified during incidents can be:

* Added back into MISP
* Reused for future detections

---

## Feedback Loop

A key design feature is the **feedback loop**:

1. Incident occurs
2. Indicators are identified
3. Indicators are stored in MISP
4. Future alerts are enriched using these indicators

This continuously improves detection and response over time.

---

## Example Interface

### Event Overview

*Add screenshot here*

Displays:

* Threat events and associated indicators
* Tags and classifications

---

### IOC Details

*Add screenshot here*

Shows:

* Indicator type and value
* Associated threat context
* Related events

---

### Search & Correlation

*Add screenshot here*

Shows:

* Ability to search for indicators
* Matches across historical data

---

## Key Benefits

* Centralised IOC repository
* Faster identification of known threats
* Improves alert accuracy and prioritisation
* Enables intelligence-driven response
* Builds long-term threat visibility

---

## Limitations

* Requires manual input or feeds to remain effective
* Limited value without integration (handled via n8n)
* Not used for external threat sharing in this project

---

## Design Approach

MISP is used with a focused and practical approach:

* **IOC-first usage** – prioritising indicators over complex threat modelling
* **Tight integration with automation** – used during enrichment workflows
* **Continuous improvement** – feeding new indicators back into the system
* **Lightweight deployment** – avoiding unnecessary complexity

---

## Summary

MISP provides the **threat intelligence layer** of the SOC by acting as a central IOC database.

It enables:

* Faster identification of malicious indicators
* Improved alert enrichment and prioritisation
* A feedback loop that strengthens detection over time

This ensures the SOC evolves and improves with each incident handled.

---
