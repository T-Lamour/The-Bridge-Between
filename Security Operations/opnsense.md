# OPNsense + Suricata — Network Security Layer

## Overview

OPNsense is the **network perimeter firewall and traffic control system** in this SOC environment. It handles traffic filtering between VLANs, enforces firewall rules, and acts as the automated enforcement point when n8n triggers a response action.

Suricata runs as a plugin on OPNsense in IDS mode, providing real-time deep packet inspection using the Emerging Threats Open ruleset. Alerts are forwarded to Wazuh via syslog for correlation with endpoint events.

---

## Network Architecture

OPNsense sits at the gateway of all three VLANs:

```
External Network / Internet
          │
          ▼
    OPNsense Firewall (10.10.1.254)
  ┌─────────────────────────────────────────┐
  │  VLAN 10 — Management (10.10.1.0/24)   │
  │  VLAN 20 — SOC Tools  (10.10.10.0/24)  │
  │  VLAN 30 — Victim Lab (10.10.20.0/24)  │
  └─────────────────────────────────────────┘
```

All inter-VLAN traffic passes through OPNsense, making it the single choke point for both detection and blocking.

---

## Key Responsibilities

* Traffic filtering between VLANs and to the internet
* Suricata IDS — real-time packet inspection on all interfaces
* Syslog forwarding of Suricata alerts to Wazuh (UDP 514)
* API-driven automated IP blocking via the `blocklist_dynamic` alias
* NAT and DHCP for VLAN 10 and VLAN 30

---

## Suricata IDS Integration

Suricata is deployed as an OPNsense plugin in **IDS mode** (monitoring, not inline blocking). This avoids disruption to legitimate traffic while tuning is ongoing.

### Ruleset

* **Emerging Threats Open** — updated daily via OPNsense's built-in feed management
* Covers: C2 beaconing, port scanning, known exploit traffic, malicious DNS, protocol anomalies

### Alert Flow

```
Suricata detects suspicious traffic
          │
          ▼
OPNsense logs the event
          │
          ▼
Syslog forwarded to Wazuh (10.10.10.10:514)
          │
          ▼
Wazuh correlates with endpoint logs → alert generated
          │
          ▼
n8n webhook triggered → enrichment + response
```

---

## Automated Response — IP Blocking

n8n calls the OPNsense API as the final step of any confirmed-malicious workflow. The API adds the source IP to the `blocklist_dynamic` alias, which has a block rule applied on all interfaces.

### Block call

```
POST https://10.10.1.254/api/firewall/alias/addHost/blocklist_dynamic
{
  "address": "<srcip>"
}
```

### Apply the change

```
POST https://10.10.1.254/api/firewall/alias/reconfigure
```

The alias `blocklist_dynamic` is created in OPNsense under **Firewall > Aliases** as a Host(s) type with an associated block rule on the WAN and VLAN 30 interfaces. See [01-opnsense.md](../Deployment/01-opnsense.md) for setup steps.

---

## Integration with SOC Pipeline

| From | To | Method | Purpose |
|------|----|--------|---------|
| OPNsense / Suricata | Wazuh | Syslog UDP 514 | Network alert ingestion |
| n8n | OPNsense API | HTTPS REST | Automated IP blocking |
| Wazuh | n8n | Webhook | Alert forwarding for enrichment |

---

## Firewall Rule Design

| Rule | Source | Destination | Action |
|------|--------|-------------|--------|
| VLAN 30 → SOC tools | 10.10.20.0/24 | 10.10.10.0/24 | Block (default) |
| VLAN 30 → Wazuh agent port | 10.10.20.0/24 | 10.10.10.10:1514 | Allow |
| SOC → Management | 10.10.10.0/24 | 10.10.1.0/24 | Allow |
| blocklist_dynamic | any | any | Block (applied last) |

---

## Why OPNsense

OPNsense is a fork of pfSense with a weekly release cadence, a cleaner REST API, and an active development community. The alias management endpoints used in this project (`/api/firewall/alias/addHost`, `/api/firewall/alias/reconfigure`) are stable and well-documented. The built-in Suricata plugin handles IDS configuration entirely through the UI without requiring SSH access to configure rulesets or interfaces.

---

## Limitations

* Suricata in IDS mode logs but does not drop traffic — requires tuning before switching to IPS mode
* Encrypted traffic limits deep packet inspection beyond TLS metadata
* The `blocklist_dynamic` alias must be pre-created and a block rule pre-applied before n8n automation will work
* The OPNsense API uses a self-signed certificate — HTTP Request nodes in n8n require **Ignore SSL Issues** enabled

---

## Verification Checks

| Check | How to Verify |
|-------|---------------|
| OPNsense web UI accessible | Navigate to `https://10.10.1.254` |
| Suricata running | Services > Intrusion Detection — Status: running |
| Syslog forwarding active | Wazuh dashboard shows OPNsense source events |
| blocklist_dynamic alias exists | Firewall > Aliases — alias listed |
| n8n API block works | Run end-to-end test from `06-integration.md`, confirm IP appears in alias |

---
