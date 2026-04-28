# Prerequisites — The Bridge Between SOC Lab

## Overview

This document covers everything that must be in place before deploying any individual tool. Work through it top to bottom. Each subsequent deployment guide assumes this baseline is complete.

---

## 1. Deployment Model

Five virtual machines make up the stack. OPNsense runs as a bare OS install because it is a purpose-built firewall distribution. Everything else runs Docker Compose on Ubuntu 22.04 to keep upgrades, rollbacks, and port management consistent.

| VM | Tool | Deployment Method | Reason |
| -- | ---- | ----------------- | ------ |
| opnsense | OPNsense 25.x | Bare OS install | Firewall distros need direct vNIC access — they do not run inside containers |
| wazuh | Wazuh SIEM / XDR | Docker Compose on Ubuntu 22.04 | The official stack bundles Indexer, Manager, and Dashboard in a single Compose file |
| n8n | n8n SOAR | Docker Compose on Ubuntu 22.04 | The official n8n image is the simplest production-ready deployment path |
| misp | MISP | Docker Compose on Ubuntu 22.04 | MISP-Docker is the maintained route; manual installs are a dependency nightmare |
| iris | DFIR IRIS | Docker Compose on Ubuntu 22.04 | IRIS ships as a first-class Compose stack with no supported bare-metal path |

---

## 2. Host Hardware Requirements

The hypervisor host runs all five VMs simultaneously. The figures below account for host OS overhead on top of the VM allocations in section 4.

| | Minimum | Recommended |
| -- | ------- | ----------- |
| CPU | 6 cores | 8 cores |
| RAM | 32 GB | 64 GB |
| Storage | 500 GB SSD | 1 TB NVMe |
| Network | 1 GbE NIC | 2.5 GbE or dual NIC |

A second-hand mini-PC in the **£300–£500** range is sufficient for the minimum spec: the Lenovo ThinkCentre M920q, HP EliteDesk 800 G4, and Beelink SER5 Pro are all proven homelab choices. This is a one-time hardware purchase — a managed SOC service (MSSP) providing equivalent coverage starts at £2,000–£5,000 per month.

---

## 3. Hypervisor

The hypervisor must support multiple virtual NICs per VM, VLAN tagging on virtual switches, and snapshots before each major change. Take a snapshot before every deployment step — it is faster to roll back than to rebuild.

| Hypervisor | Status | Notes |
| ---------- | ------ | ----- |
| Proxmox VE | Recommended | Free, purpose-built for homelabs, excellent VLAN bridge support, active community |
| VMware Workstation / ESXi | Supported | Used in the original build; Workstation Pro is now free for personal use |
| Hyper-V | Supported | Works; VLAN configuration on the virtual switch is more involved than the others |
| VirtualBox | Evaluation only | VLAN support is limited; not suitable for the full network topology in section 5 |

The deployment guides in this project were written on VMware. Proxmox is recommended for new builds — snapshot performance is better and the documentation is more comprehensive.

---

## 4. Per-VM Resource Allocation

| VM | vCPU | RAM | Disk | OS |
| -- | ---- | --- | ---- | -- |
| OPNsense | 2 | 4 GB | 30 GB | OPNsense 25.x |
| Wazuh | 4 | 8 GB | 100 GB | Ubuntu 22.04 LTS |
| n8n | 2 | 4 GB | 40 GB | Ubuntu 22.04 LTS |
| MISP | 2 | 6 GB | 60 GB | Ubuntu 22.04 LTS |
| DFIR IRIS | 2 | 4 GB | 40 GB | Ubuntu 22.04 LTS |
| **Total** | **12 vCPU** | **26 GB** | **270 GB** | |

> **Warning:** Do not under-allocate Wazuh RAM. The Wazuh Indexer (OpenSearch) will silently OOM-kill itself below 8 GB, producing a dashboard that loads but shows no alerts — a failure mode with no obvious error message and a frustrating diagnosis path.

---

## 5. Network Plan

### VLAN Assignments

| VLAN | Name | Subnet | Gateway | Purpose |
| ---- | ---- | ------ | ------- | ------- |
| VLAN 10 | Management | 10.10.1.0/24 | 10.10.1.254 | Laptop and hypervisor management access |
| VLAN 20 | SOC | 10.10.10.0/24 | 10.10.10.254 | All SOC tool VMs |
| VLAN 30 | Victim | 10.10.20.0/24 | 10.10.20.254 | Test endpoints running Wazuh agents |

### Topology

```text
                        Internet
                            │
                     ┌──────┴──────┐
                     │  OPNsense   │  (gateway + IDS for all VLANs)
                     └──────┬──────┘
           ┌────────────────┼────────────────┐
           │                │                │
      VLAN 10           VLAN 20          VLAN 30
   Management             SOC             Victim
  10.10.1.0/24       10.10.10.0/24    10.10.20.0/24
  GW: 10.10.1.254   GW: 10.10.10.254 GW: 10.10.20.254
           │                │                │
        Laptop           Wazuh          Test Endpoint
                           n8n          (Wazuh agent)
                          MISP
                       DFIR IRIS
```

