---
draft: true
title: '37 - Zabbix'
description: "Part 37 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="VS Code" color="blue" >}}
{{< badge content="Linux" color="red" >}}

## Part 37: Traditional Monitoring — Zabbix + SNMP

> *Prometheus is the modern choice and Part 38 covers it in full. But walk into most enterprise network operations centres and you'll find Zabbix — or something that works exactly like it. Zabbix's polling model, SNMP template system, and trigger-based alerting are the conceptual foundation that SolarWinds, Cacti, LibreNMS, and dozens of other NMS products are built on. Understanding Zabbix means understanding traditional network monitoring. This part installs Zabbix on the monitoring VM, configures SNMP on every lab device via Ansible, and registers all hosts automatically through the Zabbix API — no clicking through the UI to add devices one at a time.*

---

## 37.1 — How Zabbix Works (Conceptual Foundation)

Before deploying anything, understanding the polling model is essential — it's fundamentally different from Prometheus and explains every design decision in Zabbix.

### Polling vs scraping

```
Zabbix model (PUSH and PULL — traditional NMS):

  Zabbix Server ──SNMP GET──► Network device
                              Returns: ifOperStatus, sysDescr, bgpPeerState
  Zabbix Server ──ICMP PING──► Device
  Zabbix Agent  ──────────────► Zabbix Server (agent pushes data)
  Zabbix Server stores in MySQL/PostgreSQL, evaluates triggers,
  sends alerts when thresholds are breached.

  Key characteristic: Zabbix decides when to collect data (polling interval).
  The device is passive — it just answers queries.

Prometheus model (PULL — modern observability):

  Prometheus ──HTTP GET /metrics──► Exporter on device/VM
                                    Returns: all metrics at once
  Prometheus stores in time-series DB, evaluates rules,
  sends alerts via AlertManager.

  Key characteristic: Prometheus decides when to scrape.
  The exporter is passive — it just exposes a metrics endpoint.

Why this matters:
  Zabbix SNMP polling works on any network device out of the box —
  no agent, no exporter, no software install on the device.
  A 10-year-old Cisco router responds to SNMP the same way a
  brand-new one does. This is why SNMP/Zabbix dominates enterprise
  networks — the devices already speak the protocol.

  Prometheus requires an exporter running somewhere — either on
  the device (impossible on most network OS) or a separate SNMP
  exporter that translates SNMP → Prometheus metrics format.
  Part 38 covers exactly that translation layer.
```

### Zabbix key concepts

```
Item       — A single metric collected from a host
             e.g. "ifOperStatus on GigabitEthernet1"

Trigger    — A condition evaluated against item values
             e.g. "ifOperStatus went from 1 (up) to 2 (down)"

Action     — What happens when a trigger fires
             e.g. "Send email to netops@company.com"
                  "Run a remote command"
                  "Send to Slack webhook"

Template   — A reusable set of items + triggers + graphs
             applied to a host or group of hosts
             e.g. "Cisco IOS by SNMP" template contains
             all standard Cisco SNMP items and triggers

Host       — A monitored device (maps to our network devices)

Host group — A collection of hosts (maps to Ansible groups)

Discovery  — Automatic detection of interfaces, VLANs, BGP peers
             on a device — Zabbix queries SNMP and creates items
             for each discovered entity automatically
```

---

## 37.2 — Architecture

```
monitoring VM (172.16.0.40 / 192.168.100.40)
│
├── Docker: zabbix-server       — Core Zabbix server process
│   ├── Polls devices via SNMP (UDP 161)
│   ├── Evaluates triggers
│   └── Sends alerts
│
├── Docker: zabbix-web          — Nginx + PHP web UI (port 80)
│
├── Docker: zabbix-agent        — Monitors the monitoring VM itself
│
├── Docker: mysql               — Zabbix database (MySQL 8)
│
└── Docker: snmptrapd           — Receives SNMP traps (UDP 162)
    └── Forwards traps to Zabbix server

SNMP polling flows:
  zabbix-server ──UDP 161──► wan-r1/r2 (172.16.0.11/12)
  zabbix-server ──UDP 161──► dist-01   (172.16.0.21)
  zabbix-server ──UDP 161──► spine-01  (172.16.0.31)
  zabbix-server ──UDP 161──► leaf-01/2 (172.16.0.33/34)
  zabbix-server ──UDP 161──► panos-fw01 (172.16.0.51)
  zabbix-server ──TCP 10050──► VMs via Zabbix agent (192.168.100.x)

Access:
  Web UI: http://172.16.0.40    (default credentials: Admin/zabbix)
  API:    http://172.16.0.40/api_jsonrpc.php
```

---

## 37.3 — Deploy Zabbix with Docker Compose

```bash
# SSH to monitoring VM
ssh ansible@172.16.0.40

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker ansible
newgrp docker

# Create directories
sudo mkdir -p /opt/zabbix/{mysql,alertscripts,externalscripts,snmptraps}
sudo chown -R ansible:ansible /opt/zabbix

# Generate MySQL password
MYSQL_ROOT_PASSWORD=$(openssl rand -base64 32)
MYSQL_ZABBIX_PASSWORD=$(openssl rand -base64 32)
ZABBIX_WEB_PASSWORD="ZabbixAdmin#2025"   # Change after first login

cat > ~/zabbix/.env << EOF
MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
MYSQL_ZABBIX_PASSWORD=${MYSQL_ZABBIX_PASSWORD}
ZABBIX_WEB_PASSWORD=${ZABBIX_WEB_PASSWORD}
EOF
chmod 600 ~/zabbix/.env

echo "Add to Ansible vault:"
echo "  vault_mysql_root_password: ${MYSQL_ROOT_PASSWORD}"
echo "  vault_mysql_zabbix_password: ${MYSQL_ZABBIX_PASSWORD}"
echo "  vault_zabbix_web_password: ${ZABBIX_WEB_PASSWORD}"
echo "  vault_zabbix_api_user: Admin"
```

### Docker Compose file

