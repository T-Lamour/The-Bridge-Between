# OPNsense — Firewall and IDS Setup

## Overview

This guide installs and configures OPNsense 25.x as the gateway, VLAN router, and intrusion detection system for the SOC lab. By the end of this guide you will have:

* Three isolated VLANs (Management, SOC, Victim) with OPNsense as the gateway
* Firewall rules matching the policy defined in `00-prerequisites.md`
* Suricata running in IDS mode on the WAN interface with the ET Open ruleset
* Syslog forwarding to the Wazuh VM (deployed in the next guide)

OPNsense runs as a bare OS install — not in Docker. It needs direct access to the virtual NICs to handle VLAN tagging and packet inspection.

---

## 1. Before You Begin

* The OPNsense VM must exist with the resource allocation from `00-prerequisites.md` (2 vCPU, 4 GB RAM, 30 GB disk)
* The VM must have **two vNICs** attached before first boot
* The hypervisor virtual switch connected to vNIC 2 must support 802.1Q VLAN tagging (trunk port)
* You will not need SSH for this guide — all configuration is done via the console and web UI

---

## 2. VM NIC Layout

Assign the two vNICs as follows before powering on the VM for the first time.

| vNIC | Hypervisor Port Group | OPNsense Role | Purpose |
| ---- | --------------------- | ------------- | ------- |
| vNIC 1 | WAN (NAT or bridged to physical NIC) | WAN | Uplink to internet |
| vNIC 2 | LAN trunk (VLAN-aware, all VLANs tagged) | LAN | Carries VLAN 10, 20, 30 |

**VMware:** Set vNIC 2's port group to trunk mode or use a vSwitch with no VLAN filter (VLAN ID 4095 in ESXi means pass all).

**Proxmox:** Attach vNIC 2 to a Linux bridge with VLAN-aware checked. No VLAN tag on the bridge itself — OPNsense will handle the tagging.

---

## 3. Installation

### Download

Get the amd64 DVD image from the OPNsense website. Verify the checksum before attaching it as a virtual CD-ROM.

```
https://opnsense.org/download/
Image type: dvd
Architecture: amd64
```

### Installing

1. Boot the VM from the ISO
2. At the login prompt, log in as `installer` with password `opnsense`
3. Select **Install (ZFS)** — ZFS gives you checksumming and easier snapshot recovery
4. Select **stripe** (single disk, no redundancy needed for a lab VM)
5. Select the 30 GB virtual disk
6. Confirm and wait for the install to complete (~3 minutes)
7. At the post-install prompt, select **Reboot** and detach the ISO before it boots again

Default root password after install: `opnsense` — change this in step 5.

---

## 4. First Boot — Console Setup

OPNsense boots to a console menu. Before the web UI is accessible, two things must be configured at the console.

### 4.1 Interface Assignment

Select option **1) Assign Interfaces**.

* Do you want to configure LAGGs? → **n**
* Do you want to configure VLANs? → **n** (VLANs are created in the web UI)
* WAN interface → select the NIC connected to your internet uplink (typically `em0` or `vtnet0`)
* LAN interface → select the NIC connected to the trunk port (typically `em1` or `vtnet1`)
* Optional interfaces → leave blank, press Enter

Confirm the assignment.

### 4.2 LAN IP Assignment

Select option **2) Set Interface IP Address** and choose **LAN**.

The LAN interface is temporary — it exists only to reach the web UI for the first time. You will replace it with VLAN sub-interfaces in section 6.

* Configure IPv4 via DHCP? → **n**
* IPv4 address → `10.10.1.254`
* Subnet mask (CIDR) → `24`
* Configure IPv6? → **n**
* Enable DHCP server on LAN? → **n** (configured per-VLAN later)
* Revert to HTTP? → **n** (keep HTTPS)

The web UI is now accessible at `https://10.10.1.254` from a machine on the same untagged segment. If your management machine is not yet on VLAN 10, temporarily connect it directly or use an untagged port.

---

## 5. Web UI — Initial Wizard

Navigate to `https://10.10.1.254`. Accept the self-signed certificate warning.

Credentials: `root` / `opnsense`

The setup wizard launches automatically. Work through it:

| Field | Value |
| ----- | ----- |
| Hostname | `opnsense` |
| Domain | `soc.local` |
| Primary DNS | `1.1.1.1` |
| Secondary DNS | `8.8.8.8` |
| Time zone | Your local timezone |
| WAN type | DHCP (or PPPoE if your ISP requires it) |
| LAN IP | Leave as `10.10.1.254` for now |
| Root password | Set a strong password now |

Finish the wizard. OPNsense will apply settings and reload. Log back in with your new password.

