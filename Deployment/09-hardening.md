# 09 — Hardening

## Overview

This guide locks down the SOC stack after all components are deployed and operational. It covers TLS for each service, service-specific ufw rules, Docker security settings, credential rotation, and a final verification pass.

Run this guide after completing `07-ansible.md` and `08-prometheus-grafana.md`. The Ansible hardening playbook applied the SSH and ufw baseline — this guide adds the service-level controls on top of that foundation.

---

## 1. Before You Begin

| Requirement | Status to Confirm |
| ----------- | ----------------- |
| All six SOC services running | Each web UI reachable over HTTP |
| Ansible hardening playbook complete | `ufw status` shows default deny incoming on all hosts |
| Node Exporter installed on all hosts | `curl http://<ip>:9100/metrics` returns metric lines |
| Valid domain or internal DNS for each service | Optional — self-signed certs work without DNS |

> **Order matters.** Apply TLS to a service before restricting its ufw rule to HTTPS only. If you reverse the order, you will lock yourself out of the web UI.

---

## 2. TLS — Self-Signed Certificates

Each service in this stack is accessed over a private VLAN with no internet-facing exposure. Self-signed certificates are appropriate here — they encrypt the connection without requiring a CA or public domain name. Your browser will show a warning; add a permanent exception.

For production use or if you have an internal CA, replace the self-signed approach with certificates signed by your CA. The Docker Compose bind-mount paths below are the same regardless of how the certificate was issued.

### Generate a Certificate per Host

Run the following on each host, substituting the IP and hostname for that service. Alternatively, run this from Ansible to generate all certificates in one pass.

```bash
# Run on the host being secured, or generate centrally and distribute
SERVICE_IP="10.10.10.10"     # change per host
SERVICE_NAME="wazuh"         # change per host

openssl req -x509 -newkey rsa:4096 -sha256 -days 825 -nodes \
  -keyout /opt/${SERVICE_NAME}/certs/${SERVICE_NAME}.key \
  -out /opt/${SERVICE_NAME}/certs/${SERVICE_NAME}.crt \
  -subj "/CN=${SERVICE_NAME}.soc.lab" \
  -addext "subjectAltName=IP:${SERVICE_IP},DNS:${SERVICE_NAME}.soc.lab"
```

Repeat for each service:

| Service | `SERVICE_IP` | `SERVICE_NAME` |
| ------- | ------------ | -------------- |
| Wazuh | 10.10.10.10 | wazuh |
| MISP | 10.10.10.20 | misp |
| n8n | 10.10.10.30 | n8n |
| DFIR IRIS | 10.10.10.40 | iris |
| Grafana | 10.10.10.50 | grafana |

> Wazuh manages its own internal TLS between the Manager, Indexer, and Dashboard using certificates generated during the Docker Compose deployment. Do not replace those internal certs — only add TLS termination at the reverse proxy layer if you choose to front Wazuh with nginx. If you access Wazuh via the built-in HTTPS that ships with the stack, no further TLS work is needed.

---

## 3. ufw — Service-Specific Rules

The Ansible hardening playbook set the baseline (`default deny` + `SSH from VLAN 10 only`). Each service needs its own additional rules. Apply these on the relevant host.

### Wazuh (`10.10.10.10`)

```bash
# Wazuh Dashboard — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 443 proto tcp comment "Wazuh Dashboard HTTPS"

# Wazuh agent registration and log shipping — Victim VLAN
sudo ufw allow from 10.10.20.0/24 to any port 1514 proto tcp comment "Wazuh agent log shipping"
sudo ufw allow from 10.10.20.0/24 to any port 1515 proto tcp comment "Wazuh agent registration"
sudo ufw allow from 10.10.20.0/24 to any port 1514 proto udp comment "Wazuh agent syslog UDP"

# Wazuh API — n8n and Ansible only
sudo ufw allow from 10.10.10.30 to any port 55000 proto tcp comment "n8n Wazuh API"
sudo ufw allow from 10.10.10.60 to any port 55000 proto tcp comment "Ansible Wazuh API"

# Node Exporter — monitoring LXC only
sudo ufw allow from 10.10.10.50 to any port 9100 proto tcp comment "Prometheus Node Exporter"

sudo ufw reload
```

### MISP (`10.10.10.20`)

```bash
# MISP web UI — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 443 proto tcp comment "MISP HTTPS"

# MISP API — n8n only
sudo ufw allow from 10.10.10.30 to any port 443 proto tcp comment "n8n MISP API"

# Node Exporter — monitoring LXC only
sudo ufw allow from 10.10.10.50 to any port 9100 proto tcp comment "Prometheus Node Exporter"

sudo ufw reload
```