### SOC VM IP Allocations (VLAN 20)

| VM | Hostname | IP Address |
| -- | -------- | ---------- |
| Wazuh | wazuh | 10.10.10.10 |
| n8n | n8n | 10.10.10.20 |
| MISP | misp | 10.10.10.30 |
| DFIR IRIS | iris | 10.10.10.40 |

### Required Firewall Rules

Configure these rules on OPNsense before bringing up any SOC VM. The default inter-VLAN policy is deny.

| Direction | Source | Destination | Port(s) | Protocol | Reason |
| --------- | ------ | ----------- | ------- | -------- | ------ |
| VLAN 30 → VLAN 20 | 10.10.20.0/24 | 10.10.10.10 | 1514, 1515 | TCP/UDP | Wazuh agent log shipping and registration |
| VLAN 10 → VLAN 20 | 10.10.1.0/24 | 10.10.10.0/24 | 443 | TCP | Management HTTPS access to all SOC dashboards |
| VLAN 20 → WAN | 10.10.10.0/24 | any | 443 | TCP | Outbound API calls to VirusTotal, AbuseIPDB, URLScan.io |
| Any → Any | any | any | any | any | **Deny** — default block on all other inter-VLAN traffic |

---

## 6. Ubuntu VM Prerequisites

Apply the following to each Ubuntu VM (Wazuh, n8n, MISP, DFIR IRIS) before running any tool-specific deployment guide.

* **Ubuntu 22.04 LTS** — fully patched (`apt update && apt upgrade -y`) before installing anything else
* **SSH access** — key-based authentication only; disable password auth in `/etc/ssh/sshd_config` (`PasswordAuthentication no`)
* **Static IP** — configured via Netplan, matching the IP allocation table in section 5
* **Hostname** — set to the role name (`wazuh`, `n8n`, `misp`, `iris`) using `hostnamectl set-hostname <name>`
* **Docker Engine** — installed from the official Docker apt repository, not the `docker.io` package from Ubuntu's default repos; the bundled package lags several major versions and causes Compose v2 compatibility issues
* **Docker Compose v2** — installed as part of the Docker Engine package; verify with `docker compose version` (note: no hyphen)
* **ufw** — enabled with a default-deny inbound policy; open only the ports each tool requires
* **unattended-upgrades** — configured to apply security patches automatically
* **NTP** — time synchronisation confirmed via `timedatectl status`; log correlation across VMs breaks silently if clocks drift even slightly

---

## 7. External Accounts and API Keys

All four services have a free tier that is sufficient for this lab.

| Service | Free Tier Limits | Used By | Required |
| ------- | ---------------- | ------- | -------- |
| VirusTotal | 4 requests/min, 500/day | n8n enrichment workflows | Yes |
| AbuseIPDB | 1,000 requests/day | n8n enrichment workflows | Yes |
| URLScan.io | 100 public scans/day | n8n enrichment workflows | Optional |
| MISP feeds | Unlimited (open community feeds) | MISP threat intelligence | Yes |

**API key handling:** All keys are stored in n8n's built-in credential store. They must never be hardcoded into workflow JSON. n8n exports credentials as empty placeholders when a workflow is shared, but any key written directly into a node's configuration field will appear in plaintext in the export.

---

## 8. Skills Assumed

This project assumes a working baseline and does not teach from zero.

* Linux server installation and basic administration — file permissions, systemd services, package management
* Reading and editing a `docker-compose.yml` file
* Configuring VLANs and firewall rules through a web UI
* SSH access and shell navigation
* Reading API documentation and testing endpoints with `curl`

If any of these are gaps, address them before starting. The deployment guides reference these skills without explaining them.

---

## 9. Pre-Deployment Checklist

Complete every item before moving on to `01-opnsense.md`.

- [ ] Hypervisor installed and operational on the host machine
- [ ] Host has at least 32 GB RAM and 500 GB storage available
- [ ] Five VMs created with resource allocations matching section 4
- [ ] Three VLANs configured on the hypervisor's virtual switch (VLAN 10, 20, 30)
- [ ] OPNsense VM has at least two vNICs (WAN and LAN/trunk)
- [ ] Ubuntu 22.04 LTS ISO downloaded and checksum verified
- [ ] Ubuntu VMs installed and reachable via SSH
- [ ] Each Ubuntu VM has a static IP matching the allocation table in section 5
- [ ] Each Ubuntu VM hostname set to its role name
- [ ] Docker Engine installed from the official Docker apt repository on all Ubuntu VMs
- [ ] `docker compose version` returns v2.x on all Ubuntu VMs
- [ ] `ufw` enabled with default-deny inbound on all Ubuntu VMs
- [ ] NTP synchronised on all VMs — `timedatectl status` shows no drift warning
- [ ] VirusTotal account created and API key saved securely
- [ ] AbuseIPDB account created and API key saved securely
- [ ] URLScan.io account created (optional)
- [ ] No API keys committed to git, written into `.env` files tracked by git, or stored in plaintext

---
