# Prerequisites — The Bridge Between SOC Lab

## Overview

This document covers everything that must be in place before deploying any individual tool. Work through it top to bottom. Each subsequent deployment guide assumes this baseline is complete.

---

## 1. Deployment Model

This lab runs across two Proxmox hosts and a bare-metal OPNsense firewall. Proxmox 1 hosts the heavier SIEM and threat intel workloads as full VMs. Proxmox 2 hosts the lighter automation and observability tools as LXC containers, which is sufficient for their workloads and reduces overhead.

### Physical Hosts

| Host | Hardware Role | What Runs On It |
| ---- | ------------- | --------------- |
| OPNsense | Bare metal firewall appliance | OPNsense 25.x — needs direct NIC access, cannot run inside a hypervisor |
| Proxmox 1 | Heavy workload host | Wazuh (VM), MISP (VM) |
| Proxmox 2 | Automation + observability host | n8n (LXC), DFIR IRIS (LXC), Grafana + Prometheus (LXC), Ansible (LXC or management laptop) |

### Per-Service Deployment Method

| Service | Host | Type | Deployment Method | Reason |
| ------- | ---- | ---- | ----------------- | ------ |
| OPNsense | Bare metal | OS install | Native install | Firewall distros need direct NIC access |
| Wazuh | Proxmox 1 | VM (Ubuntu 22.04) | Docker Compose | The official stack bundles Indexer, Manager, and Dashboard in one Compose file |
| MISP | Proxmox 1 | VM (Ubuntu 22.04) | Docker Compose | MISP-Docker is the maintained route; manual installs are a dependency nightmare |
| n8n | Proxmox 2 | LXC (Ubuntu 22.04) | Docker Compose | Lightweight; container overhead is negligible for a workflow engine |
| DFIR IRIS | Proxmox 2 | LXC (Ubuntu 22.04) | Docker Compose | IRIS ships as a first-class Compose stack |
| Grafana + Prometheus | Proxmox 2 | LXC (Ubuntu 22.04) | Docker Compose | Both run as a single Compose stack; lightweight enough for a container |
| Ansible | Proxmox 2 or management laptop | LXC or native | Package install | Agentless — runs playbooks from wherever it has SSH access to targets |

> **Note on Ansible placement:** Running Ansible on your management laptop is fine during active development. Moving it to a Proxmox 2 LXC makes sense once playbooks are stable — it keeps the management plane off your personal machine and ensures playbooks can run without your laptop being on.

---

## 2. Host Hardware Requirements

### OPNsense (Bare Metal)

| | Minimum | Recommended |
| -- | ------- | ----------- |
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Storage | 30 GB SSD | 60 GB SSD |
| Network | 2 NICs (WAN + LAN) | 3 NICs (WAN + LAN + dedicated management) |

A low-power dedicated appliance such as a Protectli VP2420 or a repurposed thin client with two NICs works well. The machine must have at least two physical NICs.

### Proxmox 1 — Heavy Workloads (Wazuh + MISP)

| | Minimum | Recommended |
| -- | ------- | ----------- |
| CPU | 6 cores | 8 cores |
| RAM | 32 GB | 64 GB |
| Storage | 300 GB SSD | 500 GB NVMe |
| Network | 1 GbE NIC | 2.5 GbE NIC |

Wazuh's Indexer (OpenSearch) alone will consume 16 GB RAM under any real alert volume. This host needs to be spec'd accordingly — do not try to cram both Wazuh and MISP onto a 16 GB machine.

### Proxmox 2 — Automation + Observability (n8n, IRIS, Grafana/Prometheus, Ansible)

| | Minimum | Recommended |
| -- | ------- | ----------- |
| CPU | 4 cores | 6 cores |
| RAM | 16 GB | 32 GB |
| Storage | 200 GB SSD | 300 GB SSD |
| Network | 1 GbE NIC | 1 GbE NIC |

The services on this host are significantly lighter than the SIEM stack. A second-hand mini-PC in the **£50–£100** range is sufficient: Lenovo M720q, HP EliteDesk 800 G3, or similar.

---

## 3. Hypervisor

Both Proxmox hosts run **Proxmox VE 8.x**. Proxmox is free, purpose-built for homelabs, has excellent VLAN bridge support, and handles LXC containers natively alongside KVM VMs — which is exactly what this split deployment needs.