### n8n (`10.10.10.30`)

```bash
# n8n web UI — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 5678 proto tcp comment "n8n UI"

# n8n webhook — Wazuh and OPNsense (for inbound alerts)
sudo ufw allow from 10.10.10.10 to any port 5678 proto tcp comment "Wazuh webhook"
sudo ufw allow from 10.10.1.254 to any port 5678 proto tcp comment "OPNsense webhook"

# Node Exporter — monitoring LXC only
sudo ufw allow from 10.10.10.50 to any port 9100 proto tcp comment "Prometheus Node Exporter"

sudo ufw reload
```

### DFIR IRIS (`10.10.10.40`)

```bash
# IRIS web UI — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 443 proto tcp comment "IRIS HTTPS"

# IRIS API — n8n only
sudo ufw allow from 10.10.10.30 to any port 443 proto tcp comment "n8n IRIS API"

# Node Exporter — monitoring LXC only
sudo ufw allow from 10.10.10.50 to any port 9100 proto tcp comment "Prometheus Node Exporter"

sudo ufw reload
```

### Grafana + Prometheus (`10.10.10.50`)

```bash
# Grafana — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 3000 proto tcp comment "Grafana UI"

# Prometheus — management VLAN only (do not expose to SOC VLAN broadly)
sudo ufw allow from 10.10.1.0/24 to any port 9090 proto tcp comment "Prometheus UI"

# Alertmanager — management VLAN only
sudo ufw allow from 10.10.1.0/24 to any port 9093 proto tcp comment "Alertmanager UI"

# Node Exporter — self (Prometheus scrapes itself)
sudo ufw allow from 127.0.0.1 to any port 9100 proto tcp comment "Prometheus self-scrape"

sudo ufw reload
```

Verify the final ufw state on each host:

```bash
sudo ufw status verbose
```

No rule should allow traffic from `any` (the wildcard source) except where explicitly intended.

---

## 4. Docker Hardening

Apply the following on every host running Docker Compose stacks.

### 4.1 Run Containers as Non-Root

Each Compose stack should specify a non-root `user`. Check each `docker-compose.yml`:

```bash
grep -r "user:" /opt/*/docker-compose.yml
```

For any service missing a `user:` directive, add one if the upstream image supports it. Most official images document the UID/GID to use.

### 4.2 Read-Only Root Filesystems

Where possible, add `read_only: true` to service definitions in `docker-compose.yml` and mount specific writable paths as named volumes. This limits what a compromised container can write to disk.

### 4.3 Restrict Docker API Access

Docker's socket (`/var/run/docker.sock`) grants root-equivalent access to the host. Confirm no service in the stack bind-mounts the socket unless explicitly required:

```bash
grep -r "docker.sock" /opt/*/docker-compose.yml
```

If a service requires socket access (e.g., a container management UI), scope the permissions using a Docker socket proxy rather than exposing the raw socket.

### 4.4 Log Driver Limits

Unbounded container logs will fill the disk over time. Set a global log rotation policy in `/etc/docker/daemon.json` on each Docker host:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

Restart Docker after applying:

```bash
sudo systemctl restart docker
```

---

## 5. Credential Rotation

### 5.1 Wazuh API Password

The Wazuh API user `wazuh-wui` is used by n8n and the Wazuh Dashboard internally.

```bash
# From within the Wazuh Manager container
docker exec -it wazuh.manager /var/ossec/bin/wazuh-control stop
docker exec -it wazuh.manager bash -c "echo 'wazuh-wui:<new_password>' | /var/ossec/bin/manage_agents -a"
```

After rotating, update the credential in n8n under **Settings > Credentials > Wazuh API**.

### 5.2 MISP API Key

MISP API keys are bound to a user account. Rotate by generating a new key:

1. Log into MISP as the API user
2. **My Profile > Auth Keys > Add authentication key**
3. Copy the new key
4. Update n8n: **Settings > Credentials > MISP**
5. Delete the old key from MISP

### 5.3 DFIR IRIS API Key

1. Log into IRIS as the API user
2. **My Account > API key > Regenerate**
3. Copy the new key
4. Update n8n: **Settings > Credentials > DFIR IRIS**

### 5.4 OPNsense API Key

1. Log into OPNsense as the API user
2. **System > Access > Users > Edit user > API keys > Add**
3. Download the new key and secret
4. Update n8n: **Settings > Credentials > OPNsense API**
5. Delete the old API key pair from the OPNsense user

### 5.5 Grafana Admin Password

```bash
# From within the Grafana container on the monitoring LXC
docker exec -it grafana grafana-cli admin reset-admin-password <new_password>
```

Or change via the Grafana UI: **Profile > Change Password**.

