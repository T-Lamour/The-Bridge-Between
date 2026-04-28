# DFIR IRIS — Case Management Setup

## Overview

This guide deploys DFIR IRIS on the Ubuntu VM at `10.10.10.40` using the official Docker Compose stack. By the end of this guide you will have:

* DFIR IRIS running via Docker Compose with persistent storage
* An admin account configured
* A customer record representing the organisation being protected
* An API key ready for the n8n IRIS credential

DFIR IRIS is the case management layer. When n8n determines that an alert is a confirmed or probable incident, it creates a structured case in IRIS — with the enriched alert data, IOC list, affected assets, and a pre-populated task checklist. Every incident that the SOC handles is tracked here from detection to closure.

---

## 1. Before You Begin

* Ubuntu 22.04 VM at `10.10.10.40` with hostname `iris`
* 2 vCPU and 4 GB RAM
* 40 GB disk
* Docker Engine and Compose v2 installed
* n8n deployed — the IRIS API key is added to n8n credentials at the end of this guide

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
        - 10.10.10.40/24
      routes:
        - to: default
          via: 10.10.10.254
      nameservers:
        addresses: [10.10.10.254]
```

```bash
sudo netplan apply
sudo hostnamectl set-hostname iris
```

---

## 3. Deploy DFIR IRIS

### 3.1 Clone the Repository

```bash
git clone https://github.com/dfir-iris/iris-web.git
cd iris-web
```

### 3.2 Configure the Environment

```bash
cp .env.model .env
```

Open `.env` and set the following. Leave everything else at the defaults for a lab deployment:

| Variable | Value to Set |
| -------- | ------------ |
| `IRIS_SECRET_KEY` | 32+ character random string |
| `DB_PASS` | Strong random password |
| `DB_ADMIN_PASS` | Strong random password (different) |
| `IRIS_ADMIN_PASSWORD` | Strong password — becomes the admin login |

Generate the secret key:

```bash
openssl rand -hex 32
```

> Do not set `IRIS_ADMIN_PASSWORD` to the same value as any other tool in the stack. Credential reuse across systems is the kind of thing you are building this SOC to detect.

### 3.3 Start the Stack

```bash
docker compose up -d
```

IRIS initialises its PostgreSQL database and runs schema migrations on first start. This takes 2–4 minutes. Monitor:

```bash
docker compose logs -f app
```

Wait until the logs show the Gunicorn workers starting and binding to port 8000 before accessing the web UI. The nginx container will return errors until the app is fully ready.

---

## 4. First Login

Navigate to `https://10.10.10.40` from your management machine. Accept the self-signed certificate warning.

Retrieve the generated administrator password from the container logs if you did not set `IRIS_ADMIN_PASSWORD` in `.env`:

```bash
docker compose logs app | grep "administrator password"
```

| Field | Value |
| ----- | ----- |
| Username | `administrator` |
| Password | Value from `.env` or from the log output above |

On first login IRIS will prompt you to update the administrator password. Do this immediately.

---

## 5. Create a Customer

IRIS organises cases by customer — the organisation whose environment is being monitored. Even in a homelab context, a customer record is required before cases can be created.

Navigate to **Advanced > Customers > Add Customer**.

| Field | Value |
| ----- | ----- |
| Name | `The Bridge Between Lab` |
| Description | Internal SOC homelab environment |
| SLA | Leave blank |

Save the customer. Note the **Customer ID** — n8n uses it when creating cases via the API.

To find the customer ID after creation:

```bash
curl -k -H "Authorization: Bearer <api-key>" \
     https://10.10.10.40/manage/customers/list | python3 -m json.tool
```

---

## 6. Review Case Templates

IRIS ships with default case templates covering common incident types. Navigate to **Advanced > Case Templates** to review what is available.

For this SOC, the following templates are relevant:

| Template | Use Case |
| -------- | -------- |
| Default | Generic incidents |
| Phishing | Email-based attacks |
| Ransomware | File encryption events |

When n8n creates a case, it specifies a template ID. The default template (ID: 1) is used in the base workflows. You can create custom templates with pre-populated task checklists matching your incident response procedures.

