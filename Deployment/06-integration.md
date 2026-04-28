# Integration — Connecting the Full Pipeline

## Overview

This guide wires the five deployed components into a functioning SOC pipeline and validates it with a simulated brute force attack. By the end of this guide:

* All n8n credentials are verified
* The brute force detection workflow is active
* A simulated attack produces a confirmed IRIS case, a MISP IOC, and an OPNsense block — in under 15 seconds
* The SOC is operational

This is the payoff guide. If any component was misconfigured in the previous steps, the end-to-end test will reveal it cleanly.

---

## 1. Before You Begin

Confirm the status of all five VMs before proceeding:

| VM | IP | Check |
| -- | -- | ----- |
| OPNsense | 10.10.1.254 | Suricata running, syslog forwarding active |
| Wazuh | 10.10.10.10 | All containers healthy, agent enrolled |
| n8n | 10.10.10.20 | Container running, all 6 credentials saved |
| MISP | 10.10.10.30 | Container running, feeds fetched, API key in n8n |
| DFIR IRIS | 10.10.10.40 | Container running, customer created, API key in n8n |

If any credential in n8n still shows an error icon, resolve it now. A failed API call mid-workflow will silently drop the case creation or IOC write without failing the entire workflow — the kind of subtle bug that is invisible until you look at IRIS and find it empty.

---

## 2. Verify All Credentials

In n8n, navigate to **Settings > Credentials**. For each credential, click the three-dot menu and select **Test**. All six must pass:

| Credential | Expected Test Result |
| ---------- | -------------------- |
| Wazuh API | `200 OK` from `https://10.10.10.10:55000/` |
| VirusTotal | `200 OK` with quota info |
| AbuseIPDB | `200 OK` with check result |
| MISP | `200 OK` with response object |
| DFIR IRIS | `200 OK` with `"status": "success"` |
| OPNsense API | `200 OK` from OPNsense API |

Any failure here must be resolved before the integration workflow will function reliably.

---

## 3. Import the Brute Force Workflow

### 3.1 Workflow Overview

The brute force workflow handles Wazuh rule `60106` — multiple failed authentication attempts. It follows this path:

```text
Webhook (Wazuh alert arrives)
    │
Filter (rule.id == 60106 OR rule.level >= 10)
    │
Set (extract srcip, agent.name, timestamp, rule.description)
    │
    ├── HTTP Request → AbuseIPDB (confidence score, ISP, country)
    │
    └── HTTP Request → VirusTotal (malicious detections, last analysis)
            │
IF (AbuseIPDB confidence > 50 OR VirusTotal detections > 3)
    │
    ├── HTTP Request → MISP (check existing IOCs for srcip)
    │
    ├── HTTP Request → DFIR IRIS (create case with enrichment data)
    │
    ├── HTTP Request → DFIR IRIS (add srcip as IOC to case)
    │
    ├── HTTP Request → DFIR IRIS (add agent endpoint as asset)
    │
    ├── HTTP Request → MISP (create event if not already present)
    │
    └── HTTP Request → OPNsense API (add block rule for srcip)
```

### 3.2 Create the Workflow in n8n

In n8n, create a new workflow named `SOC - Brute Force Detection`.

Build the following nodes in sequence:

#### Node 1: Webhook (Trigger)

* Type: Webhook
* Method: POST
* Path: `wazuh-alerts`
* Response: Immediately
* This node is already configured if you completed `03-n8n.md`. If not, create it now.

#### Node 2: Filter — Route by Rule

* Type: IF
* Condition: `{{ $json.rule.id }}` equals `60106` OR `{{ $json.rule.level }}` is greater than or equal to `10`

This prevents every low-severity Wazuh alert from triggering enrichment. Alerts that do not match fall through to a No-Op node and are discarded.

#### Node 3: Set — Extract Fields

* Type: Set (Manual Mapping mode)

| Output Field | Value |
| ------------ | ----- |
| `srcip` | `{{ $json.data.srcip }}` |
| `agent_name` | `{{ $json.agent.name }}` |
| `agent_ip` | `{{ $json.agent.ip }}` |
| `rule_id` | `{{ $json.rule.id }}` |
| `rule_description` | `{{ $json.rule.description }}` |
| `rule_level` | `{{ $json.rule.level }}` |
| `timestamp` | `{{ $json.timestamp }}` |

#### Node 4a: AbuseIPDB Lookup

* Type: HTTP Request
* Method: GET
* URL: `https://api.abuseipdb.com/api/v2/check`
* Query parameters: `ipAddress={{ $json.srcip }}&maxAgeInDays=90&verbose`
* Credential: `AbuseIPDB`

#### Node 4b: VirusTotal Lookup

* Type: HTTP Request
* Method: GET
* URL: `https://www.virustotal.com/api/v3/ip_addresses/{{ $json.srcip }}`
* Credential: `VirusTotal`
* Run in parallel with Node 4a using the **Merge** node after both complete

#### Node 5: Merge Enrichment Results

* Type: Merge
* Mode: Merge By Index
* This node combines the AbuseIPDB and VirusTotal responses into a single item

#### Node 6: Decision — Is it Malicious?

