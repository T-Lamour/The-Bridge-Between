# Use Case: Suspicious Network Traffic

## Scenario Overview

An internal endpoint begins generating unusual outbound network traffic — making periodic beaconing connections to an external IP on a non-standard port. pfSense's Suricata IDS detects the pattern, logs are forwarded to Wazuh, and the automated pipeline identifies a likely Command and Control (C2) connection, blocks the traffic, and creates an incident for analyst investigation.

---

## Attack Description

| Field | Detail |
|---|---|
| **Attack Type** | C2 Beaconing — Post-Exploitation |
| **Source Host** | `DESKTOP-DEV04` (`10.0.2.31`) |
| **Destination IP** | `194.165.16.78` |
| **Destination Port** | `4444` |
| **Protocol** | TCP |
| **Traffic Pattern** | Regular 60-second interval beaconing |
| **Session Duration** | 22 minutes before detection |
| **Bytes Sent** | 48 KB outbound |
| **Bytes Received** | 312 KB inbound |
| **Time First Seen** | 2026-04-23 14:08:42 UTC |
| **Time Detected** | 2026-04-23 14:30:11 UTC |

The endpoint has been compromised — likely via the malicious file download described in a prior incident — and a Meterpreter shell is now beaconing home on port 4444. The consistent interval and small outbound/large inbound ratio are characteristic of C2 check-in behaviour.

---

## Stage 1 — Detection

Detection occurs at two points in the pipeline simultaneously.

### Detection Point A — pfSense (Suricata IDS)

Suricata's ruleset fires on the combination of known-malicious destination IP and the use of port 4444 (a well-known Metasploit default listener port).

**Suricata Rule Match:**

```
alert tcp $HOME_NET any -> $EXTERNAL_NET 4444 (
  msg:"ET MALWARE Possible Meterpreter/Reverse Shell Beacon";
  flow:established,to_server;
  threshold:type both,track by_src,count 5,seconds 300;
  classtype:trojan-activity;
  sid:2019284; rev:5;
)
```

pfSense logs the event and forwards it to Wazuh via syslog.

---

### Detection Point B — Wazuh

Wazuh correlates the pfSense Suricata alert with internal network logs and fires its own rule.

#### Rule Triggered

| Field | Value |
|---|---|
| **Rule ID** | `86601` |
| **Rule Description** | `Suricata: High-severity alert — trojan-activity detected` |
| **MITRE ATT&CK** | T1071.001 — Web Protocols C2, T1571 — Non-Standard Port |
| **Severity Level** | 12 (Critical) |

### Wazuh Alert Payload (JSON)

```json
{
  "timestamp": "2026-04-23T14:30:11.992Z",
  "rule": {
    "id": "86601",
    "description": "Suricata: High-severity alert — trojan-activity detected",
    "level": 12,
    "mitre": {
      "technique": ["T1071.001", "T1571"],
      "tactic": ["Command and Control"]
    }
  },
  "agent": {
    "id": "001",
    "name": "pfSense-Firewall",
    "ip": "10.0.0.1"
  },
  "data": {
    "suricata": {
      "alert": {
        "action": "allowed",
        "signature_id": 2019284,
        "signature": "ET MALWARE Possible Meterpreter/Reverse Shell Beacon",
        "category": "trojan-activity",
        "severity": 1
      },
      "src_ip": "10.0.2.31",
      "src_port": 49823,
      "dest_ip": "194.165.16.78",
      "dest_port": 4444,
      "proto": "TCP",
      "app_proto": "failed",
      "flow": {
        "pkts_toserver": 264,
        "pkts_toclient": 1842,
        "bytes_toserver": 49152,
        "bytes_toclient": 319488
      }
    }
  },
  "full_log": "Suricata ALERT: ET MALWARE Possible Meterpreter/Reverse Shell Beacon. SRC: 10.0.2.31:49823 -> DST: 194.165.16.78:4444. Severity: 1. Category: trojan-activity."
}
```

This payload is forwarded to n8n via **webhook**.

---

## Stage 2 — Enrichment (n8n)

n8n receives the alert and begins the enrichment workflow.

### Step 1 — Data Extraction

n8n extracts the following fields:

- Source internal IP: `10.0.2.31`
- Source hostname: `DESKTOP-DEV04` (resolved via internal DNS lookup)
- Destination IP: `194.165.16.78`
- Destination port: `4444`
- Suricata signature: `ET MALWARE Possible Meterpreter/Reverse Shell Beacon`
- Alert severity: `1` (Critical)
- Flow bytes inbound: `312 KB` (indicative of payload delivery)

