# 07 — Ansible — Provisioning, Patching, and Hardening

## Overview

This guide covers deploying Ansible and running the three core playbook categories used in this lab: provisioning (Docker and baseline packages), patching (OS updates and Docker image refreshes), and hardening (SSH, ufw, and service-level controls). By the end of this guide:

* Ansible is installed and reachable on its LXC
* Inventory is configured for all SOC VMs and containers
* SSH key access from the Ansible controller is confirmed to all targets
* All three playbook categories have been run and verified

Ansible runs agentlessly over SSH — nothing is installed on the managed hosts beyond Python 3, which Ubuntu 22.04 provides by default.

---

## 1. Before You Begin

This guide assumes `06-integration.md` is complete and all five SOC components are deployed and operational. Ansible manages those already-running hosts — it does not deploy them from scratch.

| Requirement | Where to Check |
| ----------- | -------------- |
| Ansible LXC running on Proxmox 2 | Proxmox 2 UI — container status |
| SSH reachable from Ansible LXC to all target VMs | `ssh ansible@10.10.10.10` etc. from the Ansible LXC |
| Python 3 present on all targets | `python3 --version` on each target |
| Ansible service account with passwordless sudo on all targets | `sudo -l` as the service account |

---

## 2. Install Ansible

SSH into the Ansible LXC (`10.10.10.60`).

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Verify
ansible --version
```

Confirm the version is 2.14 or later. Earlier versions lack collection support for some modules used in the hardening playbook.

---

## 3. Configure the Ansible Service Account on Each Target

Run the following on **every VM and LXC that Ansible will manage** — Wazuh, MISP, n8n, DFIR IRIS, and Grafana/Prometheus. Substitute `ansible` for the username if you prefer a different name.

```bash
# Create the service account
sudo useradd -m -s /bin/bash ansible
sudo usermod -aG sudo ansible

# Grant passwordless sudo
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
```

Then, from the **Ansible LXC**, generate an SSH key and distribute it:

```bash
# Generate a key if one does not already exist
ssh-keygen -t ed25519 -C "ansible@soc-lab" -f ~/.ssh/id_ed25519 -N ""

# Copy to each target (repeat for each IP)
ssh-copy-id ansible@10.10.10.10   # Wazuh
ssh-copy-id ansible@10.10.10.20   # MISP
ssh-copy-id ansible@10.10.10.30   # n8n
ssh-copy-id ansible@10.10.10.40   # DFIR IRIS
ssh-copy-id ansible@10.10.10.50   # Grafana + Prometheus
```

Verify each host is reachable before writing any playbooks:

```bash
for ip in 10.10.10.10 10.10.10.20 10.10.10.30 10.10.10.40 10.10.10.50; do
  ssh -o BatchMode=yes ansible@$ip "echo $ip OK" 2>&1
done
```

All five should return `<ip> OK`. Any failure must be resolved before continuing.

---

## 4. Inventory

Create the working directory structure on the Ansible LXC:

```bash
mkdir -p ~/soc-ansible/{inventory,playbooks,roles}
```

Create `~/soc-ansible/inventory/hosts.ini`:

```ini
[soc]
wazuh      ansible_host=10.10.10.10
misp       ansible_host=10.10.10.20
n8n        ansible_host=10.10.10.30
iris       ansible_host=10.10.10.40
monitoring ansible_host=10.10.10.50