* Type: IF
* Condition A: `{{ $json.abuseipdb.data.abuseConfidenceScore }}` greater than `50`
* OR Condition B: `{{ $json.virustotal.data.attributes.last_analysis_stats.malicious }}` greater than `3`

If neither condition is met, the workflow ends and no case is created. Log the event to a Slack or email node if you want visibility on discarded alerts.

#### Node 7: Check MISP for Existing IOC

* Type: HTTP Request
* Method: POST
* URL: `https://10.10.10.30/attributes/restSearch`
* Credential: `MISP`
* Body (JSON):
```json
{
  "returnFormat": "json",
  "value": "{{ $json.srcip }}",
  "type": "ip-src"
}
```

#### Node 8: Create IRIS Case

* Type: HTTP Request
* Method: POST
* URL: `https://10.10.10.40/api/v1/cases/add`
* Credential: `DFIR IRIS`
* Body (JSON):
```json
{
  "case_name": "BRF-{{ $now.toFormat('yyyyMMdd-HHmm') }} - Brute Force from {{ $json.srcip }}",
  "case_description": "Wazuh rule {{ $json.rule_id }} | Level {{ $json.rule_level }} | Agent: {{ $json.agent_name }}\n\nAbuseIPDB Confidence: {{ $json.abuseipdb.data.abuseConfidenceScore }}%\nVirusTotal Malicious: {{ $json.virustotal.data.attributes.last_analysis_stats.malicious }}\nCountry: {{ $json.abuseipdb.data.countryCode }}\nISP: {{ $json.abuseipdb.data.isp }}",
  "case_customer": 1,
  "case_template_id": 1,
  "case_soc_id": "WZ-{{ $json.rule_id }}-{{ $now.toUnixInteger() }}"
}
```

Store the returned `case_id` from the response for the subsequent nodes.

#### Node 9: Add IOC to IRIS Case

* Type: HTTP Request
* Method: POST
* URL: `https://10.10.10.40/api/v1/case/ioc/add`
* Credential: `DFIR IRIS`
* Body (JSON):
```json
{
  "ioc_value": "{{ $json.srcip }}",
  "ioc_type_id": 76,
  "ioc_tlp_id": 2,
  "ioc_description": "Source IP from brute force alert. AbuseIPDB: {{ $json.abuseipdb.data.abuseConfidenceScore }}%",
  "cid": "{{ $('Create IRIS Case').item.json.data.case_id }}"
}
```

IOC type ID 76 is `ip-src` in the default IRIS schema. Verify this in your instance at **Advanced > IOC Types**.

#### Node 10: Add Asset to IRIS Case

* Type: HTTP Request
* Method: POST
* URL: `https://10.10.10.40/api/v1/case/asset/add`
* Credential: `DFIR IRIS`
* Body (JSON):
```json
{
  "asset_name": "{{ $json.agent_name }}",
  "asset_type_id": 9,
  "asset_ip": "{{ $json.agent_ip }}",
  "asset_description": "Wazuh-monitored endpoint that received the brute force attack",
  "cid": "{{ $('Create IRIS Case').item.json.data.case_id }}"
}
```

#### Node 11: Write IOC to MISP (if not already present)

* Type: IF → HTTP Request
* Condition: `{{ $('Check MISP').item.json.response.Attribute.length }}` equals `0` (IOC not already in MISP)
* If true, POST to `https://10.10.10.30/events/add`:

```json
{
  "Event": {
    "info": "Brute Force Source IP - {{ $json.srcip }}",
    "distribution": 0,
    "threat_level_id": 2,
    "analysis": 1,
    "Attribute": [
      {
        "type": "ip-src",
        "value": "{{ $json.srcip }}",
        "comment": "Confirmed brute force source. AbuseIPDB: {{ $json.abuseipdb.data.abuseConfidenceScore }}%"
      }
    ]
  }
}
```

#### Node 12: Block IP via OPNsense

* Type: HTTP Request
* Method: POST
* URL: `https://10.10.1.254/api/firewall/alias/addHost/blocklist_dynamic`
* Credential: `OPNsense API`
* Body (JSON):
```json
{
  "address": "{{ $json.srcip }}"
}
```

Follow this with a second HTTP Request to apply the change:
* URL: `https://10.10.1.254/api/firewall/alias/reconfigure`
* Method: POST

> This requires a firewall alias named `blocklist_dynamic` to exist on OPNsense, with a block rule applied to it. Create the alias in OPNsense under **Firewall > Aliases** before testing. Type: Host(s), leave it empty. Then create a block rule referencing this alias on the WAN and VLAN 30 interfaces.

#### Node 13: Error Handling

Add an **Error Trigger** node to the workflow. Route it to an HTTP Request that logs the error to a Slack webhook or sends an email. Silent workflow failures are difficult to diagnose without this.

---

## 4. Activate the Workflow

Toggle the workflow to **Active** using the switch in the top-right corner of the n8n editor. The Webhook node will now receive alerts from Wazuh in real time.

---

## 5. End-to-End Test — Simulated Brute Force

