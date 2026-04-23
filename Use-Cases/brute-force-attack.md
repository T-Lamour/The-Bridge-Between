# Use Case: Brute Force Attack

## Scenario Overview

An external attacker attempts to gain unauthorised access to a Windows endpoint by repeatedly trying common credentials against the RDP service. Wazuh detects the pattern of repeated authentication failures, triggers an alert, and the automated pipeline enriches, escalates, and responds — all without analyst intervention.

---

## Attack Description

| Field | Detail |
|---|---|
| **Attack Type** | RDP Brute Force |
| **Target** | Windows 10 endpoint (`DESKTOP-HR01`) |
| **Source IP** | `185.220.101.47` |
| **Source Country** | Russia |
| **Time** | 2026-04-23 02:14:38 UTC |
| **Duration** | ~4 minutes |
| **Attempts** | 87 failed login attempts |

The attacker cycles through a credential list targeting the local Administrator account and several common usernames (`admin`, `user`, `test`). The volume and rate of failures exceed the brute force threshold defined in the Wazuh ruleset.

---

## Stage 1 — Detection (Wazuh)

Wazuh monitors Windows Security Event Logs forwarded by the agent running on `DESKTOP-HR01`.

### Rule Triggered

| Field | Value |
|---|---|
| **Rule ID** | `60106` |
| **Rule Description** | `Windows: Multiple authentication failures` |
| **MITRE ATT&CK** | T1110 — Brute Force |
| **Severity Level** | 10 (Critical) |

Wazuh fires this rule after detecting more than 8 failed logon events (Event ID 4625) from the same source IP within a 2-minute window.

### Wazuh Alert Payload (JSON)

```json
{
  "timestamp": "2026-04-23T02:14:38.412Z",
  "rule": {
    "id": "60106",
    "description": "Windows: Multiple authentication failures",
    "level": 10,
    "mitre": {
      "technique": ["T1110"],
      "tactic": ["Credential Access"]
    }
  },
  "agent": {
    "id": "007",
    "name": "DESKTOP-HR01",
    "ip": "10.0.1.15"
  },
  "data": {
    "win": {
      "eventdata": {
        "ipAddress": "185.220.101.47",
        "targetUserName": "Administrator",
        "logonType": "10",
        "failureReason": "Unknown user name or bad password"
      },
      "system": {
        "eventID": "4625",
        "channel": "Security"
      }
    }
  },
  "full_log": "An account failed to log on. Source IP: 185.220.101.47. Logon Type: 10 (RemoteInteractive). Failure count: 87"
}
```

This payload is forwarded to n8n via **webhook**.

---

## Stage 2 — Enrichment (n8n)

n8n receives the alert and begins the enrichment workflow.

### Step 1 — Data Extraction

n8n extracts the following fields from the Wazuh payload:

- Source IP: `185.220.101.47`
- Target user: `Administrator`
- Agent hostname: `DESKTOP-HR01`
- Rule level: `10`
- Timestamp: `2026-04-23T02:14:38Z`

---

### Step 2 — Threat Intelligence Lookup

#### AbuseIPDB

```json
{
  "ipAddress": "185.220.101.47",
  "abuseConfidenceScore": 97,
  "totalReports": 312,
  "lastReportedAt": "2026-04-23T01:55:00Z",
  "usageType": "Tor Exit Node",
  "countryCode": "RU",
  "isTor": true
}
```

#### VirusTotal

```json
{
  "ip": "185.220.101.47",
  "malicious": 14,
  "suspicious": 3,
  "harmless": 0,
  "last_analysis_date": "2026-04-22T18:30:00Z"
}
```

#### MISP

MISP returns a match against an existing event:

```json
{
  "event_id": "1042",
  "event_info": "Threat Actor: RDP-Scanning Botnet — Eastern Europe",
  "attribute_type": "ip-src",
  "value": "185.220.101.47",
  "tags": ["tlp:amber", "brute-force", "rdp"]
}
```

---

### Step 3 — Decision Logic

n8n evaluates the enriched data:

| Condition | Result |
|---|---|
| AbuseIPDB score > 80 | True — IP confirmed malicious |
| VirusTotal malicious detections > 5 | True |
| MISP match found | True |
| Rule level >= 10 | True |

