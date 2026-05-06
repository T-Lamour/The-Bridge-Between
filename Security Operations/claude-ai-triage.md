# Claude AI Triage

## Overview

Claude is an AI model (by Anthropic) integrated into the n8n SOAR pipeline as an analysis layer between enrichment and response. It reads the full alert and enrichment context, applies reasoning about the specific client's business, and returns a structured verdict.

This replaces hardcoded threshold logic (e.g. "block if AbuseIPDB > 50") with contextual judgement that accounts for:

* Combinations of weak signals that individually fall below threshold
* Business context — a brute force against an accounting firm's finance server is not the same as one against a test VM
* False positive patterns — cloud scanner IPs, pentest tools, known benign services
* Multi-tenancy — different clients get appropriate, relevant output

---

## Architecture Role

Claude sits inside the n8n workflow, called via the Anthropic Messages API:

```
Wazuh alert
    ↓
n8n — Extract fields + Set client context
    ↓
AbuseIPDB + VirusTotal (parallel enrichment)
    ↓
Merge enrichment results
    ↓
Claude — Alert Triage  ◄── NEW
    ↓
Parse Claude response
    ↓
IF claude_severity == HIGH or CRITICAL
    ├── MISP IOC check
    ├── Create IRIS case (with Claude-drafted description)
    ├── Add IOC + asset to IRIS
    ├── Write IOC to MISP
    ├── Block IP via OPNsense
    └── Send plain-English notification to client
```

---

## What Claude Receives

Each call to Claude includes:

* **Client context** — business name, type (accounting firm, dental practice, etc.), headcount, critical asset list
* **Alert fields** — Wazuh rule ID, description, severity level, source IP, affected endpoint, timestamp
* **Enrichment results** — AbuseIPDB confidence score, country, ISP, prior report count; VirusTotal detection count out of total engines

---

## What Claude Returns

Claude responds with a structured JSON object every time:

| Field | Type | Description |
|-------|------|-------------|
| `severity` | `CRITICAL \| HIGH \| MEDIUM \| LOW \| FP` | Overall verdict |
| `confidence` | `0–100` | Claude's confidence in the verdict |
| `triage_summary` | string | 2–3 sentences for the analyst: what happened, what enrichment shows, why this severity |
| `client_notification` | string | 1–2 plain-English sentences for the business owner — no jargon |
| `analyst_notes` | string | Technical investigation guidance: next log sources to check, lateral movement risk, persistence indicators |
| `recommended_actions` | string[] | Ordered list of concrete actions |
| `false_positive_reasoning` | string | Only present when `severity == FP`; explains why |

---

## Severity Definitions

| Severity | Meaning | Automated response triggered? |
|----------|---------|-------------------------------|
| **CRITICAL** | Active compromise, data exfiltration likely | Yes — immediate block + account disable |
| **HIGH** | Strong indicators of malicious activity | Yes — block + IRIS case + client notification |
| **MEDIUM** | Suspicious but ambiguous | IRIS case created, no automated block; analyst review |
| **LOW** | Likely benign, worth logging | Logged only; included in weekly digest |
| **FP** | False positive | Discarded; Claude explains reasoning in `false_positive_reasoning` |

---

## Multi-Tenant Client Context

Each SMB client is registered in the `Set Client Context` Code node inside n8n. The registry maps a `client_id` (derived from the Wazuh agent name prefix) to business context:

```
Agent name: acme-accounting-dc01
                ↑
           client_id = "acme-accounting"
                ↓
Looked up in client registry → client_name, client_type, client_assets, contact email
```

To add a new SMB client:

1. Enroll their Wazuh agents with names prefixed by their `client_id` (e.g. `bright-dental-reception01`)
2. Add an entry to the `clients` object in the `Set Client Context` Code node
3. Set `client_assets` to describe what matters most to that business — this is what Claude uses to contextualise impact

---

## IRIS Case Output

With Claude integrated, every IRIS case opens with a pre-drafted structure:

```
## Triage Summary
<Claude's 2-3 sentence assessment>

## Analyst Notes
<Technical investigation guidance>

## Recommended Actions
- <Action 1>
- <Action 2>

---
## Raw Alert Context
Wazuh rule 60106 | Level 12 | Agent: acme-accounting-dc01
AbuseIPDB: 87% | VirusTotal: 9 detections
Country: RU | ISP: Hosting company LLC
```

Analysts spend their time investigating, not reformatting raw JSON into readable notes.

---

## Client Notifications

For HIGH and CRITICAL alerts, an automated email goes to the SMB's contact address. Example output:

> *"Someone attempted to break into your server (acme-accounting-dc01) from a Russian IP address that has been reported for malicious activity 312 times. We've automatically blocked the source and opened an investigation — no action is needed from you."*

This keeps non-technical clients appropriately informed without creating panic or noise.

---

## Operational Notes

### API Rate Limits

Claude Sonnet 4.6 allows 2,000 requests per minute at Tier 1. For a small MSSP with 5–15 SMB clients generating 20–50 alerts each per day, this limit will never be reached.

If you add a high-volume client (enterprise-grade log volume), add a **Wait** node (1 second) before the Claude node to avoid burst spikes during incident surges.

### Failure Handling

The `Parse Claude Response` Code node includes a fallback: if Claude's response cannot be parsed as JSON, the alert is classified as `MEDIUM` with a note to the analyst. This prevents alerts from being silently dropped on API errors.

The n8n Error Trigger node (Node 13 in `06-integration.md`) also catches HTTP-level failures (network timeout, 5xx responses from the API).

### Prompt Stability

The system prompt is fixed and lives inside the `Claude — Alert Triage` HTTP Request node body. If you update it:

1. Test with a simulated brute force before re-activating the workflow
2. Verify the response is still valid JSON and contains all required keys
3. Check that the `Parse Claude Response` node handles the new output correctly

### Cost Control

Set a monthly spend limit in the [Anthropic Console](https://console.anthropic.com) under **Billing > Spend Limit**. At typical MSSP volumes, $50/month covers 10–15 SMB clients with headroom.

Monitor token usage per workflow execution in the n8n execution log — each Claude node call logs the full response including token counts.

---

## Limitations

* Claude does not have memory between alerts — each call is independent. It cannot spot slow-burn patterns across multiple low-severity alerts over time. This is a SIEM/correlation function, not an AI function.
* Client context is static (hardcoded in n8n). For a growing MSSP, replace the Code node lookup with an API call to a CRM or database so client details stay current without editing n8n workflows.
* Claude's false positive detection is probabilistic. It will occasionally misclassify — treat `FP` verdicts as a signal to review, not a guarantee. Tune by reviewing discarded alerts weekly.

---

## Summary

Claude adds a reasoning layer that transforms raw enrichment data into structured, actionable, client-aware output. Analysts open IRIS cases with context already written. SMB owners receive plain-English notifications. The automated response pipeline uses a nuanced verdict instead of a single threshold.

The integration is a single HTTP Request node in n8n — no new infrastructure, no new VMs, no new agents. It runs on the existing n8n VM alongside the rest of the pipeline.
