# Use Case: Suspicious Login

## Scenario Overview

A valid user account successfully authenticates from an IP address flagged as malicious, at an unusual time, from a country inconsistent with the user's normal access patterns. Unlike a brute force attempt, this represents a potentially compromised credential — the attacker is already in. The SOC pipeline detects the anomaly, enriches the alert, and takes immediate protective action before damage can occur.

---

## Attack Description

| Field | Detail |
|---|---|
| **Attack Type** | Account Compromise — Credential Stuffing |
| **Target Account** | `j.harrison@company.com` |
| **Source IP** | `91.108.56.192` |
| **Source Country** | Ukraine |
| **Normal Access Pattern** | UK-based, business hours (08:00–18:00) |
| **Login Time** | 2026-04-23 03:47:11 UTC |
| **Login Method** | Microsoft 365 (Azure AD / Entra) |
| **Authentication Result** | **Success** |

The attacker uses credentials likely obtained from a credential stuffing list or phishing campaign. The account has no MFA enrolled, allowing the login to succeed.

---

## Stage 1 — Detection (Wazuh)

Wazuh ingests Microsoft Entra sign-in logs forwarded via the Wazuh agent on the Microsoft 365 log collector.

### Rule Triggered

| Field | Value |
|---|---|
| **Rule ID** | `91530` |
| **Rule Description** | `Microsoft Entra: Successful login from suspicious IP` |
| **MITRE ATT&CK** | T1078 — Valid Accounts |
| **Severity Level** | 12 (Critical) |

This custom rule fires when a successful Entra authentication event (Event ID `Sign-in success`) is received from an IP with a prior AbuseIPDB or MISP match, or from a country not in the user's baseline.

### Wazuh Alert Payload (JSON)

```json
{
  "timestamp": "2026-04-23T03:47:11.801Z",
  "rule": {
    "id": "91530",
    "description": "Microsoft Entra: Successful login from suspicious IP",
    "level": 12,
    "mitre": {
      "technique": ["T1078"],
      "tactic": ["Defence Evasion", "Persistence", "Initial Access"]
    }
  },
  "agent": {
    "id": "012",
    "name": "M365-LogCollector",
    "ip": "10.0.1.20"
  },
  "data": {
    "azure": {
      "signinlogs": {
        "userPrincipalName": "j.harrison@company.com",
        "ipAddress": "91.108.56.192",
        "location": {
          "city": "Kyiv",
          "countryOrRegion": "UA"
        },
        "status": {
          "errorCode": 0,
          "failureReason": null
        },
        "appDisplayName": "Microsoft 365",
        "riskLevelDuringSignIn": "high",
        "authenticationRequirement": "singleFactorAuthentication"
      }
    }
  },
  "full_log": "Successful sign-in: j.harrison@company.com from 91.108.56.192 (Kyiv, UA). Risk: high. MFA: not required."
}
```

This payload is forwarded to n8n via **webhook**.

---

## Stage 2 — Enrichment (n8n)

n8n receives the alert and begins the enrichment workflow.

### Step 1 — Data Extraction

n8n extracts the following fields:

- Source IP: `91.108.56.192`
- User: `j.harrison@company.com`
- Country: `UA` (Ukraine)
- Login app: `Microsoft 365`
- MFA used: `No`
- Entra risk level: `high`
- Timestamp: `2026-04-23T03:47:11Z`

---

### Step 2 — Threat Intelligence Lookup

#### AbuseIPDB

```json
{
  "ipAddress": "91.108.56.192",
  "abuseConfidenceScore": 84,
  "totalReports": 76,
  "lastReportedAt": "2026-04-21T14:22:00Z",
  "usageType": "Data Center / Web Hosting",
  "countryCode": "UA",
  "isTor": false
}
```

#### VirusTotal

```json
{
  "ip": "91.108.56.192",
  "malicious": 8,
  "suspicious": 2,
  "harmless": 4,
  "last_analysis_date": "2026-04-22T09:00:00Z"
}
```

#### MISP

```json
{
  "event_id": "1058",
  "event_info": "Credential Stuffing Campaign — M365 Accounts",
  "attribute_type": "ip-src",
  "value": "91.108.56.192",
  "tags": ["tlp:red", "credential-stuffing", "microsoft-365"]
}
```

MISP confirms this IP is actively associated with a credential stuffing campaign targeting Microsoft 365 accounts.

---

### Step 3 — Decision Logic

n8n evaluates the enriched data:

| Condition | Result |
|---|---|
| Login successful | True — active session exists |
| AbuseIPDB score > 80 | True |
| VirusTotal malicious > 5 | True |
| MISP match found | True (TLP:RED — credential stuffing) |
| Country anomaly (non-UK login) | True |
| MFA not used | True |

**Decision: CRITICAL — Account compromise likely. Immediately revoke sessions and disable account.**