```bash
mkdir -p ~/zabbix
cat > ~/zabbix/docker-compose.yml << 'EOF'
---
version: "3.8"

services:

  mysql:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: "${MYSQL_ZABBIX_PASSWORD}"
    command:
      - mysqld
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_bin
      - --default-authentication-plugin=mysql_native_password
    volumes:
      - /opt/zabbix/mysql:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost",
             "-u", "zabbix", "-p${MYSQL_ZABBIX_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 10

  zabbix-server:
    image: zabbix/zabbix-server-mysql:ubuntu-7.0-latest
    container_name: zabbix-server
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_SERVER_HOST: mysql
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: "${MYSQL_ZABBIX_PASSWORD}"
      ZBX_STARTPOLLERS: 10
      ZBX_STARTSNMPPOLLERS: 10
      ZBX_STARTTRAPPERS: 5
      ZBX_CACHESIZE: 128M
      ZBX_HISTORYCACHESIZE: 64M
      ZBX_TRENDCACHESIZE: 32M
    volumes:
      - /opt/zabbix/alertscripts:/usr/lib/zabbix/alertscripts
      - /opt/zabbix/externalscripts:/usr/lib/zabbix/externalscripts
      - /opt/zabbix/snmptraps:/var/lib/zabbix/snmptraps
    ports:
      - "10051:10051"    # Zabbix trapper port (active agents)
    networks:
      - backend
      - frontend
    ulimits:
      nproc: 65535
      nofile:
        soft: 65535
        hard: 65535

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:ubuntu-7.0-latest
    container_name: zabbix-web
    restart: unless-stopped
    depends_on:
      - zabbix-server
      - mysql
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: "${MYSQL_ZABBIX_PASSWORD}"
      PHP_TZ: "America/Chicago"
    ports:
      - "80:8080"
    networks:
      - backend
      - frontend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 5

  zabbix-agent:
    image: zabbix/zabbix-agent2:ubuntu-7.0-latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      ZBX_HOSTNAME: monitoring
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: "10051"
    networks:
      - backend

  snmptrapd:
    image: zabbix/zabbix-snmptraps:ubuntu-7.0-latest
    container_name: zabbix-snmptraps
    restart: unless-stopped
    ports:
      - "162:1162/udp"   # SNMP trap receiver
    volumes:
      - /opt/zabbix/snmptraps:/var/lib/zabbix/snmptraps
    networks:
      - backend

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge
EOF
```

### Start Zabbix

```bash
cd ~/zabbix

# Pull images (several hundred MB — takes a few minutes)
docker compose pull

# Start — MySQL init takes 60-90 seconds on first run
docker compose up -d

# Watch until web UI is healthy
docker compose logs -f zabbix-web | grep -E "started|error|ready"

# Verify all containers are running
docker compose ps

# Test web UI
curl -s -o /dev/null -w "%{http_code}" http://172.16.0.40/
# Expected: 200

# Test API
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"apiinfo.version","id":1}' \
  http://172.16.0.40/api_jsonrpc.php
# Expected: {"jsonrpc":"2.0","result":"7.0.x","id":1}
```

---

## 37.4 — Ansible Playbook: Deploy Zabbix

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/zabbix/templates

cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/deploy.yml << 'EOF'
---
# Deploy Zabbix monitoring stack on the monitoring VM
# Usage: ansible-playbook playbooks/infrastructure/zabbix/deploy.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Zabbix | Deploy Zabbix stack"
  hosts: monitoring_hosts
  gather_facts: true
  become: true
  tags: [zabbix, deploy]

  vars:
    zabbix_dir: /home/ansible/zabbix
    zabbix_data_dir: /opt/zabbix

  tasks:

    - name: "Docker | Install Docker"
      ansible.builtin.apt:
        name: [docker.io, docker-compose-plugin]
        state: present
        update_cache: true

    - name: "Docker | Add ansible to docker group"
      ansible.builtin.user:
        name: ansible
        groups: docker
        append: true

    - name: "Dirs | Create Zabbix data directories"
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
      loop:
        - "{{ zabbix_dir }}"
        - "{{ zabbix_data_dir }}/mysql"
        - "{{ zabbix_data_dir }}/alertscripts"
        - "{{ zabbix_data_dir }}/externalscripts"
        - "{{ zabbix_data_dir }}/snmptraps"

    - name: "Compose | Deploy docker-compose.yml"
      ansible.builtin.template:
        src: zabbix-compose.yml.j2
        dest: "{{ zabbix_dir }}/docker-compose.yml"
        owner: ansible
        mode: '0644'

    - name: "Compose | Deploy environment file"
      ansible.builtin.template:
        src: zabbix.env.j2
        dest: "{{ zabbix_dir }}/.env"
        owner: ansible
        mode: '0600'

    - name: "UFW | Allow Zabbix web UI"
      community.general.ufw:
        rule: allow
        port: "80"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Zabbix web UI"

    - name: "UFW | Allow Zabbix trapper (active agents)"
      community.general.ufw:
        rule: allow
        port: "10051"
        proto: tcp
        src: "192.168.100.0/24"
        comment: "Zabbix active agent connections"

    - name: "UFW | Allow SNMP traps"
      community.general.ufw:
        rule: allow
        port: "162"
        proto: udp
        comment: "SNMP traps from network devices"

    - name: "Docker | Start Zabbix stack"
      community.docker.docker_compose_v2:
        project_src: "{{ zabbix_dir }}"
        pull: always
        state: present
      become: false

    - name: "Zabbix | Wait for web UI"
      ansible.builtin.uri:
        url: "http://172.16.0.40/"
        status_code: 200
      register: zabbix_check
      retries: 20
      delay: 15
      until: zabbix_check.status == 200
      delegate_to: localhost

    - name: "Zabbix | Change default admin password via API"
      ansible.builtin.uri:
        url: "http://172.16.0.40/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "user.login"
          params:
            username: Admin
            password: zabbix
          id: 1
        status_code: 200
      register: login_result
      delegate_to: localhost
      failed_when: login_result.json.result is not defined
      no_log: true

    - name: "Zabbix | Store API token fact"
      ansible.builtin.set_fact:
        zabbix_api_token: "{{ login_result.json.result }}"
      no_log: true

    - name: "Zabbix | Update admin password"
      ansible.builtin.uri:
        url: "http://172.16.0.40/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: "user.update"
          params:
            userid: "1"
            passwd: "{{ vault_zabbix_web_password }}"
          auth: "{{ zabbix_api_token }}"
          id: 2
        status_code: 200
      delegate_to: localhost
      no_log: true

    - name: "Zabbix | Report deployment"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════"
          - " Zabbix is running"
          - " Web UI: http://172.16.0.40"
          - " Login:  Admin / (see vault_zabbix_web_password)"
          - " API:    http://172.16.0.40/api_jsonrpc.php"
          - "════════════════════════════════════════"
