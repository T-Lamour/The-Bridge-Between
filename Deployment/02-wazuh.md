# Wazuh — SIEM and XDR Setup

## Overview

This guide deploys the Wazuh single-node stack on the Ubuntu VM at `10.10.10.10`. By the end of this guide you will have:

* Wazuh Manager, Indexer, and Dashboard running via Docker Compose
* Syslog receiver configured to ingest OPNsense and Suricata alerts
* One Wazuh agent enrolled from a Victim VLAN endpoint
* The Wazuh API accessible for n8n integration

Wazuh is the detection core of the stack. Everything else in the pipeline depends on it generating alerts.

---

## 1. Before You Begin

* Ubuntu 22.04 VM at `10.10.10.10` with hostname `wazuh`
* 4 vCPU and **at least 8 GB RAM** allocated — the Wazuh Indexer (OpenSearch) will silently crash below this threshold
* 100 GB disk — Wazuh stores all indexed alert data locally
* Docker Engine and Compose v2 installed from the official Docker apt repo
* OPNsense syslog forwarding configured (covered in `01-opnsense.md`)
* Outbound HTTPS permitted from VLAN 20 (to pull Docker images)

---

## 2. Static IP and Hostname

Set the static IP via Netplan before proceeding. Edit `/etc/netplan/00-installer-config.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens18:                        # replace with your interface name
      dhcp4: false
      addresses:
        - 10.10.10.10/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses: [10.10.10.254]
```

Apply and set the hostname:

```bash
sudo netplan apply
sudo hostnamectl set-hostname wazuh
```

Verify connectivity before continuing:

```bash
ping -c 3 1.1.1.1
```

---

## 3. Deploy the Wazuh Docker Stack

### 3.1 Clone the Official Repository

Wazuh publishes a maintained single-node Docker Compose stack. Always deploy from the tagged release matching your target version — do not use an untagged clone.

```bash
# Check https://github.com/wazuh/wazuh-docker/releases for the latest tag
WAZUH_VERSION=v4.9.2

git clone https://github.com/wazuh/wazuh-docker.git -b $WAZUH_VERSION
cd wazuh-docker/single-node
```

### 3.2 Generate Indexer Certificates

The Wazuh Indexer (OpenSearch) requires TLS certificates between components. The repo includes a generator that creates them automatically.

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

This creates a `config/wazuh_indexer_ssl_certs/` directory. Do not modify or delete the contents.

### 3.3 Set Passwords

Open `docker-compose.yml` and locate the environment variables for `wazuh.manager`, `wazuh.indexer`, and `wazuh.dashboard`. Change all default passwords before starting the stack. The defaults (`SecretPassword`, `kibanaserver`, etc.) are publicly known.

Record your passwords — they are needed for:
* Web UI login (admin account)
* n8n API integration (covered in `03-n8n.md`)
* The Wazuh API (covered in section 6)

### 3.4 Configure Syslog Receiver

Before starting the stack, add the syslog listener to the Wazuh Manager configuration. Create a local override file:

```bash
mkdir -p config/wazuh_cluster
```

Create `config/wazuh_cluster/local_ossec.conf` with the following content:

```xml
<ossec_config>
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>10.10.10.254</allowed-ips>
  </remote>
</ossec_config>
```

In `docker-compose.yml`, add a volume mount to the `wazuh.manager` service so this file is loaded at startup:

```yaml
volumes:
  - ./config/wazuh_cluster/local_ossec.conf:/wazuh-config-mount/var/ossec/etc/ossec.conf.d/local_ossec.conf
```

> The `.conf.d/` directory is auto-included by the Wazuh Manager. Placing config here avoids overwriting the main `ossec.conf` and survives container rebuilds.

Also confirm that port `514/udp` is mapped in the `wazuh.manager` service ports section. It should be present by default:

```yaml
ports:
  - "1514:1514"
  - "1515:1515"
  - "514:514/udp"
  - "55000:55000"
```

### 3.5 Start the Stack

```bash
docker compose up -d
```

The first start takes 3–5 minutes. The Wazuh Indexer performs setup tasks before the other services become healthy. Monitor progress:

```bash
docker compose logs -f wazuh.manager
docker compose logs -f wazuh.indexer
```

Wait until you see `[INFO] Wazuh indexer is ready` in the indexer logs before proceeding.

---

## 4. Web UI — First Login

Navigate to `https://10.10.10.10` from your management machine. Accept the self-signed certificate warning.

| Field | Value |
| ----- | ----- |
| Username | `admin` |
| Password | The password set in section 3.3 |

On first login, Wazuh will prompt you to acknowledge the EULA. Accept and proceed.

Confirm the following in the dashboard:

* **Agents** → shows zero agents (expected — none enrolled yet)
* **Security Events** → no events yet, but the module must load without errors
* **Vulnerabilities** → may show loading state until an agent reports in

---

## 5. Verify Syslog Receiver

With OPNsense already forwarding syslog to `10.10.10.10:514`, check that logs are arriving:

```bash
docker exec -it single-node-wazuh.manager-1 bash
tail -f /var/ossec/logs/ossec.log | grep syslog
```

You should see entries referencing the OPNsense SOC interface IP (`10.10.10.254`) within a few seconds. If nothing appears after 60 seconds, check the OPNsense syslog configuration from `01-opnsense.md` and confirm the firewall rule permits UDP 514 from `10.10.10.254` to `10.10.10.10`.

Navigate to **Security Events** in the dashboard and filter by `agent.name:opnsense` to confirm events appear in the UI.

---

## 6. Wazuh API

The Wazuh API is used by n8n to query alert data and trigger agent actions. It runs on port `55000`.

Verify it is accessible:

```bash
curl -k -u admin:<your-password> https://10.10.10.10:55000/
```

Expected response includes `"title": "Wazuh App REST API"` and the version string. Record this URL and credentials — they are entered into n8n in `03-n8n.md`.

---

## 7. Enrol a Wazuh Agent

Deploy an agent to a test endpoint on VLAN 30 (Victim). This validates the full agent pipeline before any incidents are simulated.

### 7.1 Generate an Enrolment Command

In the Wazuh dashboard, navigate to **Agents > Deploy New Agent**.

Select:
* OS: matching the test endpoint OS (Ubuntu/Windows/etc.)
* Architecture: amd64
* Wazuh Manager address: `10.10.10.10`

Wazuh generates a one-line install command. Copy it.

### 7.2 Run on the Test Endpoint

On a machine in VLAN 30 (`10.10.20.x`), run the generated command. It installs the agent, registers it with the Manager on port 1515, and starts it.

If the agent fails to connect, check:
* OPNsense firewall rule — VLAN 30 → `10.10.10.10` on ports 1514 and 1515
* Agent config at `/var/ossec/etc/ossec.conf` — `<address>` must be `10.10.10.10`
* Agent service status: `sudo systemctl status wazuh-agent`

### 7.3 Verify in Dashboard

Navigate to **Agents** in the Wazuh dashboard. The endpoint must appear with status **Active** within 30 seconds of the agent service starting.

---

## 8. Custom Detection Rules (Optional at This Stage)

Wazuh ships with thousands of built-in rules covering common attack patterns. The Use Cases in this project reference the following rule IDs — verify they exist before testing:

| Rule ID | Event | Severity |
| ------- | ----- | -------- |
| 60106 | Multiple authentication failures | 10 |
| 87105 | Malicious file detected (VirusTotal match) | 12 |
| 100002 | Successful login from unusual location | 9 |
| 5712 | SSH brute force attempt | 10 |

Navigate to **Management > Rules** and search each ID. If any are missing, they are added in the integration guide (`06-integration.md`).

---

## 9. Verification

| Check | Command / Location | Expected Result |
| ----- | ------------------ | --------------- |
| Stack running | `docker compose ps` | All three containers show `healthy` |
| Dashboard loads | `https://10.10.10.10` | Login page renders |
| Syslog ingesting | Security Events, filter `agent.name:opnsense` | OPNsense events visible |
| API responds | `curl -k -u admin:<pw> https://10.10.10.10:55000/` | JSON with version info |
| Agent enrolled | Dashboard > Agents | Endpoint shows Active |
| Disk usage | `df -h` | Root partition below 70% |

---

## 10. What's Next

`03-n8n.md` — deploy the n8n SOAR engine on `10.10.10.20` and configure the Wazuh webhook integration that triggers automation workflows on new alerts.

---

## 11. Post-Installation Checklist

- [ ] Ubuntu 22.04 fully patched before deployment
- [ ] Static IP `10.10.10.10/24` set via Netplan
- [ ] Hostname set to `wazuh`
- [ ] Docker Engine installed from the official apt repo
- [ ] Wazuh repository cloned from the tagged release
- [ ] Indexer certificates generated
- [ ] All default passwords changed before stack start
- [ ] Syslog listener config added to `local_ossec.conf`
- [ ] Port `514/udp` mapped in docker-compose.yml
- [ ] Stack started and all containers healthy
- [ ] Dashboard accessible at `https://10.10.10.10`
- [ ] OPNsense syslog events visible in Security Events
- [ ] Wazuh API responding on port 55000
- [ ] At least one Wazuh agent enrolled and showing Active
- [ ] Wazuh API credentials recorded for n8n setup
- [ ] Snapshot taken of the VM in this clean state

---