**Key Proxmox configuration to do before deploying any service:**

- Create a Linux Bridge on each host (`vmbr0` for LAN, `vmbr1` for WAN if applicable)
- Enable VLAN-aware mode on the bridge used for SOC traffic
- Configure the Proxmox backup schedule to run nightly CT/VM snapshots — this is your rollback path; take a manual snapshot before every deployment step as well
- Set up Proxmox cluster between the two hosts if you want live migration capability (optional but useful)

---

## 4. Per-Service Resource Allocation

### Proxmox 1 — VMs

| VM | vCPU | RAM | Disk | OS |
| -- | ---- | --- | ---- | -- |
| Wazuh | 4 | **16 GB** | 150 GB | Ubuntu 22.04 LTS |
| MISP | 2 | 6 GB | 60 GB | Ubuntu 22.04 LTS |
| **P1 Total** | **6 vCPU** | **22 GB** | **210 GB** | |

> **Wazuh RAM warning:** The Wazuh Indexer (OpenSearch) requires a minimum of 16 GB to function reliably under real alert volume. Below this threshold it will OOM-kill itself silently — the dashboard loads but shows no alerts, with no obvious error message. This is one of the most common and frustrating failure modes in Wazuh deployments. Do not compromise on this allocation.

### Proxmox 2 — LXC Containers

| Container | vCPU | RAM | Disk | OS Template |
| --------- | ---- | --- | ---- | ----------- |
| n8n | 2 | 4 GB | 40 GB | Ubuntu 22.04 |
| DFIR IRIS | 2 | 4 GB | 40 GB | Ubuntu 22.04 |
| Grafana + Prometheus | 2 | 4 GB | 50 GB | Ubuntu 22.04 |
| Ansible | 1 | 2 GB | 20 GB | Ubuntu 22.04 |
| **P2 Total** | **7 vCPU** | **14 GB** | **150 GB** | |

> **LXC vs VM:** n8n, IRIS, Grafana, and Ansible do not require kernel-level isolation and run fine in unprivileged LXC containers. Docker runs inside LXC containers on Proxmox — enable nesting (`features: nesting=1`) in the container options before installing Docker.

---

## 5. Network Plan

### VLAN Assignments

| VLAN | Name | Subnet | Gateway | Purpose |
| ---- | ---- | ------ | ------- | ------- |
| VLAN 10 | Management | 10.10.1.0/24 | 10.10.1.254 | Management laptop, Proxmox host UIs, Ansible |
| VLAN 20 | SOC | 10.10.10.0/24 | 10.10.10.254 | All SOC tool VMs and containers |
| VLAN 30 | Victim | 10.10.20.0/24 | 10.10.20.254 | Test endpoints running Wazuh agents |

### Topology

```text
                        Internet
                            │
                     ┌──────┴──────┐
                     │  OPNsense   │  (bare metal — gateway + IDS for all VLANs)
                     └──────┬──────┘
           ┌────────────────┼────────────────┐
           │                │                │
      VLAN 10           VLAN 20          VLAN 30
   Management             SOC             Victim
  10.10.1.0/24       10.10.10.0/24    10.10.20.0/24
           │                │                │
        Laptop          Proxmox 1        Test Endpoint
        Ansible          ├─ Wazuh         (Wazuh agent)
      (or P2 LXC)        └─ MISP
                        Proxmox 2
                          ├─ n8n
                          ├─ DFIR IRIS
                          ├─ Grafana + Prometheus
                          └─ Ansible (LXC)
```

### IP Allocations (VLAN 20 — SOC)

| Service | Host | Hostname | IP Address |
| ------- | ---- | -------- | ---------- |
| Wazuh | Proxmox 1 VM | wazuh | 10.10.10.10 |
| MISP | Proxmox 1 VM | misp | 10.10.10.20 |
| n8n | Proxmox 2 LXC | n8n | 10.10.10.30 |
| DFIR IRIS | Proxmox 2 LXC | iris | 10.10.10.40 |
| Grafana + Prometheus | Proxmox 2 LXC | monitoring | 10.10.10.50 |
| Ansible | Proxmox 2 LXC | ansible | 10.10.10.60 |