EOF
```

---

## 37.5 — SNMP on Network Devices

### SNMP v2c — get monitoring working first

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/configure_snmp.yml << 'EOF'
---
# Configure SNMP on all network devices
# Covers SNMP v2c (quick start) then v3 (production hardening)
# Usage: ansible-playbook playbooks/infrastructure/zabbix/configure_snmp.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt
#        --tags snmp_v2c     (v2c only)
#        --tags snmp_v3      (v3 only)
#        --tags snmp         (both)

# ── Cisco IOS-XE (wan-r1, wan-r2, dist-01, leaf-01, leaf-02) ────────
- name: "SNMP | Configure Cisco IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: [snmp]

  vars:
    snmp_community_ro: "{{ vault_snmp_community_ro }}"   # v2c read-only community
    snmp_v3_user: "zabbix"
    snmp_v3_auth_pass: "{{ vault_snmp_v3_auth_pass }}"
    snmp_v3_priv_pass: "{{ vault_snmp_v3_priv_pass }}"
    snmp_location: "Lab / Proxmox Host"
    snmp_contact: "admin@lab.local"
    zabbix_server_ip: "172.16.0.40"

  tasks:

    # ── SNMP v2c ──────────────────────────────────────────────────────
    - name: "SNMPv2c | Configure read-only community and system info"
      cisco.ios.ios_config:
        lines:
          - "snmp-server community {{ snmp_community_ro }} RO"
          - "snmp-server location {{ snmp_location }}"
          - "snmp-server contact {{ snmp_contact }}"
          - "snmp-server host {{ zabbix_server_ip }} version 2c {{ snmp_community_ro }}"
          - "snmp-server enable traps snmp linkdown linkup"
          - "snmp-server enable traps bgp"
          - "snmp-server enable traps config"
        save_when: changed
      tags: [snmp, snmp_v2c]

    # ── SNMP v3 ───────────────────────────────────────────────────────
    # v3 provides authentication (SHA) and encryption (AES128)
    # This is the production-grade approach — replaces v2c community strings
    - name: "SNMPv3 | Create SNMP group with auth+priv"
      cisco.ios.ios_config:
        lines:
          - "snmp-server group ZABBIX-V3 v3 priv"
      tags: [snmp, snmp_v3]

    - name: "SNMPv3 | Create SNMP user"
      cisco.ios.ios_config:
        lines:
          - "snmp-server user {{ snmp_v3_user }} ZABBIX-V3 v3
             auth sha {{ snmp_v3_auth_pass }}
             priv aes 128 {{ snmp_v3_priv_pass }}"
      no_log: true
      tags: [snmp, snmp_v3]

    - name: "SNMPv3 | Configure v3 trap host"
      cisco.ios.ios_config:
        lines:
          - "snmp-server host {{ zabbix_server_ip }} version 3 priv
             {{ snmp_v3_user }}"
        save_when: changed
      tags: [snmp, snmp_v3]

    # ── Verify ────────────────────────────────────────────────────────
    - name: "Verify | Show SNMP configuration"
      cisco.ios.ios_command:
        commands:
          - show snmp community
          - show snmp user
      register: snmp_verify
      tags: [snmp, verify]

    - name: "Verify | Display SNMP config"
      ansible.builtin.debug:
        msg: "{{ snmp_verify.stdout }}"
      tags: [snmp, verify]

# ── Cisco NX-OS (spine-01) ───────────────────────────────────────────
- name: "SNMP | Configure Cisco NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tags: [snmp]

  vars:
    snmp_community_ro: "{{ vault_snmp_community_ro }}"
    snmp_v3_user: "zabbix"
    snmp_v3_auth_pass: "{{ vault_snmp_v3_auth_pass }}"
    snmp_v3_priv_pass: "{{ vault_snmp_v3_priv_pass }}"
    zabbix_server_ip: "172.16.0.40"

  tasks:

    - name: "SNMPv2c | Configure NX-OS SNMP"
      cisco.nxos.nxos_config:
        lines:
          - "snmp-server community {{ snmp_community_ro }} ro"
          - "snmp-server location Lab/Proxmox"
          - "snmp-server contact admin@lab.local"
          - "snmp-server host {{ zabbix_server_ip }} traps version 2c {{ snmp_community_ro }}"
          - "snmp-server enable traps snmp linkdown linkup"
          - "snmp-server enable traps bgp"
        save_when: changed
      tags: [snmp, snmp_v2c]

    - name: "SNMPv3 | Configure NX-OS SNMP v3 user"
      cisco.nxos.nxos_config:
        lines:
          - "snmp-server user {{ snmp_v3_user }} network-admin auth sha
             {{ snmp_v3_auth_pass }} priv aes-128 {{ snmp_v3_priv_pass }}"
        save_when: changed
      no_log: true
      tags: [snmp, snmp_v3]

    - name: "Verify | Show NX-OS SNMP"
      cisco.nxos.nxos_command:
        commands: [show snmp user, show snmp community]
      register: snmp_verify
      tags: [snmp, verify]

    - name: "Verify | Display"
      ansible.builtin.debug:
        msg: "{{ snmp_verify.stdout }}"
      tags: [snmp, verify]

# ── Palo Alto PAN-OS ─────────────────────────────────────────────────
- name: "SNMP | Configure PAN-OS SNMP"
  hosts: panos_devices
  gather_facts: false
  tags: [snmp]

  vars:
    snmp_community_ro: "{{ vault_snmp_community_ro }}"
    snmp_v3_user: "zabbix"
    snmp_v3_auth_pass: "{{ vault_snmp_v3_auth_pass }}"
    snmp_v3_priv_pass: "{{ vault_snmp_v3_priv_pass }}"
    zabbix_server_ip: "172.16.0.40"

  tasks:

    - name: "SNMPv2c | Configure PAN-OS SNMP v2c"
      paloaltonetworks.panos.panos_config_element:
        provider: "{{ panos_provider }}"
        xpath: "/config/devices/entry[@name='localhost.localdomain']/deviceconfig/system"
        element: |
          <snmp-setting>
            <access-setting>
              <version>
                <v2c>
                  <snmp-community-string>{{ snmp_community_ro }}</snmp-community-string>
                </v2c>
              </version>
            </access-setting>
            <snmp-system>
              <location>Lab/Proxmox</location>
              <contact>admin@lab.local</contact>
            </snmp-system>
          </snmp-setting>
      tags: [snmp, snmp_v2c]

    - name: "SNMPv3 | Configure PAN-OS SNMP v3"
      paloaltonetworks.panos.panos_config_element:
        provider: "{{ panos_provider }}"
        xpath: "/config/devices/entry[@name='localhost.localdomain']/deviceconfig/system/snmp-setting/access-setting/version/v3"
        element: |
          <views>
            <entry name="all">
              <view>
                <entry name="all">
                  <oid>1.3.6.1</oid>
                  <option>include</option>
                  <mask>0xff</mask>
                </entry>
              </view>
            </entry>
          </views>
          <groups>
            <entry name="ZABBIX-V3">
              <view>all</view>
              <users>
                <entry name="{{ snmp_v3_user }}">
                  <authpwd>{{ snmp_v3_auth_pass }}</authpwd>
                  <privpwd>{{ snmp_v3_priv_pass }}</privpwd>
                </entry>
              </users>
            </entry>
          </groups>
      no_log: true
      tags: [snmp, snmp_v3]

    - name: "PAN-OS | Commit SNMP changes"
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"
      tags: [snmp]
EOF
```