### 5.6 n8n Encryption Key

n8n encrypts stored credentials with a key set via the `N8N_ENCRYPTION_KEY` environment variable in its `docker-compose.yml`. If the key is the default or was never set, set one now:

```bash
# Generate a strong key
openssl rand -hex 32
```

Add it to `/opt/n8n/docker-compose.yml` under the n8n service environment:

```yaml
environment:
  - N8N_ENCRYPTION_KEY=<generated_key>
```

Then restart:

```bash
cd /opt/n8n && docker compose up -d
```

> Rotating the `N8N_ENCRYPTION_KEY` after credentials have been saved will make all existing saved credentials unreadable. Either set this key before saving any credentials, or rotate it and re-enter all credentials afterward.

---

## 6. OPNsense Hardening

### 6.1 Disable Web UI on WAN

Confirm the OPNsense web UI is not reachable from WAN:

**System > Settings > Administration > TCP port** — ensure the GUI is bound to the LAN interface only, not WAN.

### 6.2 Management Interface Lockdown

The OPNsense API is used by n8n to add/remove firewall blocks. Restrict API access:

**System > Access > Users > [API user]** — confirm the user's allowed subnets include only `10.10.10.30/32` (n8n).

### 6.3 Suricata — Block Mode

If Suricata is running in IDS (alert-only) mode, switch it to IPS (inline blocking) mode once the false positive rate is acceptable:

**Services > Intrusion Detection > Settings > Mode > IPS**

Run in IDS mode for at least 72 hours after deployment and review all alerts before enabling inline blocking — blocking mode can interrupt legitimate traffic if rules are not tuned.

---

## 7. Final Verification Pass

Run the following checks after completing this guide. Every item must pass before the SOC is considered hardened.

### SSH

```bash
# From management laptop — must succeed
ssh ansible@10.10.10.10

# From Victim VLAN — must fail
ssh ansible@10.10.10.10  # run from 10.10.20.x
```

### ufw

```bash
# On each host
sudo ufw status verbose
# Expected: default deny incoming, deny outgoing blocked, only listed ports open
```

### Exposed Ports

Scan each host from the management laptop to confirm only expected ports are open:

```bash
# Install nmap if not present
sudo apt install -y nmap

# Scan Wazuh (example)
nmap -sS -p 1-65535 10.10.10.10
```

Expected open ports per host:

| Host | Expected Open Ports |
| ---- | ------------------- |
| Wazuh | 22, 443, 1514, 1515, 9100, 55000 |
| MISP | 22, 443, 9100 |
| n8n | 22, 5678, 9100 |
| DFIR IRIS | 22, 443, 9100 |
| Grafana + Prometheus | 22, 3000, 9090, 9093, 9100 |

Any port not in this table that shows as open should be investigated and closed.

### Docker Socket

```bash
# On each Docker host — should return nothing (no match)
grep -r "docker.sock" /opt/*/docker-compose.yml
```

### Credential Validity

From n8n, run a test on each credential (**Settings > Credentials > three-dot menu > Test**). All must pass after rotation. A failed credential means the rotation was not propagated to n8n.

---

## 8. Final Checklist

**TLS**
- [ ] Self-signed certificates generated for each service
- [ ] Certificates bind-mounted into each Compose stack
- [ ] All services accessible over HTTPS — browser warning accepted, not bypassed by switching to HTTP

**ufw**
- [ ] Service-specific rules applied on each host
- [ ] `ufw status verbose` shows default deny incoming on all hosts
- [ ] No rule with source `any` except where explicitly justified

**Docker**
- [ ] Log rotation configured in `/etc/docker/daemon.json` on all Docker hosts
- [ ] Docker socket not exposed to any container unless required
- [ ] `docker compose ps` shows all services healthy after Docker restart

**Credential rotation**
- [ ] Wazuh API password rotated and updated in n8n
- [ ] MISP API key rotated and updated in n8n
- [ ] DFIR IRIS API key rotated and updated in n8n
- [ ] OPNsense API key rotated and updated in n8n
- [ ] Grafana admin password changed from default
- [ ] n8n encryption key set in environment before or after re-entering credentials

**OPNsense**
- [ ] Web UI not accessible from WAN interface
- [ ] API user restricted to n8n source IP only
- [ ] Suricata running (IDS mode minimum; IPS mode after tuning)

**Port scan**
- [ ] nmap scan from management laptop — all hosts show only expected ports

**End-to-end test**
- [ ] Simulated brute force from `06-integration.md` repeated — full pipeline executes correctly over HTTPS
- [ ] n8n credentials all pass test after rotation
- [ ] Snapshot taken of all VMs and LXC containers after completing this guide

---