---

## 7. Generate API Key for n8n

Navigate to **My Profile > API key** (top-right user menu).

Click **Generate** (or **Show** if one already exists from a previous session). Copy the key.

The IRIS API key is tied to the administrator account and inherits its permissions — full read/write access to all cases, alerts, and assets.

**Add this key to n8n now:**

In the n8n web UI, navigate to **Settings > Credentials** and complete the `DFIR IRIS` credential created in `03-n8n.md`:

| Field | Value |
| ----- | ----- |
| Name | `Authorization` |
| Value | `Bearer <your-iris-api-key>` |

---

## 8. IRIS Case Structure

Understanding how n8n creates IRIS cases helps when debugging workflow failures. A case created via the API has the following structure:

```json
{
  "case_name": "BRF-001 - RDP Brute Force from 203.0.113.42",
  "case_description": "Wazuh rule 60106 | Severity 10 | Agent: victim-01",
  "case_customer": 1,
  "case_template_id": 1,
  "case_soc_id": "WZ-60106-1700000000"
}
```

After creation, n8n makes additional API calls to:

* Add IOCs (the source IP, file hash, or domain)
* Add assets (the affected endpoint from `agent.name`)
* Add a timeline note with the full enrichment output (VirusTotal score, AbuseIPDB confidence)
* Set the initial task status to **In Progress**

These are separate API endpoints — the case creation call only creates the shell. The full workflow is in `06-integration.md`.

---

## 9. IRIS API Reference

The key endpoints used by n8n in this project:

| Action | Method | Endpoint |
| ------ | ------ | -------- |
| Create case | POST | `/api/v1/cases/add` |
| List cases | GET | `/api/v1/cases/list` |
| Add IOC | POST | `/api/v1/case/ioc/add` |
| Add asset | POST | `/api/v1/case/asset/add` |
| Add timeline event | POST | `/api/v1/case/timeline/events/add` |
| Update case state | POST | `/api/v1/cases/<id>/state/update` |

Full API documentation is accessible within the running IRIS instance at:

```
https://10.10.10.40/api/v1/ui/
```

This Swagger interface lets you test API calls directly in the browser, which is useful for debugging n8n HTTP Request node payloads.

---

## 10. Verify IRIS API

From the n8n VM, test connectivity:

```bash
curl -k -H "Authorization: Bearer <your-api-key>" \
     -H "Content-Type: application/json" \
     https://10.10.10.40/api/v1/cases/list
```

Expected: a JSON response with `"status": "success"` and an empty or populated cases array.

---

## 11. Verification

| Check | How to Verify | Expected |
| ----- | ------------- | -------- |
| Stack running | `docker compose ps` | All containers `running` |
| Web UI loads | `https://10.10.10.40` | Login page renders |
| Admin login works | Web UI login | Dashboard visible |
| Customer exists | Advanced > Customers | Lab customer listed |
| API responds | curl test from section 10 | `"status": "success"` |
| n8n credential updated | n8n > Settings > Credentials | IRIS credential saved |

---

## 12. What's Next

`06-integration.md` — import the n8n workflows, run a simulated brute force attack, and verify the full pipeline from Wazuh alert to IRIS case to OPNsense block.

---

## 13. Post-Installation Checklist

- [ ] Static IP `10.10.10.40/24` set via Netplan
- [ ] Hostname set to `iris`
- [ ] `.env` file created with strong `IRIS_SECRET_KEY` and DB passwords
- [ ] `.env` not committed to git
- [ ] IRIS stack started and all containers healthy
- [ ] Web UI accessible at `https://10.10.10.40`
- [ ] Administrator password set and logged out/back in to confirm
- [ ] Customer record created for the lab organisation
- [ ] Customer ID noted for n8n workflow configuration
- [ ] API key generated from administrator profile
- [ ] API key added to n8n DFIR IRIS credential with `Bearer` prefix
- [ ] API connectivity verified via curl test from n8n VM
- [ ] IRIS API Swagger UI accessible at `/api/v1/ui/`
- [ ] Snapshot taken of the VM in this clean state

---
