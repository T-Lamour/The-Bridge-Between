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
- MISP – Threat Intelligence Database
- pfSense - Firewall, netowkr control and segmentation


## Features

- Detection & Visibility
- Automation
- Threat Intelligence Enrichment
- Incident Response

## Future Implementation

Security is always evolving, so there are always improvements to this project:

- Infrastructure monitoring with Grafana & Prometheus
- Compliance CIS Benchmarks with Wazuh Security Configuration Assessment (SCA) to evaluate endpoint configurations against CIS Benchmark standards
- Security dashboard metrics such as mean-time-to-detect and mean-time-to-respond 
- Endpoint network isolation automatition
- Automated IOC ingestion into MISP
- Wazuh active response & MISP integration to automatically 
- Improved alert correlation across multiple sources