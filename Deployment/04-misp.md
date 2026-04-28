# MISP — Threat Intelligence Setup

## Overview

This guide deploys MISP on the Ubuntu VM at `10.10.10.30` using the official MISP-Docker stack. By the end of this guide you will have:

* MISP running via Docker Compose with a persistent database and file storage
* An admin account with a known password
* CIRCL and abuse.ch threat intelligence feeds enabled and synchronised
* An API key ready for the n8n MISP credential

MISP serves as the internal IOC database for this SOC. It does two things: n8n queries it to check whether a suspicious indicator (IP, domain, hash) is already known, and n8n writes new confirmed IOCs back to it after incidents are closed. This feedback loop means the SOC learns from every incident.

---

## 1. Before You Begin

* Ubuntu 22.04 VM at `10.10.10.30` with hostname `misp`
* 2 vCPU and 6 GB RAM — MISP's MySQL and Redis processes are memory-hungry at rest
* 60 GB disk
* Docker Engine and Compose v2 installed
* Outbound HTTPS permitted from VLAN 20 (required to pull feeds)

---

## 2. Static IP and Hostname

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses:
        - 10.10.10.30/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses: [10.10.10.254]
```

```bash
sudo netplan apply
sudo hostnamectl set-hostname misp
```

---

## 3. Deploy MISP-Docker

### 3.1 Clone the Repository

```bash
git clone https://github.com/MISP/misp-docker.git
cd misp-docker
```

### 3.2 Configure the Environment

Copy the example environment file and edit it:

```bash
cp template.env .env
```

Open `.env` and set the following values. Leave everything else at its default for the initial deployment:

| Variable | Value to Set |
| -------- | ------------ |
| `BASE_URL` | `https://10.10.10.30` |
| `MYSQL_PASSWORD` | Strong random password |
| `MYSQL_ROOT_PASSWORD` | Strong random password (different from above) |
| `ADMIN_EMAIL` | An email address you control |
| `ADMIN_PASSWORD` | Strong password — this becomes the MISP admin password |
| `ADMIN_KEY` | Leave blank — MISP generates this on first run |
| `REDIS_PASSWORD` | Strong random password |

Generate passwords with:

```bash
openssl rand -base64 32
```

### 3.3 Start the Stack

```bash
docker compose up -d
```

MISP initialises a MySQL database and runs database migrations on first start. This takes 3–8 minutes. Monitor progress:

```bash
docker compose logs -f misp-core
```

Wait until the logs show `MISP is ready` before attempting to access the web UI. Attempting to log in during initialisation will result in errors that look like deployment failures but are not.

---

## 4. First Login

Navigate to `https://10.10.10.30` from your management machine. Accept the self-signed certificate warning.

| Field | Value |
| ----- | ----- |
| Email | The `ADMIN_EMAIL` value from `.env` |
| Password | The `ADMIN_PASSWORD` value from `.env` |

On first login MISP may prompt you to acknowledge the terms of use and confirm your organisation details. Complete this before proceeding.

---

## 5. Organisation Configuration

Navigate to **Administration > My Profile > My Organisation**.

Set your organisation details:

| Field | Value |
| ----- | ----- |
| Name | `The Bridge Between` (or your org name) |
| Description | Internal SOC homelab |
| URL | `https://10.10.10.30` |
| Nationality | Your country |

This organisation name appears in all events and IOCs created by this instance. It does not need to match any real organisation.

---

## 6. Enable Threat Intelligence Feeds

MISP can pull IOC data from external feeds. For this lab, enable two free feeds that cover the most common threats.

Navigate to **Sync Actions > Feeds**.

### 6.1 CIRCL OSINT Feed

CIRCL (Computer Incident Response Center Luxembourg) publishes a comprehensive OSINT IOC feed. It is included in MISP's default feed list.

Find `CIRCL OSINT Feed` in the list and click **Enable**. Set:

* Enabled: checked
* Caching enabled: checked
* Distribution: Your organisation only

### 6.2 abuse.ch Feeds