### IP Allocations (VLAN 10 — Management)

| Device | IP Address |
| ------ | ---------- |
| Proxmox 1 (host UI) | 10.10.1.10 |
| Proxmox 2 (host UI) | 10.10.1.11 |
| Management laptop | 10.10.1.100 |

### Required Firewall Rules

Configure these rules on OPNsense before bringing up any SOC service. The default inter-VLAN policy is deny.

| Direction | Source | Destination | Port(s) | Protocol | Reason |
| --------- | ------ | ----------- | ------- | -------- | ------ |
| VLAN 30 → VLAN 20 | 10.10.20.0/24 | 10.10.10.10 | 1514, 1515 | TCP/UDP | Wazuh agent log shipping and registration |
| VLAN 10 → VLAN 20 | 10.10.1.0/24 | 10.10.10.0/24 | 443 | TCP | Management HTTPS access to all SOC dashboards |
| VLAN 10 → VLAN 10 | 10.10.1.100 | 10.10.1.10–11 | 8006 | TCP | Management laptop access to Proxmox UIs |
| VLAN 20 → WAN | 10.10.10.0/24 | any | 443 | TCP | Outbound CTI API calls (AbuseIPDB, OTX, abuse.ch, URLScan.io) |
| VLAN 10 → VLAN 20 | 10.10.1.0/24 | 10.10.10.0/24 | 22 | TCP | Ansible SSH access from management VLAN to SOC VMs |
| Any → Any | any | any | any | any | **Deny** — default block on all other inter-VLAN traffic |

---

## 6. Host OS Prerequisites

### Deployment Method — Docker Engine and Docker Compose

Every tool in this stack (Wazuh, MISP, n8n, DFIR IRIS, Grafana + Prometheus) is deployed as a **Docker Compose stack**. Docker Engine and Docker Compose v2 must be installed on every VM and LXC container before running any tool-specific guide.

> **Use the official Docker apt repository — not `docker.io`.** The `docker.io` package from Ubuntu's default repos is outdated and will cause compatibility issues with the Compose stacks used here.