### Add SNMP secrets to vault

```bash
# Add to inventory/group_vars/all/vault.yml:
# vault_snmp_community_ro: "lab-monitor-ro"
# vault_snmp_v3_auth_pass: "SnmpAuth#2025!"
# vault_snmp_v3_priv_pass: "SnmpPriv#2025!"

# Run v2c first — verify it works before enabling v3
ansible-playbook \
  playbooks/infrastructure/zabbix/configure_snmp.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt \
  --tags snmp_v2c

# Verify SNMP from monitoring VM
ssh ansible@172.16.0.40
# Test SNMP v2c poll against wan-r1
docker exec zabbix-server snmpget \
  -v 2c \
  -c lab-monitor-ro \
  172.16.0.11 \
  1.3.6.1.2.1.1.1.0    # sysDescr OID
# Expected: STRING: "Cisco IOS XE Software..."

# Test interface table
docker exec zabbix-server snmpwalk \
  -v 2c \
  -c lab-monitor-ro \
  172.16.0.11 \
  1.3.6.1.2.1.2.2.1.2  # ifDescr table
# Expected: list of interface names

# Once v2c is confirmed working, enable v3
ansible-playbook \
  playbooks/infrastructure/zabbix/configure_snmp.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt \
  --tags snmp_v3

# Verify v3
docker exec zabbix-server snmpget \
  -v 3 \
  -u zabbix \
  -l authPriv \
  -a SHA \
  -A "SnmpAuth#2025!" \
  -x AES \
  -X "SnmpPriv#2025!" \
  172.16.0.11 \
  1.3.6.1.2.1.1.1.0
```

### v2c vs v3 — why both matter

```
SNMP v2c in production:
  Community string = password sent in clear text over UDP
  Anyone on the same network segment can capture it with tcpdump
  Many legacy devices only support v2c — no choice
  Still the most common in enterprise environments today

SNMP v3 improvements:
  Authentication: SHA-256 hash proves the message came from
                  a trusted source — prevents spoofed SNMP responses
  Privacy:        AES-128/256 encryption — community string
                  equivalent is never transmitted in clear text
  Message timing: Prevents replay attacks

Lab strategy (what this guide uses):
  Configure v2c first — easier to troubleshoot, instant feedback
  Confirm SNMP connectivity to all 7 devices
  Configure v3 on top — does not remove v2c (both active)
  In Zabbix, create v3-only hosts — v2c community becomes
  a fallback/emergency access method only

  In a real production environment:
  Deploy v3 only. Remove v2c communities after migration.
  Segment SNMP management traffic to a dedicated VLAN.
  Restrict SNMP access with ACLs (snmp-server community X RO <acl>).
```

---

## 37.6 — Register Hosts in Zabbix via Ansible

Ansible's `community.zabbix` collection drives the Zabbix API to create host groups, import templates, and register all devices — no clicking through the UI.