---

### Step 2 — Threat Intelligence Lookup

#### AbuseIPDB

```json
{
  "ipAddress": "194.165.16.78",
  "abuseConfidenceScore": 91,
  "totalReports": 189,
  "lastReportedAt": "2026-04-23T13:44:00Z",
  "usageType": "Data Center / Web Hosting",
  "countryCode": "NL",
  "isTor": false
}
```

#### VirusTotal

```json
{
  "ip": "194.165.16.78",
  "malicious": 19,
  "suspicious": 4,
  "harmless": 1,
  "last_analysis_date": "2026-04-23T12:00:00Z",
  "tags": ["c2", "meterpreter", "cobalt-strike"]
}
```

#### MISP

```json
{
  "event_id": "1083",
  "event_info": "Active C2 Infrastructure — Meterpreter / Cobalt Strike — April 2026",
  "attribute_type": "ip-dst",
  "value": "194.165.16.78",
  "tags": ["tlp:red", "c2", "meterpreter", "post-exploitation"],
  "related_attributes": [
    {
      "type": "port",
      "value": "4444",
      "comment": "Confirmed C2 listener port"
    },
    {
      "type": "hostname",
      "value": "cdn-update.securehosting-nl.com",
      "comment": "Associated C2 domain"
    }
  ]
}
```

MISP confirms this IP is an active C2 node associated with post-exploitation frameworks (Meterpreter / Cobalt Strike).

---

### Step 3 — Decision Logic

n8n evaluates the enriched data:

| Condition | Result |
|---|---|
| AbuseIPDB score > 80 | True — 91/100 |
| VirusTotal malicious > 5 | True — 19 detections, tagged C2 |
| MISP match — C2 infrastructure | True (TLP:RED) |
| Suricata severity == 1 | True — highest priority |
| Traffic pattern consistent with beaconing | True — 60s intervals, 22 minutes |
| Inbound bytes >> outbound (payload delivery pattern) | True — 312 KB inbound |

**Decision: CRITICAL — Active C2 session in progress. Immediately block IP and isolate host.**

---

## Stage 3 — Response Actions (n8n → pfSense / Microsoft Entra)

### Action 1 — Emergency IP Block via pfSense

n8n calls the pfSense API to immediately block the C2 IP, terminating the active session:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_BLOCKLIST",
  "address": "194.165.16.78",
  "detail": "Active C2 — Meterpreter beacon — MISP Event #1083 — auto-blocked by n8n [2026-04-23T14:30:19Z]"
}
```

**Result:** Active C2 session terminated. Attacker loses interactive access to `DESKTOP-DEV04`.

---

### Action 2 — Block Associated C2 Domain via pfSense

n8n also blocks the associated domain from MISP to prevent re-establishment via DNS:

```
POST /api/v1/firewall/alias/entry
{
  "name": "SOC_DOMAIN_BLOCKLIST",
  "address": "cdn-update.securehosting-nl.com",
  "detail": "C2 domain — MISP Event #1083 — auto-blocked [2026-04-23T14:30:20Z]"
}
```

---

### Action 3 — Disable Logged-in User Account (Microsoft Entra)

n8n identifies the active user session on `DESKTOP-DEV04` and disables the account:

```
PATCH /v1.0/users/d.okonkwo@company.com
{
  "accountEnabled": false
}
```

**Result:** User account locked to prevent credential reuse if the attacker harvested them during the active session.

---

### Action 4 — Update MISP

n8n records the observed compromise back into MISP event `#1083`:

```json
{
  "event_id": "1083",
  "attribute": {
    "type": "hostname",
    "value": "DESKTOP-DEV04",
    "comment": "Compromised host — active Meterpreter C2 beacon observed — 2026-04-23 14:08-14:30 UTC. User: d.okonkwo"
  }
}
```

---

## Stage 4 — Incident Creation (DFIR IRIS)

n8n creates a structured case in DFIR IRIS.

### IRIS Case

| Field | Value |
|---|---|
| **Case Title** | C2 Beaconing Detected — Meterpreter — DESKTOP-DEV04 |
| **Severity** | Critical |
| **Status** | Open |
| **Created By** | n8n (automated) |
| **Created At** | 2026-04-23 14:30:23 UTC |
| **Assigned To** | SOC Analyst (Tier 2 / IR Lead) |

### Case Description