---

## 6. VLAN Configuration

### 6.1 Create VLANs

Navigate to **Interfaces > Other Types > VLAN**.

Create three VLANs on the LAN parent interface (`em1` or `vtnet1`):

| VLAN Tag | Description | Parent Interface |
| -------- | ----------- | ---------------- |
| 10 | Management | LAN (em1 / vtnet1) |
| 20 | SOC | LAN (em1 / vtnet1) |
| 30 | Victim | LAN (em1 / vtnet1) |

Save after each entry.

### 6.2 Assign VLAN Interfaces

Navigate to **Interfaces > Assignments**.

Add each VLAN as a new interface using the dropdown at the bottom of the assignments list. OPNsense will name them OPT1, OPT2, OPT3 by default — rename them immediately.

| Auto Name | Rename To | VLAN |
| --------- | --------- | ---- |
| OPT1 | MGMT | VLAN 10 |
| OPT2 | SOC | VLAN 20 |
| OPT3 | VICTIM | VLAN 30 |

Save and apply.

### 6.3 Configure Interface IPs

Navigate to each interface under **Interfaces > [name]** and apply the following settings. Enable the interface, set it to static IPv4, and save.

| Interface | IPv4 Address | Subnet | Enable DHCP Server |
| --------- | ------------ | ------ | ------------------ |
| MGMT | 10.10.1.254 | /24 | Yes — range .100–.200 |
| SOC | 10.10.10.254 | /24 | No — all SOC VMs use static IPs |
| VICTIM | 10.10.20.254 | /24 | Yes — range .100–.200 |

After saving each interface, click **Apply Changes**.

> The original LAN interface (`em1` without a VLAN tag) is no longer needed. You can leave it assigned but unconfigured, or remove it from assignments. Do not delete the physical interface — only the assignment.

---

## 7. DHCP Server

Navigate to **Services > DHCPv4**.

