<p align="center">
  <img src="Images/Iris/logo
.png" width="420"/>
</p>

## Overview

DFIR IRIS is used as the **case management and incident response platform** within this SOC. It acts as the central location for tracking, investigating, and managing security incidents generated from alerts.

Within this project, IRIS ensures that all security events are properly documented, investigated, and resolved in a structured and repeatable way.

---

## Key Responsibilities

* Incident creation and tracking
* Case management and organisation
* Investigation workflow support
* Evidence and artefact collection
* Documentation of response actions

---

## Architecture Role

DFIR IRIS sits at the **end of the detection and enrichment pipeline**, where alerts become structured incidents:

```id="iris-flow"
     Wazuh SIEM
 (Detection & Alerts)
        │
        ▼
        n8n
 (Enrichment & Automation)
        │
        ▼
    DFIR IRIS
 (Case Management)
```

---

## Incident Creation

Incidents are automatically created in IRIS via **n8n workflows**.

### Included Data:

* Alert details from Wazuh
* Enriched threat intelligence
* Indicators (IP addresses, domains, hashes)
* Event timeline and metadata

This ensures analysts have all relevant context from the start.

---

## Case Structure

Each incident in IRIS is structured to support efficient investigation:

### Typical Case Includes:

* **Title** – Summary of the incident (e.g. "Malicious Login Detected")
* **Description** – Detailed context and alert information
* **Severity Level** – Based on enrichment and detection logic
* **Indicators of Compromise (IOCs)** – IPs, domains, hashes
* **Tasks** – Investigation and response steps
* **Comments** – Analyst notes and findings

---

## Investigation Workflow

IRIS supports a structured investigation process:

1. **Triage**

   * Review alert context and severity
   * Determine if the alert is a true positive

2. **Analysis**

   * Examine logs and enriched data
   * Correlate with threat intelligence
   * Identify scope and impact

3. **Containment**

   * Execute or verify response actions
   * Prevent further malicious activity

4. **Eradication & Recovery**

   * Remove threats and restore systems
   * Confirm environment is secure

5. **Closure**

   * Document findings
   * Mark incident as resolved
   * Retain for future reference

---

## Integration with SOC Workflow

DFIR IRIS integrates with other tools in the environment:

* **n8n** → creates and updates incidents automatically
* **Wazuh** → provides original alert data
* **MISP** → supplies threat intelligence for correlation
* **Response Systems** → actions triggered via automation (pfSense, Microsoft Entra)

This ensures a seamless transition from detection to response.

---

## Example Interface

### Case Dashboard

*Add screenshot here*

Displays:

* Open and closed incidents
* Severity distribution
* Investigation status

---

### Incident Details

*Add screenshot here*

Shows:

* Full alert context
* Enriched data
* Associated IOCs

---

### Task Management

*Add screenshot here*

Shows:

* Investigation steps
* Assigned actions
* Progress tracking

---

## Key Benefits

* Centralised incident tracking
* Structured and repeatable investigation process
* Improved visibility into security events
* Supports collaboration and documentation
* Aligns with real-world SOC workflows

---

## Limitations

* Requires integration for full automation (via n8n)
* Manual investigation still required for complex cases
* UI and features may require familiarisation

---

## Design Approach

The use of DFIR IRIS in this project follows key principles:

* **Automation-driven intake** – incidents created automatically from alerts
* **Structured investigations** – consistent handling of all incidents
* **Full visibility** – all actions and findings documented
* **Scalability** – suitable for multiple clients or environments

---

## Summary

DFIR IRIS provides the **incident management backbone** of the SOC.

It ensures that:

* Alerts are not lost or ignored
* Investigations follow a structured process
* All actions are documented and auditable

This transforms raw alerts into fully managed security incidents.

---
