# Bridge Between

This project simulates a security architecture for SME environments. The goal is to design and document a complete security operations 
ecosystem using open-source tools that can align with ISO 27001 operational security principles. 

The environment simulates a mid-sized organization and demonstrates how security monitoring, threat intelligence, incident response, vulnerability management, and configuration compliance work together.

## Stack

- pfSense – Firewall
- Microsoft Entra ID – Identity and access management 
- Wazuh – SIEM, XDR, Vulnerability Detection
- DFIR IRIS – Incident Case Management
- N8N – SOAR Automation
- MISP – Threat Intelligence Databse
- Grafana & Prometheus – Infrastructure monitoring dashboards (planned) 

## Architecture

The lab runs inside VMware Workstation Pro using an internal network within a VLAN.

| System | IP | Purpose |
|------|------|------|
| pfSense | 10.10.1.254 | Firewall |
| Kali | 10.10.1.50 | Security operations |
| Wazuh | 10.10.1.51 | SIEM |
| DFIR IRIS | 10.10.1.52 | Case Management |
| N8N | 10.10.1.53 | SOAR |
| MISP | 10.10.1.54 | Threa Intelligence |

## Features

- Automated SIEM alert enrichment
- Incident response playbooks
- ISO27001 control mapping
- Threat intelligence integration

## Future Implementation

- Infrastructure monitoring with metrics and dashboards with Grafana & Prometheus
- Compliance CIS Benchmarks with Wazuh Security Configuration Assessment (SCA) to evaluate endpoint configurations against CIS Benchmark standards
- Security dashboard metrics such as mean-time-to-detect (MTTD) and mean-time-to-respond (MTTR)
- Endpoint network isolation automatition
- Automated IOC ingestion into MISP
- Wazuh active response & MISP integration to automatically 
- Improved alert correlation across multiple sources