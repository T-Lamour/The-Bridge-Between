# Use Case: Malicious File Download

## Scenario Overview

A user on a Windows endpoint downloads a file that matches a known malware hash. Wazuh's File Integrity Monitoring (FIM) detects the file creation event and triggers an alert. The SOC pipeline enriches the hash against threat intelligence sources, confirms the malware classification, creates an incident, and initiates containment before the file can be executed or spread laterally.

---

## Attack Description

| Field | Detail |
|---|---|
| **Attack Type** | Malware Delivery — Drive-by Download |
| **Target** | Windows 10 endpoint (`DESKTOP-FIN02`) |
| **User** | `s.patel` (Finance Department) |
| **File Name** | `invoice_april_2026.exe` |
| **File Path** | `C:\Users\s.patel\Downloads\invoice_april_2026.exe` |
| **File Hash (SHA256)** | `4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f` |
| **File Size** | 1.2 MB |
| **Download Source** | Phishing email link |
| **Time** | 2026-04-23 10:22:05 UTC |

The user received a phishing email containing a link to what appeared to be a supplier invoice. Clicking the link triggered an automatic download of a renamed Emotet dropper variant.

---

## Stage 1 — Detection (Wazuh)

Wazuh's **File Integrity Monitoring (FIM)** is configured to monitor the `Downloads`, `Desktop`, and `Temp` directories on all endpoints. The moment the file is written to disk, Wazuh captures the event.

### Rule Triggered

| Field | Value |
|---|---|
| **Rule ID** | `554` |
| **Rule Description** | `File added to monitored directory` |
| **Secondary Rule** | `87105` — `Wazuh-VirusTotal: File hash found in threat intelligence` |
| **MITRE ATT&CK** | T1566.002 — Spearphishing Link, T1204.002 — Malicious File |
| **Severity Level** | 12 (Critical) |

Wazuh FIM captures the SHA256 hash of every new file in monitored directories. Rule `87105` fires when FIM hash lookup returns a positive match from the integrated VirusTotal module.

### Wazuh Alert Payload (JSON)

```json
{
  "timestamp": "2026-04-23T10:22:07.334Z",
  "rule": {
    "id": "87105",
    "description": "Wazuh-VirusTotal: File hash found in threat intelligence",
    "level": 12,
    "mitre": {
      "technique": ["T1566.002", "T1204.002"],
      "tactic": ["Initial Access", "Execution"]
    }
  },
  "agent": {
    "id": "019",
    "name": "DESKTOP-FIN02",
    "ip": "10.0.1.42"
  },
  "syscheck": {
    "path": "C:\\Users\\s.patel\\Downloads\\invoice_april_2026.exe",
    "event": "added",
    "sha256_after": "4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f",
    "size_after": "1258496",
    "uname_after": "s.patel",
    "mtime_after": "2026-04-23T10:22:05Z"
  },
  "data": {
    "virustotal": {
      "found": 1,
      "positives": 54,
      "total": 72,
      "permalink": "https://www.virustotal.com/gui/file/4a8f2c1e..."
    }
  },
  "full_log": "FIM: New file detected. Path: C:\\Users\\s.patel\\Downloads\\invoice_april_2026.exe. SHA256: 4a8f2c1e... VT Positives: 54/72"
}
```

This payload is forwarded to n8n via **webhook**.

---

## Stage 2 — Enrichment (n8n)

n8n receives the alert and begins the enrichment workflow.

### Step 1 — Data Extraction

n8n extracts the following fields:

- File hash (SHA256): `4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f`
- File path: `C:\Users\s.patel\Downloads\invoice_april_2026.exe`
- File name: `invoice_april_2026.exe`
- Affected user: `s.patel`
- Affected host: `DESKTOP-FIN02`
- VirusTotal initial positives: `54/72`
- Timestamp: `2026-04-23T10:22:07Z`

---

### Step 2 — Threat Intelligence Lookup

#### VirusTotal (Full Hash Analysis)

```json
{
  "sha256": "4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f",
  "meaningful_name": "invoice_april_2026.exe",
  "type_description": "Win32 EXE",
  "malicious": 54,
  "suspicious": 6,
  "harmless": 0,
  "popular_threat_classification": {
    "suggested_threat_label": "trojan.emotet/dropper",
    "popular_threat_category": "trojan",
    "popular_threat_name": "Emotet"
  },
  "first_submission_date": "2026-04-21T08:11:00Z",
  "last_analysis_date": "2026-04-23T09:55:00Z"
}
```

#### MISP

```json
{
  "event_id": "1071",
  "event_info": "Emotet Resurgence Campaign — April 2026 — Phishing Invoices",
  "attribute_type": "sha256",
  "value": "4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f",
  "tags": ["tlp:amber", "emotet", "dropper", "phishing-invoice"],
  "related_attributes": [
    {
      "type": "domain",
      "value": "invoices-secure-docs.net",
      "comment": "Phishing download domain"
    },
    {
      "type": "ip-dst",
      "value": "45.132.227.88",
      "comment": "Emotet C2 server"
    }
  ]
}
```

MISP confirms the file is part of an active Emotet campaign using phishing invoice lures, and provides the associated C2 IP.

---

### Step 3 — Decision Logic

n8n evaluates the enriched data:

| Condition | Result |
|---|---|
| VirusTotal positives > 10 | True — 54/72 engines |
| Malware family identified | True — Emotet (trojan/dropper) |
| MISP match found | True — active campaign (TLP:AMBER) |
| File in user-writable directory | True — `Downloads` folder |
| File not yet executed (no process event) | True — window for containment exists |

**Decision: CRITICAL — Malware confirmed. Isolate host. Block C2 IP. Disable user session.**