Enable the DHCP server on MGMT and VICTIM only. SOC VMs will have addresses configured via Netplan (covered in each tool's deployment guide).

| Interface | Range Start | Range End | DNS |
| --------- | ----------- | --------- | --- |
| MGMT | 10.10.1.100 | 10.10.1.200 | 10.10.1.254 |
| VICTIM | 10.10.20.100 | 10.10.20.200 | 10.10.20.254 |

Save and apply each.

---

## 8. Firewall Rules

OPNsense blocks all traffic between interfaces by default unless a rule permits it. The WAN interface has a default block-all inbound rule. You only need to add the rules below.

Navigate to **Firewall > Rules** and select each interface tab.

### 8.1 VICTIM → SOC (Wazuh agent traffic)

Create two rules on the **VICTIM** interface:

| # | Protocol | Source | Destination | Port | Description |
| - | -------- | ------ | ----------- | ---- | ----------- |
| 1 | TCP | VICTIM net | 10.10.10.10 | 1515 | Wazuh agent registration |
| 2 | UDP | VICTIM net | 10.10.10.10 | 1514 | Wazuh agent log shipping |

### 8.2 MGMT → SOC (dashboard access)

Create one rule on the **MGMT** interface:

| # | Protocol | Source | Destination | Port | Description |
| - | -------- | ------ | ----------- | ---- | ----------- |
| 1 | TCP | MGMT net | SOC net | 443 | HTTPS access to SOC dashboards |

### 8.3 SOC → WAN (outbound API calls)

Create one rule on the **SOC** interface:

| # | Protocol | Source | Destination | Port | Description |
| - | -------- | ------ | ----------- | ---- | ----------- |
| 1 | TCP | SOC net | any | 443 | Outbound HTTPS for enrichment APIs |

### 8.4 Default deny

No action required. OPNsense blocks all other inter-VLAN traffic by default. Do not add a catch-all allow rule. Verify there is no pre-existing LAN allow-all rule that was carried over from the initial wizard — remove it if present.

### 8.5 DNS resolution for VMs

SOC and VICTIM VMs need to resolve external hostnames for package installs and API calls. Add the following on both the SOC and VICTIM interface tabs:

| # | Protocol | Source | Destination | Port | Description |
| - | -------- | ------ | ----------- | ---- | ----------- |
| 1 | UDP | [interface] net | [interface gateway] | 53 | DNS to OPNsense |

Apply all rules after saving.

---

## 9. Suricata IDS

### 9.1 Install the Plugin

Navigate to **System > Firmware > Plugins**.

Find `os-suricata` in the list and click the install icon. OPNsense will install the plugin and add it under the Services menu. No reboot required.

### 9.2 Enable and Configure

Navigate to **Services > Intrusion Detection > Administration**.

| Setting | Value |
| ------- | ----- |
| Enabled | Checked |
| IPS Mode | **Unchecked** — run as IDS (alert only) until rules are tuned |
| Promiscuous mode | Checked |
| Interfaces | WAN |
| Pattern matcher | Aho-Corasick (default) |
| Log level | Info |
| Log retention | 7 days |

Click **Apply**.

> Start in IDS mode. Switching to IPS mode (inline blocking) before tuning the rulesets will generate false positives that drop legitimate traffic. Enable IPS only after reviewing alerts over several days.

### 9.3 Rulesets

Navigate to the **Download** tab.

Enable the **ET Open** ruleset — it is free, requires no registration, and covers the most common threats. Click **Download and Update Rules** after enabling it.

| Ruleset | Cost | Recommended |
| ------- | ---- | ----------- |
| ET Open (Emerging Threats) | Free | Yes |
| ET Pro | Paid | No — not needed for this lab |
| Abuse.ch | Free | Optional — adds botnet/malware C2 rules |

### 9.4 Schedule Rule Updates

Navigate to the **Schedule** tab. Enable automatic rule updates and set the interval to daily. This keeps signatures current without manual intervention.

---

## 10. Syslog Forwarding to Wazuh

OPNsense forwards logs to Wazuh via syslog. Configure this now — the Wazuh deployment guide will handle the receiving side.

Navigate to **System > Settings > Logging > Remote**.

| Field | Value |
| ----- | ----- |
| Enable | Checked |
| Transport | UDP |
| Remote syslog host | `10.10.10.10` (Wazuh VM) |
| Remote syslog port | `514` |
| Log level | Informational |

Enable the following log sources:

* Firewall events
* Intrusion Detection (Suricata alerts)
* System events

Save and apply.

> The Wazuh syslog receiver is not active until the Wazuh VM is deployed. These settings will queue silently until then — nothing to action now.

---

## 11. Verification

Run these checks before moving to the next guide.

### 11.1 VLAN Connectivity

From a machine on VLAN 10 (Management), verify gateway reachability:

```
ping 10.10.1.254     # MGMT gateway — must reply
ping 10.10.10.254    # SOC gateway — must reply
ping 10.10.20.254    # VICTIM gateway — must reply
```

### 11.2 Inter-VLAN Block

From a machine on VLAN 30 (Victim), attempt to reach the SOC subnet on any port other than 1514/1515. It must time out.

```
ping 10.10.10.10     # must time out — no ICMP rule permits this
```

### 11.3 Outbound Internet

From OPNsense itself (**Interfaces > Diagnostics > Ping**):

* Source: any
* Hostname: `1.1.1.1`
* Must receive replies

### 11.4 Suricata Running

Navigate to **Services > Intrusion Detection**. The status indicator must show **Running**. If it shows stopped, check that the WAN interface has traffic — Suricata will not start if it cannot bind to the selected interface.

### 11.5 Firewall Logs

Navigate to **Firewall > Log Files > Live View**. Generate some cross-VLAN traffic and confirm the correct allow/block entries appear.

---

## 12. What's Next

`02-wazuh.md` — deploy the Wazuh SIEM on the Ubuntu VM at `10.10.10.10`, configure it to receive syslog from OPNsense, and deploy the first Wazuh agent to a Victim VLAN endpoint.

---

## 13. Post-Installation Checklist

- [ ] OPNsense installed and booting from disk (ISO detached)
- [ ] Root password changed from default
- [ ] WAN interface has internet connectivity
- [ ] VLAN 10, 20, 30 created and assigned as MGMT, SOC, VICTIM
- [ ] MGMT interface IP is 10.10.1.254/24
- [ ] SOC interface IP is 10.10.10.254/24
- [ ] VICTIM interface IP is 10.10.20.254/24
- [ ] DHCP enabled on MGMT and VICTIM, disabled on SOC
- [ ] Firewall rule: VICTIM → Wazuh 1514/1515 (TCP/UDP)
- [ ] Firewall rule: MGMT → SOC 443 (TCP)
- [ ] Firewall rule: SOC → WAN 443 (TCP)
- [ ] No catch-all allow rule between VLANs
- [ ] Suricata plugin installed and running on WAN
- [ ] ET Open ruleset downloaded and active
- [ ] Suricata running in IDS mode (not IPS)
- [ ] Rule update schedule set to daily
- [ ] Syslog forwarding to 10.10.10.10:514 configured
- [ ] Ping to all three VLAN gateways succeeds from MGMT
- [ ] Ping from VICTIM to SOC subnet times out
- [ ] Snapshot taken of the VM in this clean state

---
