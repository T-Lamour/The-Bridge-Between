# Claude AI Triage — Intelligent Alert Analysis

## Overview

This guide adds Claude (Anthropic's AI) as an analysis layer inside your existing n8n workflows. Claude sits between enrichment and response, reading the full alert and enrichment context and returning a structured verdict: severity rating, plain-English client notification, analyst notes, and recommended actions.

This replaces the hardcoded `IF (abuseConfidenceScore > 50)` threshold logic with reasoning that accounts for business context, combinations of weak signals, and false positive patterns — things a fixed number cannot do.

By the end of this guide:

* Claude is configured as an n8n credential
* The brute force workflow (from `06-integration.md`) has a Claude triage node inserted after enrichment
* IRIS cases are created with Claude-drafted descriptions pre-populated
* A plain-English notification is ready to send to the SMB client
* A client context lookup pattern is in place for multi-tenant use

---

## Prerequisites

* `06-integration.md` completed — the brute force workflow is active and passing end-to-end tests
* An Anthropic API key from [console.anthropic.com](https://console.anthropic.com)
* The n8n instance is running at `http://10.10.10.20:5678`

---

## 1. Add the Anthropic Credential in n8n

n8n does not have a native Anthropic credential type. Use **Generic Credential** with a custom header.

1. In n8n, go to **Settings > Credentials > Add Credential**
2. Select **Generic Credential Type → Header Auth**
3. Configure as follows:

| Field | Value |
|-------|-------|
| Name | `Anthropic API` |
| Header Name | `x-api-key` |
| Header Value | `<your Anthropic API key>` |

4. Save. The credential does not have a built-in test — you will validate it in Step 4.

---

## 2. Add a Client Context Node

Before calling Claude, the workflow needs to know which client this alert belongs to and what their business context is. This is what makes the output useful to a non-technical SMB owner.

In the brute force workflow, insert a **Code** node directly after the **Set — Extract Fields** node (Node 3 in `06-integration.md`). Name it `Set Client Context`.

**Code node — JavaScript:**

```javascript
// Client registry — add an entry for each SMB you monitor.
// In production, replace this with an HTTP lookup to your CRM or a Notion/Airtable database.
const clients = {
  "default": {
    client_name: "Unknown Client",
    client_type: "small business",
    client_size: "unknown",
    client_assets: "general endpoints and servers",
    client_contact_email: ""
  },
  "acme-accounting": {
    client_name: "Acme Accounting Ltd",
    client_type: "accounting firm",
    client_size: "12",
    client_assets: "finance server (10.10.20.10), QuickBooks host, client data NAS",
    client_contact_email: "it@acme-accounting.example.com"
  },
  "bright-dental": {
    client_name: "Bright Dental Practice",
    client_type: "dental practice",
    client_size: "8",
    client_assets: "patient records server, appointment system, reception PCs",
    client_contact_email: "admin@bright-dental.example.com"
  }
};

// Derive client_id from the Wazuh agent group name or a custom tag on the agent.
// Set the agent group to match the client key during Wazuh agent enrollment.
const agentName = $input.item.json.agent_name || "";
const clientId = agentName.split("-")[0] || "default";

const ctx = clients[clientId] || clients["default"];

return {
  json: {
    ...$input.item.json,
    ...ctx,
    client_id: clientId
  }
};
```

> **Multi-tenant onboarding:** When enrolling a new SMB's Wazuh agents, name them `<client-id>-<hostname>` (e.g. `acme-accounting-dc01`). The Code node above extracts the prefix automatically. Add a matching entry to the `clients` object.

---

## 3. Insert the Claude Triage Node

After the **Merge Enrichment Results** node (Node 5) and before the **Decision — Is it Malicious?** IF node, insert an **HTTP Request** node. Name it `Claude — Alert Triage`.

### Node Configuration

| Field | Value |
|-------|-------|
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |
| Authentication | Generic Credential Type → `Anthropic API` |
| Send Headers | Enabled |

**Additional Headers:**

| Name | Value |
|------|-------|
| `anthropic-version` | `2023-06-01` |
| `content-type` | `application/json` |

**Body — JSON (Expression mode):**

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a SOC analyst at an MSSP serving small businesses. You receive security alerts with enrichment data and must assess them clearly and quickly.\n\nAlways respond with valid JSON only — no markdown, no explanation outside the JSON. Use this exact structure:\n{\n  \"severity\": \"CRITICAL|HIGH|MEDIUM|LOW|FP\",\n  \"confidence\": <0-100>,\n  \"triage_summary\": \"<2-3 sentences: what happened, what the enrichment shows, why you rated it this way>\",\n  \"client_notification\": \"<1-2 plain English sentences for a non-technical business owner. No jargon. Start with what happened, end with what was done about it or what they need to know.>\",\n  \"analyst_notes\": \"<technical context: what to investigate next, log sources to check, lateral movement risk, persistence indicators to look for>\",\n  \"recommended_actions\": [\"<action 1>\", \"<action 2>\"],\n  \"false_positive_reasoning\": \"<only include this key if severity is FP>\"\n}\n\nSeverity guide:\n- CRITICAL: Active compromise, data exfiltration likely, immediate action required\n- HIGH: Strong indicators of malicious activity, automated response warranted\n- MEDIUM: Suspicious but ambiguous — human review recommended before blocking\n- LOW: Likely benign but worth logging\n- FP: False positive — explain why",
  "messages": [
    {
      "role": "user",
      "content": "Client: {{ $json.client_name }} ({{ $json.client_type }}, ~{{ $json.client_size }} employees)\nCritical assets: {{ $json.client_assets }}\n\nAlert:\n- Rule: {{ $json.rule_description }} (ID: {{ $json.rule_id }}, Level: {{ $json.rule_level }}/15)\n- Source IP: {{ $json.srcip }}\n- Affected endpoint: {{ $json.agent_name }} ({{ $json.agent_ip }})\n- Timestamp: {{ $json.timestamp }}\n\nEnrichment:\n- AbuseIPDB confidence score: {{ $json.abuseipdb.data.abuseConfidenceScore }}%\n- AbuseIPDB country: {{ $json.abuseipdb.data.countryCode }}\n- AbuseIPDB ISP: {{ $json.abuseipdb.data.isp }}\n- AbuseIPDB total prior reports: {{ $json.abuseipdb.data.totalReports }}\n- VirusTotal malicious detections: {{ $json.virustotal.data.attributes.last_analysis_stats.malicious }} / {{ $json.virustotal.data.attributes.last_analysis_stats.total }} engines\n- MISP known IOC: {{ $json.misp_known ?? 'not checked yet' }}\n\nAssess this alert."
    }
  ]
}
```

### Extract Claude's Response

After the HTTP Request node, add a **Code** node named `Parse Claude Response`. This extracts Claude's JSON from the API response and merges it with the existing alert data.

```javascript
const response = $input.item.json;

// Claude returns content as an array of content blocks
const rawText = response.content[0].text;

let claudeOutput;
try {
  claudeOutput = JSON.parse(rawText);
} catch (e) {
  // If Claude didn't return clean JSON (shouldn't happen), default to MEDIUM
  // so the alert is not silently dropped
  claudeOutput = {
    severity: "MEDIUM",
    confidence: 50,
    triage_summary: "Claude response could not be parsed — manual review required.",
    client_notification: "Our security system flagged unusual activity. Our team is reviewing it now.",
    analyst_notes: "Claude API returned unparseable response. Raw: " + rawText.slice(0, 500),
    recommended_actions: ["Manual analyst review"]
  };
}

return {
  json: {
    ...$input.item.json,
    claude: claudeOutput,
    claude_severity: claudeOutput.severity,
    claude_confidence: claudeOutput.confidence
  }
};
```

---

## 4. Update the Decision Node

Replace the original hardcoded IF node (**Decision — Is it Malicious?**, Node 6) with a new IF node that uses Claude's verdict.

**Condition A:** `{{ $json.claude_severity }}` equals `HIGH`
**OR Condition B:** `{{ $json.claude_severity }}` equals `CRITICAL`
**OR Condition C:** `{{ $json.claude_severity }}` equals `MEDIUM` AND `{{ $json.claude_confidence }}` greater than `70`

Alerts rated `LOW` or `FP` fall through to a No-Op node. Optionally log them to a **weekly digest** workflow using n8n's Schedule trigger for client reporting.

---

## 5. Update the IRIS Case Creation Node

Replace the hardcoded `case_description` in **Node 8: Create IRIS Case** with Claude's pre-drafted output:

```json
{
  "case_name": "{{ $json.claude_severity }}-{{ $now.toFormat('yyyyMMdd-HHmm') }} - {{ $json.rule_description }} from {{ $json.srcip }}",
  "case_description": "## Triage Summary\n{{ $json.claude.triage_summary }}\n\n## Analyst Notes\n{{ $json.claude.analyst_notes }}\n\n## Recommended Actions\n{{ $json.claude.recommended_actions.join('\\n- ') }}\n\n---\n## Raw Alert Context\nWazuh rule {{ $json.rule_id }} | Level {{ $json.rule_level }} | Agent: {{ $json.agent_name }}\n\nAbuseIPDB: {{ $json.abuseipdb.data.abuseConfidenceScore }}% | VirusTotal: {{ $json.virustotal.data.attributes.last_analysis_stats.malicious }} detections\nCountry: {{ $json.abuseipdb.data.countryCode }} | ISP: {{ $json.abuseipdb.data.isp }}",
  "case_customer": 1,
  "case_template_id": 1,
  "case_soc_id": "WZ-{{ $json.rule_id }}-{{ $now.toUnixInteger() }}"
}
```

Analysts opening the case now see a structured investigation starting point, not a wall of raw JSON.

---

## 6. Add a Client Notification Node (Optional)

For SMB clients who want to be kept informed, add an **Email** or **Slack** node after IRIS case creation. Only fire it for HIGH and CRITICAL alerts.

**Email node body:**

```
Subject: Security Alert — {{ $json.client_name }} — {{ $now.toFormat('dd MMM yyyy HH:mm') }}

{{ $json.claude.client_notification }}

Our team has opened a case to investigate. Reference: {{ $('Create IRIS Case').item.json.data.case_id }}

If you have any questions, reply to this email.

— Your Security Team
```

This gives SMB owners immediate, plain-English awareness without overwhelming them with technical detail.

---

## 7. Validate the Integration

Trigger the brute force simulation from `06-integration.md`:

```bash
for i in {1..10}; do ssh invalid_user@TARGET; done
```

Check each stage:

| Stage | What to verify |
|-------|---------------|
| **Claude — Alert Triage node** | Execution shows `200 OK`; response body contains `severity` and `triage_summary` keys |
| **Parse Claude Response node** | `claude_severity` and `claude_confidence` fields present in output |
| **Decision node** | Routes correctly based on Claude's severity (not hardcoded thresholds) |
| **IRIS case** | Case description contains Claude's triage summary and analyst notes |
| **Client notification** | Email/Slack sent with plain-English summary (HIGH/CRITICAL only) |

If the Claude node returns a `401`, re-check the `x-api-key` header value in the credential. If it returns `529` (overloaded), add a **Wait** node (2 seconds) before the Claude node and retry once.

---

## 8. Extend to Other Workflows

Apply the same pattern to the other three workflows:

| Workflow | Prompt addition |
|----------|----------------|
| **Suspicious Login** | Include `login_successful: true/false` and Entra sign-in risk score in the user message |
| **Malicious File** | Include VirusTotal file hash scan results and file path in the user message |
| **C2 Beaconing** | Include Suricata rule name, destination domain, and beacon interval in the user message |

The system prompt stays the same across all workflows — only the user message content changes to reflect what was detected.

---

## 9. Cost Estimate

Claude Sonnet 4.6 pricing (as of May 2026):

| Assumption | Value |
|------------|-------|
| Alerts processed per day (per SMB) | ~20–50 |
| Tokens per call (in + out) | ~800 input / ~400 output |
| Cost per 1M input tokens | $3.00 |
| Cost per 1M output tokens | $15.00 |
| **Daily cost per SMB** | **~$0.05–$0.12** |
| **Monthly cost for 10 SMBs** | **~$15–$36** |

This is a negligible cost relative to the analyst time saved and the improvement in triage quality.

---

## 10. Checklist

- [ ] Anthropic API key added as Header Auth credential in n8n
- [ ] `Set Client Context` Code node inserted after `Set — Extract Fields`
- [ ] `Claude — Alert Triage` HTTP Request node inserted after `Merge Enrichment Results`
- [ ] `Parse Claude Response` Code node parses and merges Claude output
- [ ] Decision IF node updated to use `claude_severity`
- [ ] IRIS case description updated to use `claude.triage_summary` and `claude.analyst_notes`
- [ ] Client notification node added (optional)
- [ ] End-to-end test completed — Claude triage node shows `200 OK`
- [ ] IRIS case opens with structured Claude-drafted description
- [ ] Cost monitoring enabled in Anthropic console (set a monthly spend limit)

---