---

## Stage 3 — Response Actions (n8n → pfSense / Microsoft Entra / Wazuh)

### Action 1 — Block C2 IP via pfSense

n8n calls the pfSense API to block the known Emotet C2 address identified via MISP:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_BLOCKLIST",
  "address": "45.132.227.88",
  "detail": "Emotet C2 — MISP Event #1071 — auto-blocked by n8n [2026-04-23T10:22:18Z]"
}
```

**Result:** Any callback attempts from `DESKTOP-FIN02` to the C2 server are dropped at the perimeter, preventing Emotet from establishing persistence or downloading secondary payloads.

---

### Action 2 — Block Phishing Download Domain via pfSense

n8n also blocks the phishing delivery domain to prevent other users downloading the same file:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_DOMAIN_BLOCKLIST",
  "address": "invoices-secure-docs.net",
  "detail": "Emotet phishing domain — MISP Event #1071 — auto-blocked [2026-04-23T10:22:19Z]"
}
```

---

### Action 3 — Disable User Account (Microsoft Entra)

n8n disables `s.patel`'s account to prevent the user interacting further with the system while investigation is underway:

```
PATCH /v1.0/users/s.patel@company.com
{
  "accountEnabled": false
}
```

**Result:** Account suspended pending host remediation.

---

### Action 4 — Update MISP

n8n records the new observed-data back into the MISP campaign event:

```json
{
  "event_id": "1071",
  "attribute": {
    "type": "hostname",
    "value": "DESKTOP-FIN02",
    "comment": "Host that downloaded Emotet dropper — s.patel — 2026-04-23 10:22 UTC"
  }
}
```

---

## Stage 4 — Incident Creation (DFIR IRIS)

n8n creates a structured case in DFIR IRIS.

### IRIS Case

| Field | Value |
|---|---|
| **Case Title** | Malware Detection — Emotet Dropper — DESKTOP-FIN02 |
| **Severity** | Critical |
| **Status** | Open |
| **Created By** | n8n (automated) |
| **Created At** | 2026-04-23 10:22:21 UTC |
| **Assigned To** | SOC Analyst (Tier 2) |

### Case Description

```
[AUTOMATED ALERT — SOC Pipeline]

Wazuh FIM detected a malicious file written to disk on DESKTOP-FIN02 by user s.patel.

FILE DETAILS:
- Name: invoice_april_2026.exe
- Path: C:\Users\s.patel\Downloads\
- SHA256: 4a8f2c1e9b3d6f0a7e5c2b9d1f4e8a3c6b0d5e2f9a7c4b1e8d3f6a0c5b2e9d7f
- Size: 1.2 MB

ENRICHMENT SUMMARY:
- VirusTotal: 54/72 malicious detections — classified as Emotet dropper (trojan)
- MISP Match: Event #1071 — Emotet Resurgence Campaign — April 2026 (TLP:AMBER)
- Associated C2: 45.132.227.88
- Delivery Domain: invoices-secure-docs.net

AUTOMATED ACTIONS TAKEN:
- C2 IP 45.132.227.88 blocked via pfSense
- Phishing domain invoices-secure-docs.net blocked via pfSense
- User account s.patel@company.com disabled via Microsoft Entra
- MISP event #1071 updated with affected host

INVESTIGATION REQUIRED:
- Determine if file was executed (check process creation events on DESKTOP-FIN02)
- Check for lateral movement or credential harvesting if Emotet executed
- Assess whether other users received the same phishing email
- Perform forensic triage or full reimaging of DESKTOP-FIN02
```

### IOCs Added to Case

| Type | Value | Source |
|---|---|---|
| SHA256 Hash | `4a8f2c1e...` | Wazuh FIM / VirusTotal / MISP |
| Malware Family | `Emotet` | VirusTotal |
| C2 IP | `45.132.227.88` | MISP Event #1071 |
| Phishing Domain | `invoices-secure-docs.net` | MISP Event #1071 |
| Affected Host | `DESKTOP-FIN02` | Wazuh Agent |
| Affected User | `s.patel` | Wazuh FIM |

### Analyst Tasks (Auto-Generated)

1. Check Wazuh process creation events on `DESKTOP-FIN02` to determine if `invoice_april_2026.exe` was executed
2. If executed — review for scheduled tasks, registry run keys, and network connections to C2
3. Search email gateway logs for the phishing email — identify other recipients
4. Quarantine `DESKTOP-FIN02` from the network if execution is confirmed
5. Perform forensic triage or reimage the endpoint
6. Reset credentials for `s.patel` before re-enabling the account
7. Brief all staff who received the phishing email
8. Close case and document findings

---

## Outcome Summary

| Stage | Action | Result |
|---|---|---|
| Detection | Wazuh FIM + rule 87105 | Malicious file identified on write to disk |
| Enrichment | VirusTotal / MISP | Confirmed Emotet, active campaign identified |
| Response | pfSense — C2 block | Outbound C2 callback prevented |
| Response | pfSense — domain block | Other users protected from same download |
| Response | Entra account disabled | User account locked pending investigation |
| Case Management | IRIS case created | Tier 2 analyst assigned for triage |

**Mean Time to Respond (MTTR): ~14 seconds from file creation to C2 block.**

---

## MITRE ATT&CK Mapping

| Technique | ID | Tactic |
|---|---|---|
| Spearphishing Link | T1566.002 | Initial Access |
| Malicious File | T1204.002 | Execution |
| Emotet — Dropper | T1587.001 | Resource Development |
| C2 Communication | T1071 | Command and Control |
| Credential Dumping (potential) | T1003 | Credential Access |

---