Enable all three abuse.ch feeds from the default list:

* `Abuse.ch Feodo Tracker` — banking trojans and botnet C2 IPs
* `Abuse.ch URLhaus` — malware distribution URLs
* `Abuse.ch MalwareBazaar` — malware file hashes

### 6.3 Fetch All Feeds

After enabling feeds, trigger an immediate fetch:

Navigate to **Sync Actions > Fetch all feeds**. This runs in the background and populates the MISP database with current IOC data. Depending on feed size, this takes 5–20 minutes on first run.

Schedule regular feed updates:

Navigate to **Administration > Scheduled Tasks**. Enable the following tasks with a daily frequency:

* Fetch feeds
* Cache feeds

---

## 7. Generate API Key for n8n

Navigate to **Administration > List Auth Keys > Add auth key**.

| Field | Value |
| ----- | ----- |
| User | Your admin account |
| Comment | `n8n integration` |
| Allowed IPs | `10.10.10.20` (n8n VM only) |
| Read only | Unchecked — n8n needs write access to create events |

Click **Submit**. Copy the API key immediately — it is only shown once. If you lose it, delete this key and generate a new one.

**Add this key to n8n now:**

In the n8n web UI, navigate to **Settings > Credentials** and complete the `MISP` credential created in `03-n8n.md`:

| Field | Value |
| ----- | ----- |
| Name | `Authorization` |
| Value | `<your-misp-api-key>` |

---

## 8. MISP in the SOC Pipeline

n8n queries MISP at two points in each enrichment workflow:

* **Before enrichment** — check whether the alert's source IP, domain, or file hash already exists in MISP as a known IOC. A match elevates the alert severity immediately.
* **After incident closure** — write newly confirmed IOCs back to MISP as a MISP event, so future alerts involving the same indicator are caught earlier.

This means MISP grows more accurate with every incident the SOC handles.

---

## 9. Verify MISP API

From the n8n VM, test the API connection:

```bash
curl -k -H "Authorization: <your-api-key>" \
     -H "Accept: application/json" \
     https://10.10.10.30/attributes/restSearch \
     -d '{"returnFormat":"json","limit":1}'
```

Expected: a JSON response with `"response"` containing an attributes array (may be empty if feeds have not finished importing).

If the request is refused, check:
* The API key's allowed IP is set to `10.10.10.20`
* The MGMT → SOC firewall rule permits HTTPS from `10.10.10.20` to `10.10.10.30`

---

## 10. Verification

| Check | How to Verify | Expected |
| ----- | ------------- | -------- |
| Stack running | `docker compose ps` | All containers `running` or `healthy` |
| Web UI loads | `https://10.10.10.30` | Login page renders |
| Feeds enabled | Sync Actions > Feeds | CIRCL and abuse.ch feeds show Enabled |
| Feed data present | Search > Attributes | Results returned after feed fetch completes |
| API responds | curl test from section 9 | JSON response with no auth error |
| n8n credential updated | n8n > Settings > Credentials | MISP credential shows saved (no error icon) |

---

## 11. What's Next

`05-iris.md` — deploy DFIR IRIS on `10.10.10.40` for case management, generate the API key, and complete the final n8n credential.

---

## 12. Post-Installation Checklist

- [ ] Static IP `10.10.10.30/24` set via Netplan
- [ ] Hostname set to `misp`
- [ ] `.env` file created with strong passwords for MySQL and Redis
- [ ] `.env` not committed to git
- [ ] MISP stack started and all containers healthy
- [ ] Web UI accessible at `https://10.10.10.30`
- [ ] Admin password changed from any default value
- [ ] Organisation details configured
- [ ] CIRCL OSINT feed enabled and fetched
- [ ] abuse.ch feeds enabled and fetched
- [ ] Daily feed fetch and cache scheduled
- [ ] API key generated with IP restriction to `10.10.10.20`
- [ ] API key added to n8n MISP credential
- [ ] API connectivity verified via curl test
- [ ] Snapshot taken of the VM in this clean state

---