---

## Stage 3 — Response Actions (n8n → Microsoft Entra / pfSense)

### Action 1 — Revoke All Active Sessions

n8n immediately revokes all active tokens for the compromised account via Microsoft Graph:

```
POST /v1.0/users/j.harrison@company.com/revokeSignInSessions
```

**Result:** Active session established from `91.108.56.192` terminated within seconds.

---

### Action 2 — Disable User Account

n8n disables the account to prevent re-authentication:

```
PATCH /v1.0/users/j.harrison@company.com
{
  "accountEnabled": false
}
```

**Result:** Account locked until analyst review and credential reset.

---

### Action 3 — Block IP via pfSense

n8n calls the pfSense API to block the source IP:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_BLOCKLIST",
  "address": "91.108.56.192",
  "detail": "Credential stuffing source — auto-blocked by n8n [2026-04-23T03:47:19Z]"
}
```

**Result:** IP blocked at the network perimeter.

---

### Action 4 — Update MISP

n8n correlates findings back to MISP event `#1058`:

```json
{
  "event_id": "1058",
  "attribute": {
    "type": "ip-src",
    "value": "91.108.56.192",
    "comment": "Used in successful M365 login against j.harrison@company.com — 2026-04-23 03:47 UTC. Account disabled and sessions revoked."
  }
}
```

---

## Stage 4 — Incident Creation (DFIR IRIS)

n8n creates a structured case in DFIR IRIS.

### IRIS Case

| Field | Value |
|---|---|
| **Case Title** | Suspicious Login — Account Compromise — j.harrison@company.com |
| **Severity** | Critical |
| **Status** | Open |
| **Created By** | n8n (automated) |
| **Created At** | 2026-04-23 03:47:22 UTC |
| **Assigned To** | SOC Analyst (Tier 2) |

### Case Description

```
[AUTOMATED ALERT — SOC Pipeline]

Wazuh detected a successful Microsoft 365 login for j.harrison@company.com from 
91.108.56.192 (Kyiv, Ukraine) at 03:47 UTC. This is inconsistent with the user's 
normal access patterns (UK-based, business hours). No MFA was enforced.

ENRICHMENT SUMMARY:
- AbuseIPDB Score: 84/100 (76 prior reports)
- VirusTotal: 8 malicious vendor detections
- MISP Match: Event #1058 — Credential Stuffing Campaign targeting M365 (TLP:RED)
- Entra Risk Level: High

AUTOMATED ACTIONS TAKEN:
- Active sessions revoked via Microsoft Graph
- Account j.harrison@company.com disabled via Microsoft Entra
- IP 91.108.56.192 blocked via pfSense
- MISP event #1058 updated

INVESTIGATION REQUIRED:
- Determine if attacker accessed any data during the active session
- Review audit logs for file access, email exfiltration, forwarding rules
- Identify how credentials were obtained (phishing, prior breach)
- Coordinate with user to reset credentials and enrol MFA before re-enabling account
```

### IOCs Added to Case

| Type | Value | Source |
|---|---|---|
| IP Address | `91.108.56.192` | AbuseIPDB / VirusTotal / MISP |
| User Account | `j.harrison@company.com` | Entra Sign-in Log |
| Campaign | `Credential Stuffing — M365` | MISP Event #1058 |

### Analyst Tasks (Auto-Generated)

1. Review Microsoft 365 audit logs for activity during the compromised session
2. Check for new mail forwarding rules or inbox rules created by the attacker
3. Check for any file downloads or SharePoint access during the session
4. Identify credential source — check Have I Been Pwned or internal phishing logs
5. Reset credentials for `j.harrison@company.com` and enforce MFA enrolment
6. Re-enable account only after credential reset and MFA confirmation
7. Brief user on the incident and phishing awareness
8. Close case and document findings

---

## Outcome Summary

| Stage | Action | Result |
|---|---|---|
| Detection | Wazuh rule 91530 triggered | Alert generated on successful anomalous login |
| Enrichment | AbuseIPDB / VirusTotal / MISP | IP linked to active credential stuffing campaign |
| Response | Sessions revoked | Attacker access terminated |
| Response | Account disabled | Re-authentication prevented |
| Response | pfSense block | IP blocked at perimeter |
| Case Management | IRIS case created | Tier 2 analyst assigned for investigation |

**Mean Time to Respond (MTTR): ~11 seconds from login detection to session revocation.**

---

## MITRE ATT&CK Mapping

| Technique | ID | Tactic |
|---|---|---|
| Valid Accounts | T1078 | Initial Access / Persistence |
| Credential Stuffing | T1110.004 | Credential Access |
| Email Collection | T1114 | Collection (potential follow-on) |
| Email Forwarding Rules | T1114.003 | Collection (potential follow-on) |

---