```
[AUTOMATED ALERT — SOC Pipeline]

Suricata on pfSense detected active C2 beaconing from DESKTOP-DEV04 (10.0.2.31) 
to 194.165.16.78:4444. The traffic pattern is consistent with a Meterpreter reverse 
shell — 60-second beacon intervals, 22 minutes of active session time, and 312 KB 
of inbound data (likely payload delivery).

TIMELINE:
- 14:08:42 UTC — First beacon observed
- 14:30:11 UTC — Suricata alert triggered and detected by Wazuh
- 14:30:19 UTC — C2 IP blocked via pfSense (session terminated)

ENRICHMENT SUMMARY:
- AbuseIPDB Score: 91/100 (189 prior reports)
- VirusTotal: 19 malicious detections — tagged as C2 / Meterpreter / Cobalt Strike
- MISP Match: Event #1083 — Active C2 Infrastructure, April 2026 (TLP:RED)
- Associated C2 domain: cdn-update.securehosting-nl.com

AUTOMATED ACTIONS TAKEN:
- C2 IP 194.165.16.78 blocked via pfSense (active session terminated)
- C2 domain cdn-update.securehosting-nl.com blocked via pfSense
- User account d.okonkwo@company.com disabled via Microsoft Entra
- MISP event #1083 updated with affected host

CRITICAL NOTE:
An active session ran for approximately 22 minutes before detection. 312 KB of data 
was received by the endpoint during this window. Assume host is compromised and treat 
as a full incident response engagement.

INVESTIGATION REQUIRED:
- Determine initial compromise vector (phishing, malicious download, exploitation)
- Assess what commands were executed during the 22-minute active session
- Check for credential harvesting, lateral movement, or data exfiltration
- Full forensic triage or reimaging of DESKTOP-DEV04 required
- Review all other hosts for similar beaconing patterns
```

### IOCs Added to Case

| Type | Value | Source |
|---|---|---|
| C2 IP | `194.165.16.78` | AbuseIPDB / VirusTotal / MISP |
| C2 Port | `4444` | Suricata / MISP |
| C2 Domain | `cdn-update.securehosting-nl.com` | MISP Event #1083 |
| Affected Host | `DESKTOP-DEV04` (`10.0.2.31`) | pfSense / Wazuh |
| Affected User | `d.okonkwo@company.com` | Active session |
| Malware Family | `Meterpreter / Cobalt Strike` | VirusTotal / MISP |

### Analyst Tasks (Auto-Generated)

1. Review pfSense flow logs for the full 22-minute session — extract all data transferred
2. Run Wazuh process and command execution logs on `DESKTOP-DEV04` for the session window
3. Check for evidence of credential harvesting (LSASS access, credential files)
4. Check for lateral movement — review authentication logs on other internal hosts
5. Check for data exfiltration — review outbound traffic volume across all hosts
6. Search all endpoints for similar beaconing patterns to the same or related C2 IPs
7. Identify initial compromise vector — cross-reference with prior incidents (malicious file download)
8. Perform full forensic triage or reimage `DESKTOP-DEV04`
9. Reset credentials for `d.okonkwo` before re-enabling account
10. Close case and document findings — update threat intelligence in MISP

---

## Outcome Summary

| Stage | Action | Result |
|---|---|---|
| Detection | Suricata (pfSense) + Wazuh rule 86601 | Active C2 beacon identified |
| Enrichment | AbuseIPDB / VirusTotal / MISP | Confirmed Meterpreter C2 infrastructure |
| Response | pfSense — C2 IP block | Active attacker session terminated |
| Response | pfSense — C2 domain block | Re-establishment via DNS prevented |
| Response | Entra account disabled | Credential reuse prevented |
| Case Management | IRIS case created | IR Lead assigned for full incident response |

**Mean Time to Respond (MTTR): ~8 seconds from Wazuh alert to C2 IP block.**

**Note:** The 22-minute dwell time before detection represents a key lesson — Suricata's threshold-based rule required 5 beacon events over 5 minutes before firing. Reducing this threshold or adding a first-occurrence rule would decrease detection latency for future incidents.

---

## MITRE ATT&CK Mapping

| Technique | ID | Tactic |
|---|---|---|
| C2 — Web Protocols | T1071.001 | Command and Control |
| Non-Standard Port | T1571 | Command and Control |
| Remote Access Software | T1219 | Command and Control |
| OS Credential Dumping (potential) | T1003 | Credential Access |
| Lateral Movement — Pass the Hash (potential) | T1550.002 | Lateral Movement |
| Exfiltration Over C2 Channel (potential) | T1041 | Exfiltration |

---
