# The Bridge Between

Cybersecurity has a bit of an identity crisis. 

One side you've got enterprise companies running full-scale SOCs with tools costing more than a deposit on a house. Then on the other side, you've got small to medium sized businesses thinking "we've got antivirus running, so we should be fine...right?"

Somewhere in between those two worlds is a massive gap, and this project exists to fill it. The Bridge Between is exactly what it sounds like - a way to get from "no real security" to "actually decent security" without needing a ridiculous budget.

## How Much Money Can Be Saved?

I've once saw a quote from an MSSP that offered a "SIEM Solution", which was basically a bundle of outdated, vulnerable open-source tools that was lightly stitched together. No innovation, no real engineering, just "That will be £15,000 a month please."

Meanwhile, this project runs on:

£0 in licensing. Zero.

Everything here runs entirely on free tooling, and it's not the "free tier but actually useless" free tooling, but an actual security stack. The only real cost is theinfrastructure and your time.

## The Stack

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

- Infrastructure monitoring with metrics and dashboards with Grafana & Prometheus
- Compliance CIS Benchmarks with Wazuh Security Configuration Assessment (SCA) to evaluate endpoint configurations against CIS Benchmark standards
- Security dashboard metrics such as mean-time-to-detect (MTTD) and mean-time-to-respond (MTTR)
- Endpoint network isolation automatition
- Automated IOC ingestion into MISP
- Wazuh active response & MISP integration to automatically 
- Improved alert correlation across multiple sources