Simulate an RDP or SSH brute force from the Victim VLAN endpoint to trigger Wazuh rule 60106. The simplest method is a repeated failed SSH login:

```bash
# Run on the Victim VLAN endpoint (10.10.20.x)
# Replace TARGET with a host in VLAN 30 (or the Wazuh VM)
for i in {1..10}; do ssh invalid_user@TARGET; done
```

Alternatively, use a tool like `hydra` against the SSH service on a test target if you want to trigger the alert with a real external-looking source IP.

### Expected Timeline

| Time | Event |
| ---- | ----- |
| T+0s | Failed logins generated on endpoint |
| T+2s | Wazuh agent ships logs to Wazuh Manager |
| T+3s | Wazuh rule 60106 fires, alert POSTed to n8n webhook |
| T+4s | n8n workflow starts — AbuseIPDB and VirusTotal queries run |
| T+6s | Enrichment results merged, decision node evaluates |
| T+7s | IRIS case created |
| T+8s | IOC and asset added to IRIS case |
| T+9s | MISP IOC event created (if IP not already known) |
| T+10s | OPNsense blocklist updated, alias reconfigured |

### Verify Each Stage

**Wazuh:**
* Dashboard > Security Events — rule 60106 alert visible

**n8n:**
* Executions list — workflow shows `Succeeded` with green status
* Click the execution to inspect each node's input/output

**DFIR IRIS:**
* Cases list — a new BRF case visible with the source IP in the title
* Open the case — IOC and asset tabs populated
* Timeline tab — enrichment note present

**MISP:**
* Events list — new event with the source IP as an attribute (if IP was not already known)

**OPNsense:**
* Firewall > Aliases > `blocklist_dynamic` — source IP listed
* Firewall > Logs — subsequent traffic from the IP shows as blocked

---

## 6. Importing Additional Workflows

The brute force workflow above is the base pattern. The other three use cases documented in this project follow the same structure with different rule IDs and enrichment logic:

| Use Case | Wazuh Rule | Primary Enrichment | Additional Action |
| -------- | ---------- | ------------------ | ----------------- |
| Brute Force | 60106 | AbuseIPDB, VirusTotal | OPNsense block |
| Malicious File | 87105 | VirusTotal (file hash) | Agent isolation (Wazuh active response) |
| Suspicious Login | 100002 | AbuseIPDB, Entra sign-in logs | Entra account disable |
| C2 Beaconing | Suricata ET rules | VirusTotal (domain/IP) | OPNsense block + DNS sinkhole |

Build each as a separate workflow in n8n, triggered by the same Webhook node but routed by the Filter node on `rule.id`.

---

## 7. Ongoing Tuning

A SOC that has never been tuned generates more noise than signal. After running for 48–72 hours:

* Review the n8n executions list — identify workflows that fired and then were discarded by the decision node. If legitimate traffic is triggering rule 60106, adjust the AbuseIPDB confidence threshold or add a whitelist check.
* Review IRIS cases — identify any cases opened for clearly benign activity and mark them as False Positive. Feed that context back into the workflow decision logic.
* Review MISP — remove any IOCs that were added incorrectly. An inaccurate IOC database degrades enrichment for every future alert.
* Review OPNsense blocklist — confirm no legitimate IPs were blocked. Remove any accidental entries.

---

## 8. Verification Summary

| Component | Verified By |
| --------- | ----------- |
| Wazuh receiving alerts | Alert visible in Security Events |
| n8n receiving Wazuh webhook | Execution log shows incoming payload |
| AbuseIPDB enrichment working | Execution node shows confidence score |
| VirusTotal enrichment working | Execution node shows detection count |
| IRIS case created | Case visible with correct title and description |
| IRIS IOC populated | IOC tab shows source IP |
| IRIS asset populated | Asset tab shows affected endpoint |
| MISP IOC written | MISP event visible for source IP |
| OPNsense block active | Alias contains source IP, subsequent traffic blocked |

---

## 9. The SOC is Operational

All five components are deployed, integrated, and validated. The pipeline runs from detection to automated response in under 15 seconds with no human input required for blocking and case creation. Human analysts use DFIR IRIS to manage the ongoing investigation, update case status, and close incidents.

The next step is deploying Wazuh agents across additional endpoints and monitoring alert volume for the first 72 hours before tuning thresholds.

---

## 10. Final Checklist

- [ ] All five VMs running and reachable
- [ ] All six n8n credentials pass the Test check
- [ ] `blocklist_dynamic` alias created in OPNsense with a block rule
- [ ] Brute force workflow built and set to Active in n8n
- [ ] Error Trigger node added to the workflow
- [ ] End-to-end test completed — simulated brute force processed
- [ ] Wazuh alert visible for the test event
- [ ] n8n execution shows Succeeded for all nodes
- [ ] IRIS case created with IOC and asset populated
- [ ] MISP event created for the test source IP
- [ ] OPNsense blocklist contains the test source IP
- [ ] Test IP removed from OPNsense blocklist after verification
- [ ] Test MISP event removed or tagged as test after verification
- [ ] Test IRIS case closed as Test/Exercise
- [ ] Snapshots taken of all VMs in the operational state

---