Official installation reference: [https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

Run the following on each VM and LXC container:

```bash
# Remove any old Docker packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y $pkg 2>/dev/null; done

# Add Docker's official GPG key and apt repository
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
docker compose version
```

Docker Compose v2 is installed as a plugin (`docker compose`, no hyphen). Verify the version returned is v2.x before continuing.

---

### Ubuntu VMs (Proxmox 1 — Wazuh, MISP)

Apply the following to each Ubuntu VM before running any tool-specific guide.

- **Ubuntu 22.04 LTS** — fully patched (`apt update && apt upgrade -y`) before installing anything
- **SSH access** — key-based authentication only; disable password auth (`PasswordAuthentication no` in `/etc/ssh/sshd_config`)
- **Static IP** — configured via Netplan, matching the IP allocation table in section 5
- **Hostname** — set to the role name (`wazuh`, `misp`) via `hostnamectl set-hostname <name>`
- **Docker Engine + Compose v2** — installed per the block above
- **ufw** — enabled with default-deny inbound; open only the ports each tool requires
- **NTP** — time sync confirmed via `timedatectl status`; log correlation breaks silently if clocks drift

### LXC Containers (Proxmox 2 — n8n, IRIS, Grafana/Prometheus, Ansible)

- Use the **Ubuntu 22.04 LXC template** from the Proxmox template library
- Enable **nesting** in the container options (`features: nesting=1`) — required for Docker to run inside LXC
- Set a **static IP** via the Proxmox network config or inside the container
- Set the **hostname** to the service name (`n8n`, `iris`, `monitoring`, `ansible`)
- Install Docker Engine + Compose v2 per the block above — the Docker apt repo works identically inside LXC
- NTP is handled by the Proxmox host by default; verify with `timedatectl` inside each container

### Ansible Target Requirements

All VMs and containers that Ansible manages need:
- SSH server running (`openssh-server` installed)
- A dedicated Ansible service account with passwordless sudo
- The Ansible controller's SSH public key in `~/.ssh/authorized_keys`
- Python 3 installed (required by Ansible modules)

---

## 7. Threat Intelligence APIs

All services listed here have a **genuinely free tier** with no credit card required. n8n stores all credentials in its built-in credential store — you do not need to handle keys anywhere else.

| Service | What It Provides | Rate Limit (Free) | Used For |
| ------- | ---------------- | ----------------- | -------- |
| [AbuseIPDB](https://www.abuseipdb.com) | IP reputation scoring | 1,000 req/day | IP enrichment on all alert types |
| [OTX AlienVault](https://otx.alienvault.com) | IP, domain, hash, URL intel | No hard limit | Broad IOC enrichment across all alert types |
| [abuse.ch MalwareBazaar](https://bazaar.abuse.ch/api/) | Malware hash lookup | No hard limit | File hash enrichment on FIM alerts |
| [abuse.ch URLhaus](https://urlhaus-api.abuse.ch/) | Malicious URL and domain lookup | No hard limit | URL/domain enrichment on network alerts |
| [abuse.ch ThreatFox](https://threatfox.abuse.ch/api/) | IOC lookup (IPs, domains, hashes, URLs) | No hard limit | General IOC enrichment |
| [URLScan.io](https://urlscan.io) | URL sandbox scan | 100 public scans/day | Optional — deeper URL analysis |

> **Credential storage:** All API keys are configured directly in n8n's **Credentials** section (Settings → Credentials). n8n encrypts them at rest and injects them into workflows at runtime. There is no need to store keys in environment files, `.env` files, or any external secret manager for this lab.

---

## 8. Skills Assumed

This project assumes a working baseline and does not teach from zero.

- Linux server administration — file permissions, systemd services, package management
- Reading and editing a `docker-compose.yml` file
- Configuring VLANs and firewall rules through a web UI
- SSH access and shell navigation
- Basic Ansible — running a playbook, understanding inventory files
- Reading API documentation and testing endpoints with `curl`

---

## 9. Pre-Deployment Checklist

Complete every item before moving on to `01-opnsense.md`.

**Infrastructure**
- [ ] OPNsense installed bare metal with at least 2 physical NICs
- [ ] Proxmox VE 8.x installed on both Proxmox hosts
- [ ] Proxmox 1 has at least 32 GB RAM and 300 GB storage
- [ ] Proxmox 2 has at least 16 GB RAM and 200 GB storage
- [ ] VLAN-aware Linux Bridge configured on both Proxmox hosts

**VMs and Containers**
- [ ] Wazuh VM created on Proxmox 1 — 4 vCPU, **16 GB RAM**, 150 GB disk
- [ ] MISP VM created on Proxmox 1 — 2 vCPU, 6 GB RAM, 60 GB disk
- [ ] n8n LXC created on Proxmox 2 — 2 vCPU, 4 GB RAM, 40 GB disk, nesting enabled
- [ ] DFIR IRIS LXC created on Proxmox 2 — 2 vCPU, 4 GB RAM, 40 GB disk, nesting enabled
- [ ] Grafana + Prometheus LXC created on Proxmox 2 — 2 vCPU, 4 GB RAM, 50 GB disk, nesting enabled
- [ ] Ansible LXC (or management laptop) ready with Ansible installed

**Networking**
- [ ] Three VLANs configured (VLAN 10 Management, VLAN 20 SOC, VLAN 30 Victim)
- [ ] Static IPs assigned and matching the allocation table in section 5
- [ ] Each host and container reachable via SSH from the management laptop
- [ ] Firewall rules configured in OPNsense per section 5

**OS Baseline**
- [ ] Ubuntu 22.04 LTS installed and fully patched on both VMs
- [ ] SSH key-based auth configured; password auth disabled
- [ ] Docker Engine installed via the official apt repository (section 6) on all VMs and LXC containers
- [ ] `docker compose version` returns v2.x on all VMs and LXC containers
- [ ] NTP synchronised — `timedatectl status` shows no drift on all hosts
- [ ] Ansible SSH access confirmed to all target VMs and containers

**Threat Intelligence**
- [ ] AbuseIPDB account created
- [ ] OTX AlienVault account created
- [ ] abuse.ch services noted (MalwareBazaar, URLhaus, ThreatFox)
- [ ] URLScan.io account created (optional)
- [ ] All API keys ready to enter into n8n Credentials — not stored anywhere else

---