```bash
# Install community.zabbix collection
ansible-galaxy collection install community.zabbix

cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/register_hosts.yml << 'EOF'
---
# Register all lab devices and VMs in Zabbix via API
# Usage: ansible-playbook playbooks/infrastructure/zabbix/register_hosts.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Zabbix | Register hosts and configure monitoring"
  hosts: localhost
  gather_facts: false
  tags: [zabbix, hosts]

  vars:
    zabbix_url: "http://172.16.0.40"
    zabbix_user: Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"

    # Network device host definitions
    network_hosts:
      - name: wan-r1
        ip: "172.16.0.11"
        groups: ["Network Devices", "Cisco IOS-XE", "WAN Routers"]
        templates: ["Cisco IOS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: wan-r2
        ip: "172.16.0.12"
        groups: ["Network Devices", "Cisco IOS-XE", "WAN Routers"]
        templates: ["Cisco IOS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: dist-01
        ip: "172.16.0.21"
        groups: ["Network Devices", "Cisco IOS-XE", "Distribution"]
        templates: ["Cisco IOS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: spine-01
        ip: "172.16.0.31"
        groups: ["Network Devices", "Cisco NX-OS", "Spine Switches"]
        templates: ["Cisco NX-OS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: leaf-01
        ip: "172.16.0.33"
        groups: ["Network Devices", "Cisco IOS-XE", "Leaf Switches"]
        templates: ["Cisco IOS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: leaf-02
        ip: "172.16.0.34"
        groups: ["Network Devices", "Cisco IOS-XE", "Leaf Switches"]
        templates: ["Cisco IOS by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

      - name: panos-fw01
        ip: "172.16.0.51"
        groups: ["Network Devices", "Palo Alto", "Firewalls"]
        templates: ["Palo Alto PA-Series by SNMP"]
        snmp_community: "{{ vault_snmp_community_ro }}"

    # Infrastructure VM host definitions
    infra_hosts:
      - name: netbox
        ip: "192.168.100.20"
        groups: ["Infrastructure VMs", "Netbox"]
        templates: ["Linux by Zabbix agent"]

      - name: automation
        ip: "192.168.100.30"
        groups: ["Infrastructure VMs", "Automation"]
        templates: ["Linux by Zabbix agent"]

      - name: monitoring
        ip: "192.168.100.40"
        groups: ["Infrastructure VMs", "Monitoring"]
        templates: ["Linux by Zabbix agent"]

      - name: observability
        ip: "192.168.100.50"
        groups: ["Infrastructure VMs", "Observability"]
        templates: ["Linux by Zabbix agent"]

      - name: logging
        ip: "192.168.100.60"
        groups: ["Infrastructure VMs", "Logging"]
        templates: ["Linux by Zabbix agent"]

  tasks:

    # ── Host groups ───────────────────────────────────────────────────
    - name: "Groups | Create host groups"
      community.zabbix.zabbix_group:
        server_url: "{{ zabbix_url }}"
        login_user: "{{ zabbix_user }}"
        login_password: "{{ zabbix_password }}"
        state: present
        host_groups:
          - Network Devices
          - Cisco IOS-XE
          - Cisco NX-OS
          - Palo Alto
          - Firewalls
          - WAN Routers
          - Distribution
          - Spine Switches
          - Leaf Switches
          - Infrastructure VMs
          - Netbox
          - Automation
          - Monitoring
          - Observability
          - Logging

    # ── Network device hosts (SNMP) ───────────────────────────────────
    - name: "Hosts | Register network device: {{ item.name }}"
      community.zabbix.zabbix_host:
        server_url: "{{ zabbix_url }}"
        login_user: "{{ zabbix_user }}"
        login_password: "{{ zabbix_password }}"
        host_name: "{{ item.name }}"
        host_groups: "{{ item.groups }}"
        link_templates: "{{ item.templates }}"
        status: enabled
        state: present
        interfaces:
          - type: 2                    # 2 = SNMP interface
            main: 1
            useip: 1
            ip: "{{ item.ip }}"
            dns: ""
            port: "161"
            details:
              version: 2               # SNMPv2c to start
              bulk: 1
              community: "{{ item.snmp_community }}"
        inventory_mode: automatic
        inventory:
          location: "Lab / Proxmox Host"
          contact: "admin@lab.local"
      loop: "{{ network_hosts }}"
      loop_control:
        label: "{{ item.name }}"

    # ── Infrastructure VM hosts (Zabbix agent) ────────────────────────
    - name: "Hosts | Register infrastructure VM: {{ item.name }}"
      community.zabbix.zabbix_host:
        server_url: "{{ zabbix_url }}"
        login_user: "{{ zabbix_user }}"
        login_password: "{{ zabbix_password }}"
        host_name: "{{ item.name }}"
        host_groups: "{{ item.groups }}"
        link_templates: "{{ item.templates }}"
        status: enabled
        state: present
        interfaces:
          - type: 1                    # 1 = Zabbix agent interface
            main: 1
            useip: 1
            ip: "{{ item.ip }}"
            dns: ""
            port: "10050"
        inventory_mode: automatic
      loop: "{{ infra_hosts }}"
      loop_control:
        label: "{{ item.name }}"

    - name: "Hosts | Report registration complete"
      ansible.builtin.debug:
        msg:
          - "Registered {{ network_hosts | length }} network devices (SNMP)"
          - "Registered {{ infra_hosts | length }} infrastructure VMs (agent)"
          - "View hosts: http://172.16.0.40/zabbix/hosts.php"
EOF
```

### ### ℹ️ UI callout: What was just created

```
After running register_hosts.yml, open the Zabbix web UI:
http://172.16.0.40  →  Admin / (vault_zabbix_web_password)

Monitoring → Hosts:
  Shows all 12 registered hosts. Each has:
  - Availability indicator (green SNMP = responding to SNMP polls)
  - Templates column (linked template names)
  - Click any host → Items to see all collected metrics

Configuration → Host groups:
  Shows the group hierarchy — Network Devices with sub-groups,
  Infrastructure VMs with sub-groups. Host groups drive dashboard
  filtering and alert routing.

Configuration → Templates:
  Built-in templates like "Cisco IOS by SNMP" contain:
  - 50-100+ pre-configured items (ifOperStatus, sysUptime, BGP MIBs...)
  - Pre-built triggers (interface down, high CPU, BGP peer lost)
  - Graphs (interface utilisation, CPU/memory trends)
  All of this comes free with the template — nothing to configure manually.
```

---

## 37.7 — Templates: Built-in and Custom

### Built-in templates — what they provide

```bash
# The official Cisco IOS by SNMP template includes:
# Items:
#   - sysUptime        (1.3.6.1.2.1.1.3.0)
#   - sysDescr         (1.3.6.1.2.1.1.1.0)
#   - ifOperStatus     (auto-discovered per interface)
#   - ifInOctets/ifOutOctets (traffic counters per interface)
#   - cpuUsage         (Cisco-specific OID)
#   - memoryUsed/Free  (Cisco-specific OID)
#   - bgpPeerState     (if BGP MIB enabled)
#
# Triggers (pre-configured thresholds):
#   - Interface down (ifOperStatus = 2)
#   - High CPU > 90% for 5 minutes
#   - Device unreachable (SNMP timeout)
#   - BGP peer state not established
#
# Graphs:
#   - Interface traffic (in/out octets per interface)
#   - CPU utilisation over time
#   - Memory utilisation over time
#
# Discovery rules:
#   - Network interfaces (automatically creates items for each interface)
#   - BGP peers (creates items for each BGP neighbor)

# Verify template is being used correctly — check items on wan-r1
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "item.get",
    "params": {
      "output": ["name","key_","lastvalue","lastclock"],
      "hostids": ["10001"],
      "sortfield": "name",
      "limit": 20
    },
    "auth": "'"${ZABBIX_API_TOKEN}"'",
    "id": 1
  }' \
  http://172.16.0.40/api_jsonrpc.php \
  | python3 -m json.tool | grep -E '"name"|"lastvalue"'
```

### Custom template — extending the built-in

The built-in Cisco IOS template doesn't include BGP prefix counts or specific interface error counters. This shows how to add custom items and a trigger to an existing template.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/custom_template.yml << 'EOF'
---
# Create a custom Zabbix template extending Cisco IOS by SNMP
# Adds: BGP prefix count per peer, interface error rate trigger