**Decision: CRITICAL — Trigger automated response and create IRIS incident.**

---

## Stage 3 — Response Actions (n8n → pfSense / Microsoft Entra)

Based on the decision logic, n8n triggers the following automated actions:

### Action 1 — Block IP via pfSense

n8n calls the pfSense API to add `185.220.101.47` to the blocklist:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_BLOCKLIST",
  "address": "185.220.101.47",
  "detail": "Brute force source — auto-blocked by n8n [2026-04-23T02:14:45Z]"
}
```

**Result:** All traffic from `185.220.101.47` dropped at the perimeter.

---

### Action 2 — Disable Administrator Account (Microsoft Entra)

As a precautionary measure, n8n disables the targeted account via the Microsoft Graph API:

```
PATCH /v1.0/users/administrator@company.com
{
  "accountEnabled": false
}
```

**Result:** Account suspended pending analyst review.

---

### Action 3 — Update MISP

n8n adds the IP as a new attribute to the existing MISP event and tags it with the current incident reference:

```json
{
  "event_id": "1042",
  "attribute": {
    "type": "ip-src",
    "value": "185.220.101.47",
    "comment": "Confirmed brute force source — RDP, DESKTOP-HR01, 2026-04-23"
  }
}
```

---

## Stage 4 — Incident Creation (DFIR IRIS)

n8n creates a structured case in DFIR IRIS with all enriched context attached.

### IRIS Case

| Field | Value |
|---|---|
| **Case Title** | Brute Force Attack — RDP — DESKTOP-HR01 |
| **Severity** | Critical |
| **Status** | Open |
| **Created By** | n8n (automated) |
| **Created At** | 2026-04-23 02:14:47 UTC |
| **Assigned To** | SOC Analyst (Tier 1) |

### Case Description

```
[AUTOMATED ALERT — SOC Pipeline]

Wazuh detected 87 failed RDP authentication attempts against DESKTOP-HR01 
over a 4-minute window originating from 185.220.101.47.

ENRICHMENT SUMMARY:
- AbuseIPDB Score: 97/100 (312 prior reports, Tor Exit Node)
- VirusTotal: 14 malicious vendor detections
- MISP Match: Event #1042 — RDP-Scanning Botnet, Eastern Europe (TLP:AMBER)

AUTOMATED ACTIONS TAKEN:
- IP 185.220.101.47 blocked via pfSense
- Administrator account disabled via Microsoft Entra
- MISP event #1042 updated with new attribute

INVESTIGATION REQUIRED:
- Confirm no successful logins occurred prior to detection
- Review account activity for Administrator on DESKTOP-HR01
- Assess exposure of RDP service and consider restricting to VPN-only access
```

### IOCs Added to Case

| Type | Value | Source |
|---|---|---|
| IP Address | `185.220.101.47` | AbuseIPDB / VirusTotal / MISP |
| Hostname | `DESKTOP-HR01` | Wazuh Agent |
| Username | `Administrator` | Windows Event Log |

### Analyst Tasks (Auto-Generated)

1. Confirm no successful logins occurred before detection threshold was reached
2. Review Windows Security Event Logs on `DESKTOP-HR01` for the 30 minutes prior to the alert
3. Re-enable Administrator account once scope is confirmed clear
4. Assess whether RDP should remain exposed or be restricted behind VPN
5. Close case and document findings

---

## Outcome Summary

| Stage | Action | Result |
|---|---|---|
| Detection | Wazuh rule 60106 triggered | Alert generated within seconds |
| Enrichment | AbuseIPDB / VirusTotal / MISP | IP confirmed malicious |
| Response | pfSense block | Attack traffic dropped at perimeter |
| Response | Entra account disabled | Target account protected |
| Case Management | IRIS case created | Analyst assigned for review |

**Mean Time to Respond (MTTR): ~9 seconds from alert generation to IP block.**

---

## MITRE ATT&CK Mapping

| Technique | ID | Tactic |
|---|---|---|
| Brute Force | T1110 | Credential Access |
| Valid Accounts | T1078 | Defence Evasion / Persistence |
| Remote Services — RDP | T1021.001 | Lateral Movement |

---
