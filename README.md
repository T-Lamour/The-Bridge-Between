# Cloud Security Homelab

A SOC homelab project designed to simulate a small-business security monitoring platform.

## Stack

- pfSense – firewall
- Wazuh – SIEM
- DFIR IRIS – case management
- N8N – SOAR automation
- Kali Linux – security operations

## Architecture

The lab runs inside VMware using an internal network.

| System | IP | Purpose |
|------|------|------|
| pfSense | 10.10.1.254 | Firewall |
| Wazuh | 10.10.1.51 | SIEM |
| DFIR IRIS | 10.10.1.57 | Case Management |
| N8N | 10.10.1.59 | Automation |
| Kali | 10.10.1.50 | Security operations |

## Features

- Automated SIEM alert enrichment
- Incident response playbooks
- ISO27001 control mapping
- Threat intelligence integration