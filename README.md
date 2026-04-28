<h1 align="center">The Bridge Between</h1>

<p align="center"><strong>An open-source SOC for businesses that can't afford a £15k/month MSSP.</strong></p>

<p align="center">
  <a href="#quick-start">Quick Start</a> |
  <a href="#the-stack">The Stack</a> |
  <a href="#use-cases">Use Cases</a> |
  <a href="Deployment/">Deployment</a> |
  <a href="Compliance/nist-csf.md">NIST CSF</a> |
  <a href="Compliance/ISO27001-control-mapping.md">ISO 27001</a>
</p>

---

## The Message

Enterprise SOCs run on six-figure tooling. SMBs run on antivirus and hope for the best. This project is the bridge — a fully functional, automated SOC built entirely on free, open-source tools. Logs in, enriched incidents out, IPs blocked, accounts disabled, no licensing fees.

> It takes you from **"something might be wrong"**
> to **"here's what's wrong, why it matters, what to do next."**

---

## Architecture

<table>
<tr>
<td><img src="Architecture/SIEM-Solution.png" width="500" alt="SOC Architecture"></td>
<td>
<table>
<thead><tr><th>Tool</th><th>Role</th><th>Why</th></tr></thead>
<tbody>
<tr><td>OPNsense + Suricata</td><td>Firewall + IDS</td><td>Purpose-built firewall distro; Suricata adds deep packet inspection and ET Open signatures</td></tr>
<tr><td>Wazuh</td><td>SIEM / XDR</td><td>Single agent covers log collection, FIM, vulnerability scanning, and rule-based detection</td></tr>
<tr><td>n8n</td><td>SOAR automation</td><td>Self-hosted, code-optional workflow engine; integrates with every tool via REST</td></tr>
<tr><td>MISP</td><td>Threat intelligence</td><td>Internal IOC database with feed ingestion; every confirmed indicator feeds future detections</td></tr>
<tr><td>DFIR IRIS</td><td>Case management</td><td>Structured incident tracking from triage to closure, with timeline, assets, and IOC links</td></tr>
</tbody>
</table>
</td>
</tr>
</table>

Each tool runs as a Docker Compose stack on its own VM. Reasoning: [Deployment/00-prerequisites.md](Deployment/00-prerequisites.md).

---

## Cost Comparison

| Solution | Cost | What You Get |
| -------- | ---- | ------------ |
| MSSP "SIEM" subscription | £120k–£240k/year | Repackaged tooling + support |
| Enterprise security stack | £20k–£100k+/year | Full ecosystem, if you can afford it |
| **The Bridge Between** | **£0 licensing + ~£500 hardware (one-off)** | **Fully functional SOC stack** |

The hardware is a one-off purchase — a second-hand mini-PC (Lenovo M920q, HP EliteDesk, or similar) with 32–64 GB RAM. Everything else is free.

---

## Quick Start

1. [00 — Prerequisites](Deployment/00-prerequisites.md) — hardware, hypervisor, network plan, VM provisioning
2. [01 — OPNsense](Deployment/01-opnsense.md) — firewall, VLANs, Suricata IDS, syslog forwarding
3. [02 — Wazuh](Deployment/02-wazuh.md) — SIEM stack, agent enrolment, syslog receiver
4. [03 — n8n](Deployment/03-n8n.md) — SOAR engine, credentials, Wazuh webhook integration
5. [04 — MISP](Deployment/04-misp.md) — threat intel platform, feed configuration, API key
6. [05 — DFIR IRIS](Deployment/05-iris.md) — case management, customer setup, API key
7. [06 — Integration](Deployment/06-integration.md) — end-to-end pipeline wiring and live test
8. 07 — Dashboard *(coming soon)* — unified SOC overview across all tools
9. 08 — Hardening *(coming soon)* — TLS, credential rotation, ufw tuning
10. 09 — Patching *(coming soon)* — automated update and snapshot strategy

Estimated deployment time: a weekend, assuming prerequisites are in place.

---

## What This Project Demonstrates

* **End-to-end SOC pipeline** — detection to enriched incident in under 15 seconds
* **Real automation** — n8n actively blocks IPs via OPNsense, disables accounts via Entra, and revokes sessions; not just "sends an email"
* **Threat intel integration** — VirusTotal, AbuseIPDB, and MISP all feed enrichment decisions before any case is opened
* **Compliance alignment** — full mapping to NIST CSF (all five functions) and ISO 27001 controls
* **Realistic use cases** — brute force, account compromise, malicious file download, and C2 beaconing; all fully documented with payloads and timelines

---

## Use Cases

| Scenario | Source | MITRE Technique | Automated MTTR |
| -------- | ------ | --------------- | -------------- |
| [RDP Brute Force](Use-Cases/brute-force-attack.md) | Wazuh rule 60106 | T1110 | ~9 sec |
| [Suspicious Login](Use-Cases/suspicious-login.md) | M365 / Entra | T1078, T1110.004 | ~11 sec |
| [Malicious File Download](Use-Cases/malicious-file-download.md) | Wazuh FIM + VirusTotal | T1566.002, T1204.002 | ~14 sec |
| [C2 Beaconing](Use-Cases/suspicious-network-traffic.md) | Suricata ET rules | T1071.001, T1571 | ~8 sec |

---

## Roadmap

| Area | Improvement | Status |
| ---- | ----------- | ------ |
| Automation | Patching and snapshot automation | In progress |
| Monitoring | Grafana + Prometheus for infrastructure metrics | Planned |
| Compliance | CIS Benchmark checks via Wazuh SCA | Planned |
| Metrics | MTTD / MTTR tracking dashboards | Planned |
| Response | Automated endpoint isolation via Wazuh active response | Planned |
| Threat Intel | Automated IOC ingestion from external MISP feeds | Planned |
| Correlation | Improved multi-source alert correlation rules | Planned |

---

## About

Built and maintained by **Tidjani Lamour**, Cyber Security Professional.

[LinkedIn](#) | [Website](#) | [CV](#)

If this project is useful to you, star the repo.