- name: "Zabbix | Create custom network template"
  hosts: localhost
  gather_facts: false
  tags: [zabbix, templates]

  vars:
    zabbix_url: "http://172.16.0.40"
    zabbix_user: Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"

  tasks:

    - name: "Auth | Get Zabbix API token"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: user.login
          params:
            username: "{{ zabbix_user }}"
            password: "{{ zabbix_password }}"
          id: 1
      register: auth_result
      no_log: true

    - name: "Auth | Set token fact"
      ansible.builtin.set_fact:
        zbx_token: "{{ auth_result.json.result }}"
      no_log: true

    - name: "Template | Create lab custom template"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: template.create
          params:
            host: "Lab Network Extensions"
            name: "Lab Network Extensions"
            description: >
              Custom items extending built-in Cisco templates.
              Apply alongside 'Cisco IOS by SNMP'.
            groups:
              - groupid: "1"    # Templates group
          auth: "{{ zbx_token }}"
          id: 2
        status_code: 200
      register: template_result
      changed_when: template_result.json.result is defined

    - name: "Template | Add BGP prefix count item"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: item.create
          params:
            name: "BGP prefix count from {#BGPPEERREMOTEADDR}"
            key_: "bgpPeerFsmEstablishedTransitions[{#BGPPEERREMOTEADDR}]"
            hostid: "{{ template_result.json.result.templateids[0] }}"
            type: 1          # 1 = SNMP agent
            snmp_oid: "1.3.6.1.2.1.15.3.1.7.{#BGPPEERREMOTEADDR}"
            value_type: 3    # 3 = unsigned integer
            delay: "5m"
            description: "Number of prefixes received from this BGP peer"
          auth: "{{ zbx_token }}"
          id: 3
        status_code: 200

    - name: "Template | Add interface error rate trigger"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: trigger.create
          params:
            description: "High interface error rate on {HOST.NAME} {#IFNAME}"
            expression: >
              change(/Lab Network Extensions/
              net.if.in.errors[{#IFNAME}])>100
            priority: 3      # 3 = Average
            comments: >
              Interface error count increased by more than 100 in
              the last check interval. Investigate physical layer
              or duplex mismatch.
          auth: "{{ zbx_token }}"
          id: 4
        status_code: 200

    - name: "Template | Link custom template to all Cisco IOS hosts"
      community.zabbix.zabbix_host:
        server_url: "{{ zabbix_url }}"
        login_user: "{{ zabbix_user }}"
        login_password: "{{ zabbix_password }}"
        host_name: "{{ item }}"
        link_templates:
          - "Cisco IOS by SNMP"
          - "Lab Network Extensions"
        state: present
      loop:
        - wan-r1
        - wan-r2
        - dist-01
        - leaf-01
        - leaf-02
EOF
```

### ### 💡 Tip: Template inheritance in Zabbix

> The correct pattern is to never modify a built-in template directly — Zabbix upgrades can overwrite customisations. Instead, create a separate template for lab extensions and link both to the host. The host then inherits items and triggers from all linked templates. This is the same principle as Ansible role composition — keep the upstream role untouched and override with your own.

---

## 37.8 — Alert Triggers in Detail

### Trigger 1: Interface up/down

```
This trigger comes pre-built in the Cisco IOS by SNMP template.
It uses Zabbix's Low-Level Discovery (LLD) — Zabbix automatically
discovers all interfaces on the device via SNMP and creates an
ifOperStatus item for each one.

OID polled: 1.3.6.1.2.1.2.2.1.8.{#SNMPINDEX}  (ifOperStatus)
Values:     1 = up, 2 = down, 3 = testing, 7 = lowerLayerDown

Trigger expression (from built-in template):
  last(/Cisco IOS by SNMP/net.if.status[ifOperStatus.{#SNMPINDEX}])=2
    and last(/Cisco IOS by SNMP/net.if.status[ifOperStatus.{#SNMPINDEX}],#1:now-1m)=1

Translation:
  "Current value is 2 (down) AND one minute ago it was 1 (up)"
  The two-condition approach prevents a false trigger if an interface
  was already down when Zabbix first polled it.

Priority:  High
Recovery:  Fires automatically when ifOperStatus returns to 1

To test:
  # Shut an interface on wan-r1 from the network-lab VM
  ssh ansible@172.16.0.10
  ssh ansible@172.16.0.11   # wan-r1
  conf t
  interface GigabitEthernet2
    shutdown
  end
  # Watch Zabbix Monitoring → Problems — trigger should fire within 1-2 polling cycles
  # (default poll interval is 1 minute)

  # Restore:
  conf t
  interface GigabitEthernet2
    no shutdown
  end
```

### Trigger 2: BGP peer state loss

```
BGP peer state is tracked via the BGP4-MIB.
OID: 1.3.6.1.2.1.15.3.1.2.{#BGPPEERREMOTEADDR}  (bgpPeerState)
Values: 1=idle, 2=connect, 3=active, 4=opensent, 5=openconfirm, 6=established

The built-in template discovers BGP peers and creates items per peer.

Trigger expression:
  last(/Cisco IOS by SNMP/bgpPeerState[{#BGPPEERREMOTEADDR}])<>6
    and last(/Cisco IOS by SNMP/bgpPeerState[{#BGPPEERREMOTEADDR}],#1:now-3m)=6

Translation:
  "Current BGP state is NOT established AND 3 minutes ago it WAS established"
  The 3-minute lookback prevents false alerts during planned maintenance
  or device restart — BGP reconverges, and if it's back up within 3 minutes
  no alert fires.

Priority: Disaster (highest)
Recovery: Fires when bgpPeerState returns to 6 (established)

To test:
  ssh ansible@172.16.0.11   # wan-r1
  conf t
  router bgp 65001
    neighbor 10.0.0.1 shutdown   # panos-fw01 peer
  end
  # Zabbix will show BGP peer loss trigger within next poll cycle
  # Restore: no neighbor 10.0.0.1 shutdown
```

### Trigger 3: High CPU / memory

```
CPU trigger (Cisco IOS — uses Cisco-specific OID):
OID: 1.3.6.1.4.1.9.2.1.57.0  (avgBusy5 — 5-minute CPU average)

Trigger expression:
  avg(/Cisco IOS by SNMP/system.cpu.util[cpmCPUTotal5min],5m)>90

Translation:
  "Average CPU utilisation over the last 5 minutes exceeds 90%"
  Uses avg() not last() — prevents false alerts from brief CPU spikes
  during BGP route processing or interface event handling.

Priority: High
Threshold: 90% (warning trigger at 75% is also recommended)

Memory trigger (Cisco IOS):
OID: 1.3.6.1.4.1.9.9.48.1.1.1.6.1  (ciscoMemoryPoolFree — processor pool)

Custom trigger expression:
  last(/Cisco IOS by SNMP/vm.memory.utilization[ciscoMemoryPoolUsed])>85

To add this custom threshold via Ansible (if not in built-in template):
  The custom_template.yml playbook structure above shows how to add
  triggers programmatically — adapt the trigger.create API call with
  the memory OID and 85% threshold.
```

---

## 37.9 — Alert Actions (Notifications)

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/configure_alerts.yml << 'EOF'
---
# Configure Zabbix alert actions
# Adds: email notification + webhook to a Slack/Teams channel

- name: "Zabbix | Configure alert actions"
  hosts: localhost
  gather_facts: false
  tags: [zabbix, alerts]

  vars:
    zabbix_url: "http://172.16.0.40"
    zabbix_user: Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"

  tasks:

    - name: "Auth | Get API token"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: user.login
          params:
            username: "{{ zabbix_user }}"
            password: "{{ zabbix_password }}"
          id: 1
      register: auth_result
      no_log: true

    - name: "Auth | Set token"
      ansible.builtin.set_fact:
        zbx_token: "{{ auth_result.json.result }}"
      no_log: true

    # ── Media type: Slack webhook ─────────────────────────────────────
    - name: "Media | Create Slack webhook media type"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: mediatype.create
          params:
            name: "Slack webhook"
            type: 4              # 4 = webhook
            parameters:
              - name: HTTPProxy
                value: ""
              - name: Message
                value: "{ALERT.MESSAGE}"
              - name: ParseMode
                value: markdown
              - name: Subject
                value: "{ALERT.SUBJECT}"
              - name: URL
                value: "{{ vault_slack_webhook_url }}"
            script: |
              var Slack = {
                  sendMessage: function(url, message) {
                      var request = new HttpRequest();
                      request.addHeader('Content-Type: application/json');
                      var response = request.post(url, JSON.stringify({
                          text: message
                      }));
                      if (request.getStatus() != 200) {
                          throw 'Response code: ' + request.getStatus();
                      }
                      return response;
                  }
              };
              try {
                  var params = JSON.parse(value);
                  Slack.sendMessage(params.URL, '*' + params.Subject + '*\n' + params.Message);
                  return 'OK';
              } catch(error) {
                  throw 'Failed: ' + error;
              }
          auth: "{{ zbx_token }}"
          id: 2
        status_code: 200

    # ── Action: alert on trigger ──────────────────────────────────────
    - name: "Action | Create network device alert action"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: action.create
          params:
            name: "Network Device Alert"
            eventsource: 0       # 0 = trigger events
            status: 0            # 0 = enabled
            filter:
              evaltype: 0        # AND
              conditions:
                - conditiontype: 4   # 4 = host group
                  operator: 0        # =
                  value: "Network Devices"
                - conditiontype: 5   # 5 = trigger severity
                  operator: 5        # >=
                  value: "3"         # Average and above
            operations:
              - operationtype: 0   # 0 = send message
                opmessage:
                  default_msg: 1
                  mediatypeid: "1"   # Email (built-in)
                  subject: >
                    [{TRIGGER.SEVERITY}] {TRIGGER.NAME} on {HOST.NAME}
                  message: |
                    Host: {HOST.NAME} ({HOST.IP})
                    Trigger: {TRIGGER.NAME}
                    Severity: {TRIGGER.SEVERITY}
                    Status: {TRIGGER.STATUS}
                    Time: {EVENT.DATE} {EVENT.TIME}
                    Item: {ITEM.NAME} = {ITEM.VALUE}
                    URL: http://172.16.0.40/tr_events.php?triggerid={TRIGGER.ID}
                opmessage_grp:
                  - usrgrpid: "7"    # Zabbix administrators group
          auth: "{{ zbx_token }}"
          id: 3
        status_code: 200
EOF
```

---

## 37.10 — Zabbix API Reference Playbook

A reusable operations playbook for common Zabbix API tasks.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/manage.yml << 'EOF'
---
# Zabbix day-to-day management operations
# Usage:
#   List all hosts:
#   ansible-playbook zabbix/manage.yml -e "action=list_hosts"
#
#   Get problems (active triggers):
#   ansible-playbook zabbix/manage.yml -e "action=list_problems"
#
#   Enable/disable host:
#   ansible-playbook zabbix/manage.yml -e "action=disable_host host=wan-r1"
#   ansible-playbook zabbix/manage.yml -e "action=enable_host host=wan-r1"
#
#   Acknowledge problem:
#   ansible-playbook zabbix/manage.yml -e "action=ack_problem event_id=123 msg='Investigating'"

- name: "Zabbix | Management operations"
  hosts: localhost
  gather_facts: false

  vars:
    zabbix_url: "http://172.16.0.40"
    zabbix_user: Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"
    action: ~

  tasks:

    - name: "Assert action set"
      ansible.builtin.assert:
        that: action is not none
        fail_msg: "Provide -e 'action=<list_hosts|list_problems|enable_host|disable_host|ack_problem>'"

    - name: "Auth | Get API token"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: user.login
          params:
            username: "{{ zabbix_user }}"
            password: "{{ zabbix_password }}"
          id: 1
      register: auth
      no_log: true

    - name: "Set token"
      ansible.builtin.set_fact:
        zbx_token: "{{ auth.json.result }}"
      no_log: true

    # ── List hosts ────────────────────────────────────────────────────
    - name: "List | Get all hosts"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: host.get
          params:
            output: [hostid, host, status]
            selectInterfaces: [ip, port, type]
            selectGroups: [name]
          auth: "{{ zbx_token }}"
          id: 2
      register: hosts_result
      when: action == 'list_hosts'

    - name: "List | Display hosts"
      ansible.builtin.debug:
        msg: >
          {{ item.host }} | IP: {{ item.interfaces[0].ip }}
          | Status: {{ 'enabled' if item.status == '0' else 'disabled' }}
          | Groups: {{ item.groups | map(attribute='name') | join(', ') }}
      loop: "{{ hosts_result.json.result }}"
      loop_control:
        label: "{{ item.host }}"
      when: action == 'list_hosts'

    # ── List active problems ───────────────────────────────────────────
    - name: "Problems | Get active triggers"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: problem.get
          params:
            output: extend
            selectAcknowledges: extend
            recent: true
            sortfield: [eventid]
            sortorder: DESC
            limit: 20
          auth: "{{ zbx_token }}"
          id: 2
      register: problems
      when: action == 'list_problems'

    - name: "Problems | Display"
      ansible.builtin.debug:
        msg: >
          [{{ item.severity | int | replace('0','Not classified')
              | replace('1','Info') | replace('2','Warning')
              | replace('3','Average') | replace('4','High')
              | replace('5','Disaster') }}]
          {{ item.name }} (eventid: {{ item.eventid }})
      loop: "{{ problems.json.result | default([]) }}"
      loop_control:
        label: "{{ item.eventid }}"
      when: action == 'list_problems'

    # ── Enable / disable host ─────────────────────────────────────────
    - name: "Host | Get hostid for {{ host | default('') }}"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: host.get
          params:
            output: [hostid]
            filter:
              host: ["{{ host | default('') }}"]
          auth: "{{ zbx_token }}"
          id: 2
      register: host_lookup
      when: action in ['enable_host', 'disable_host']

    - name: "Host | Update status"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: host.update
          params:
            hostid: "{{ host_lookup.json.result[0].hostid }}"
            status: "{{ '0' if action == 'enable_host' else '1' }}"
          auth: "{{ zbx_token }}"
          id: 3
      when: action in ['enable_host', 'disable_host']

    - name: "Host | Confirm"
      ansible.builtin.debug:
        msg: "Host {{ host }} {{ action | replace('_host', 'd') }}"
      when: action in ['enable_host', 'disable_host']
EOF
```

---

## 37.11 — Checklist

```
[ ] Monitoring VM prepared:
    [ ] Docker and docker-compose-plugin installed
    [ ] /opt/zabbix/{mysql,alertscripts,externalscripts,snmptraps} created

[ ] Zabbix stack deployed:
    [ ] ~/zabbix/docker-compose.yml created
    [ ] ~/zabbix/.env created with generated MySQL passwords
    [ ] Secrets added to Ansible vault:
        vault_mysql_root_password
        vault_mysql_zabbix_password
        vault_zabbix_web_password
        vault_zabbix_api_user
    [ ] docker compose ps — all 5 containers running/healthy
    [ ] http://172.16.0.40 loads Zabbix web UI
    [ ] Admin password changed from default (zabbix) to vault value
    [ ] API version endpoint returns 7.0.x

[ ] Ansible deploy playbook:
    [ ] playbooks/infrastructure/zabbix/deploy.yml created and tested
    [ ] Idempotent — second run shows no unexpected CHANGED

[ ] SNMP configured on all 7 network devices:
    [ ] SNMPv2c:
        [ ] wan-r1  — snmpget returns sysDescr
        [ ] wan-r2  — snmpget returns sysDescr
        [ ] dist-01 — snmpget returns sysDescr
        [ ] spine-01 — snmpget returns sysDescr
        [ ] leaf-01 — snmpget returns sysDescr
        [ ] leaf-02 — snmpget returns sysDescr
        [ ] panos-fw01 — snmpget returns sysDescr
    [ ] SNMPv3:
        [ ] All devices — snmpget with -v 3 -u zabbix returns data
        [ ] v3 credentials in vault: vault_snmp_v3_auth_pass, vault_snmp_v3_priv_pass

[ ] Hosts registered in Zabbix:
    [ ] register_hosts.yml ran successfully
    [ ] 7 network devices visible in Monitoring → Hosts (SNMP interface)
    [ ] 5 infrastructure VMs visible (agent interface)
    [ ] All network devices show green SNMP availability indicator
    [ ] Templates linked: Cisco IOS by SNMP, Cisco NX-OS by SNMP,
        Palo Alto PA-Series by SNMP, Linux by Zabbix agent

[ ] Templates verified:
    [ ] Built-in items collecting data on wan-r1:
        [ ] sysUptime — has a value
        [ ] CPU utilisation — has a value
        [ ] Interface discovery ran — items created per interface
        [ ] BGP discovery ran (if BGP MIB enabled) — peer items visible
    [ ] Custom "Lab Network Extensions" template created
    [ ] Custom template linked to all Cisco IOS-XE hosts

[ ] Alert triggers tested:
    [ ] Interface down trigger:
        [ ] Shut GigabitEthernet2 on wan-r1
        [ ] Trigger fires in Monitoring → Problems within 2 poll cycles
        [ ] No shutdown → trigger resolves automatically
    [ ] BGP peer loss trigger:
        [ ] Shut BGP neighbor on wan-r1
        [ ] bgpPeerState trigger fires
        [ ] Restore → trigger resolves
    [ ] CPU trigger visible in Configuration → Triggers (threshold set)

[ ] Alert actions configured:
    [ ] configure_alerts.yml ran
    [ ] Slack webhook media type created (if vault_slack_webhook_url set)
    [ ] "Network Device Alert" action created for High+ severity

[ ] Management playbook tested:
    [ ] manage.yml -e "action=list_hosts" returns all 12 hosts
    [ ] manage.yml -e "action=list_problems" returns current problems
    [ ] manage.yml -e "action=disable_host host=wan-r1" works
    [ ] manage.yml -e "action=enable_host host=wan-r1" restores it

[ ] UFW rules verified:
    [ ] Port 80 open from 172.16.0.0/24 (Zabbix web)
    [ ] Port 10051 open from 192.168.100.0/24 (active agents)
    [ ] Port 162 UDP open (SNMP traps)
    [ ] Port 5432 NOT accessible externally

[ ] Snapshot taken:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=104 snap_name=post-zabbix \
          snap_description='Zabbix deployed, SNMP configured, hosts registered Part 37'"

[ ] All playbooks committed to Git and visible in Gitea:
    [ ] playbooks/infrastructure/zabbix/deploy.yml
    [ ] playbooks/infrastructure/zabbix/configure_snmp.yml
    [ ] playbooks/infrastructure/zabbix/register_hosts.yml
    [ ] playbooks/infrastructure/zabbix/custom_template.yml
    [ ] playbooks/infrastructure/zabbix/configure_alerts.yml
    [ ] playbooks/infrastructure/zabbix/manage.yml
```

---

*Zabbix is polling every device in the lab. An interface going down fires a trigger within two minutes. BGP peer loss triggers immediately. The Zabbix API is the management interface — no manual host registration, no clicking through templates. Part 38 builds the modern observability stack alongside this — Prometheus scraping the same devices, Grafana showing both Zabbix and Prometheus data in one dashboard, and Loki aggregating logs from every VM. The comparison between the two approaches will be concrete rather than theoretical.*

*Next up: **Part 38 — Modern Observability: Prometheus + Grafana + Loki***