[soc:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3

[vms]
wazuh
misp

[containers]
n8n
iris
monitoring
```

Confirm Ansible can reach all hosts via the inventory:

```bash
cd ~/soc-ansible
ansible -i inventory/hosts.ini soc -m ping
```

All five hosts must return `pong` before continuing.

---

## 5. Playbook — Provisioning

The provisioning playbook applies the Docker and OS baseline to a fresh host. If a host was deployed following the earlier deployment guides, this playbook is idempotent — it will confirm the desired state is already in place without making changes.

Create `~/soc-ansible/playbooks/provision.yml`:

```yaml
---
- name: SOC host provisioning baseline
  hosts: soc
  become: true

  vars:
    docker_packages:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin

  tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Install baseline packages
      apt:
        name:
          - curl
          - ca-certificates
          - gnupg
          - ufw
          - fail2ban
          - unattended-upgrades
          - python3-pip
          - jq
        state: present

    - name: Add Docker GPG key
      shell: |
        install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
          -o /etc/apt/keyrings/docker.asc
        chmod a+r /etc/apt/keyrings/docker.asc
      args:
        creates: /etc/apt/keyrings/docker.asc

    - name: Add Docker apt repository
      shell: |
        echo \
          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
          https://download.docker.com/linux/ubuntu \
          $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
          | tee /etc/apt/sources.list.d/docker.list > /dev/null
      args:
        creates: /etc/apt/sources.list.d/docker.list

    - name: Install Docker Engine
      apt:
        name: "{{ docker_packages }}"
        update_cache: true
        state: present

    - name: Ensure Docker service is enabled and running
      systemd:
        name: docker
        enabled: true
        state: started

    - name: Verify Docker Compose v2 is available
      command: docker compose version
      register: compose_version
      changed_when: false

    - name: Print Docker Compose version
      debug:
        msg: "{{ compose_version.stdout }}"

    - name: Configure NTP sync check
      command: timedatectl show --property=NTPSynchronized --value
      register: ntp_status
      changed_when: false

    - name: Warn if NTP not synchronised
      debug:
        msg: "WARNING: NTP not synchronised on {{ inventory_hostname }}"
      when: ntp_status.stdout != "yes"
```

Run it:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/provision.yml
```

Expected output: all tasks `ok` or `changed`. No `failed` or `unreachable`.

---

## 6. Playbook — Patching

The patching playbook applies OS updates, removes unused packages, and refreshes Docker images for each service. Run this on a regular cadence — monthly is reasonable for a lab; weekly if the environment handles real traffic.

Create `~/soc-ansible/playbooks/patch.yml`:

```yaml
---
- name: SOC host patching
  hosts: soc
  become: true
  serial: 1

  tasks:
    - name: Update apt cache
      apt:
        update_cache: true

    - name: Upgrade all packages to latest
      apt:
        upgrade: dist
        autoremove: true
        autoclean: true

    - name: Check if reboot is required
      stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Notify if reboot needed
      debug:
        msg: "{{ inventory_hostname }} requires a reboot — schedule a maintenance window"
      when: reboot_required.stat.exists

    - name: Pull latest Docker images for all running stacks
      shell: |
        for dir in $(find /opt -maxdepth 2 -name "docker-compose.yml" -exec dirname {} \;); do
          echo "Pulling in $dir"
          cd "$dir" && docker compose pull
        done
      register: pull_output
      changed_when: "'Pulling' in pull_output.stdout"

    - name: Restart services after image pull
      shell: |
        for dir in $(find /opt -maxdepth 2 -name "docker-compose.yml" -exec dirname {} \;); do
          cd "$dir" && docker compose up -d
        done
      changed_when: true
```

> **`serial: 1`** ensures hosts are patched one at a time. Patching all hosts simultaneously risks taking the entire SOC offline if a container fails to restart after a bad image pull.

Run it:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/patch.yml
```

After patching, verify each tool's web UI is reachable before moving to the next host. Any host requiring a reboot must be rebooted during a maintenance window — the playbook will flag it but will not reboot automatically.

---

## 7. Playbook — Hardening

The hardening playbook applies the SSH, ufw, and fail2ban configuration that should be in place on every host. Run this after provisioning and after any major configuration change.

Create `~/soc-ansible/playbooks/harden.yml`:

```yaml
---
- name: SOC host hardening
  hosts: soc
  become: true

  tasks:
    # SSH hardening
    - name: Disable SSH password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify: Restart sshd

    - name: Disable SSH root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        state: present
      notify: Restart sshd

    - name: Set SSH idle timeout (10 minutes)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?ClientAliveInterval'
        line: 'ClientAliveInterval 600'
        state: present
      notify: Restart sshd

    - name: Set SSH max auth tries
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?MaxAuthTries'
        line: 'MaxAuthTries 3'
        state: present
      notify: Restart sshd

    # ufw — default deny, then open per-host ports
    - name: Set ufw default incoming policy to deny
      ufw:
        direction: incoming
        policy: deny

    - name: Set ufw default outgoing policy to allow
      ufw:
        direction: outgoing
        policy: allow

    - name: Allow SSH from management VLAN only
      ufw:
        rule: allow
        src: 10.10.1.0/24
        port: '22'
        proto: tcp

    - name: Enable ufw
      ufw:
        state: enabled

    # fail2ban
    - name: Ensure fail2ban is enabled and running
      systemd:
        name: fail2ban
        enabled: true
        state: started

    - name: Configure fail2ban SSH jail
      copy:
        dest: /etc/fail2ban/jail.d/sshd.local
        content: |
          [sshd]
          enabled  = true
          port     = ssh
          maxretry = 5
          bantime  = 3600
          findtime = 600
        mode: '0644'
      notify: Restart fail2ban

    # Unattended upgrades for security patches
    - name: Enable unattended security upgrades
      copy:
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";
        mode: '0644'

  handlers:
    - name: Restart sshd
      systemd:
        name: ssh
        state: restarted

    - name: Restart fail2ban
      systemd:
        name: fail2ban
        state: restarted
```

Run it:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/harden.yml
```

> The ufw rules in this playbook apply the global baseline (SSH from management VLAN only). Service-specific ports — Wazuh agent ports, Grafana HTTPS, etc. — are opened in `09-hardening.md` per host. Do not run the hardening playbook if you have not yet configured service-specific ufw rules on each host, or you will lock yourself out of dashboards.

---

## 8. Proxmox Snapshot Strategy

Automated Proxmox snapshots provide the rollback path for the entire lab. Configure a cron job on each Proxmox host to snapshot all VMs and containers nightly.

SSH into **Proxmox 1** and **Proxmox 2** (not a managed LXC — this runs on the hypervisor itself):

```bash
# On each Proxmox host — list VM/container IDs to confirm
qm list    # VMs
pct list   # LXC containers
```

Create `/etc/cron.d/soc-snapshots` on **Proxmox 1**:

```bash
# Nightly snapshots at 02:00 — Wazuh (100) and MISP (101)
# Adjust VM IDs to match your Proxmox instance
0 2 * * * root qm snapshot 100 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot" && \
               qm snapshot 101 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot"
```

Create `/etc/cron.d/soc-snapshots` on **Proxmox 2**:

```bash
# Nightly snapshots at 02:15 — n8n (200), IRIS (201), monitoring (202), ansible (203)
15 2 * * * root pct snapshot 200 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot" && \
                pct snapshot 201 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot" && \
                pct snapshot 202 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot" && \
                pct snapshot 203 nightly-$(date +\%Y\%m\%d) --description "Automated nightly snapshot"
```

Verify the cron entries are active after creating the files:

```bash
crontab -l
systemctl status cron
```

**Snapshot retention:** Proxmox does not auto-delete old snapshots. Add a cleanup step to the cron job or purge manually — keeping 7 days is reasonable:

```bash
# Add to the cron job, after the snapshot commands (example for VM 100)
qm listsnapshot 100 | awk '/nightly-/{print $2}' | sort | head -n -7 | \
  xargs -I {} qm delsnapshot 100 {}
```

Adjust VM/container IDs to match your Proxmox instance. The IDs shown here (100–203) are examples.

---

## 9. Verification

```bash
# Confirm Ansible reaches all hosts
ansible -i inventory/hosts.ini soc -m ping

# Check SSH hardening applied
ansible -i inventory/hosts.ini soc -m shell -a "sshd -T | grep -E 'passwordauthentication|permitrootlogin|maxauthtries'"

# Check ufw status on all hosts
ansible -i inventory/hosts.ini soc -m shell -a "ufw status verbose"

# Check fail2ban is running
ansible -i inventory/hosts.ini soc -m shell -a "systemctl is-active fail2ban"

# Check Docker is running
ansible -i inventory/hosts.ini soc -m shell -a "docker compose version"
```

---

## 10. Final Checklist

- [ ] Ansible LXC running at `10.10.10.60`
- [ ] `ansible` service account with passwordless sudo on all five target hosts
- [ ] SSH key distributed — `ansible -i inventory/hosts.ini soc -m ping` returns `pong` for all five
- [ ] `hosts.ini` inventory created with correct IPs and groups
- [ ] Provisioning playbook run — no `failed` tasks
- [ ] Patching playbook run — all packages updated, Docker images refreshed
- [ ] Hardening playbook run — SSH password auth disabled, ufw default-deny active, fail2ban running
- [ ] No SSH password auth confirmed on all hosts
- [ ] ufw status shows default deny incoming on all hosts
- [ ] fail2ban active on all hosts
- [ ] Proxmox snapshot cron configured on both Proxmox hosts
- [ ] Manual snapshot taken of all VMs and containers after hardening

---
