---
draft: true
title: '39 - Graylog'
description: "Part 39 of my Ansible learning geared towards Network Engineering."
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

## Part 39: Logging — Graylog + OpenSearch

> *Prometheus tells you what the metrics looked like. Zabbix tells you which triggers fired. Neither tells you what the device actually said. Syslog is where network devices speak in plain language — "BGP neighbor 10.0.0.1 Down", "%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet2, changed state to down", "committed by admin from console". Graylog collects those messages, parses them into structured fields, routes them into streams, enriches device IPs with hostnames from Netbox, and alerts on patterns that matter. By the end of this part, every log line from every device in the lab lands in a searchable, structured, correlated log platform that a network engineer can actually use.*

---

## 39.1 — Why Graylog Over Pure ELK

```
ELK stack (Elasticsearch + Logstash + Kibana):
  Pro:  Industry standard, huge ecosystem, excellent for
        application logs, rich Kibana visualisation
  Con:  Logstash configuration is complex and verbose
        Kibana is not purpose-built for network operations
        No built-in SNMP trap receiver
        No native syslog input (requires Logstash config)
        Alert rules require Kibana license features
        Resource-heavy — Elasticsearch + Kibana + Logstash
        is a lot for a single VM with 12GB RAM

Graylog + OpenSearch:
  Pro:  Native syslog input (UDP/TCP 514) — zero config for devices
        Native SNMP trap receiver
        Built-in extractors and pipeline rules for log parsing
        Stream-based routing is purpose-built for log triage
        Web UI designed for search and alert workflows
        Ships with OpenSearch (open-source Elasticsearch fork)
        API-driven configuration — Ansible can manage everything
  Con:  Smaller community than ELK
        Graylog UI less polished than Kibana for dashboards

Why OpenSearch instead of Elasticsearch:
  OpenSearch is the AWS-maintained open-source fork of Elasticsearch
  created after Elastic changed its licence in 2021. OpenSearch is
  fully open source (Apache 2.0), actively maintained, and is what
  Graylog recommends for production deployments. It is functionally
  identical to Elasticsearch 7.10 for Graylog's purposes.
```

---

## 39.2 — Architecture

```
logging VM (172.16.0.60 / 192.168.100.60)
│
├── Docker: opensearch      — Index storage and search (port 9200 internal)
├── Docker: graylog         — Log processing, streams, alerts (port 9000 web)
├── Docker: mongodb         — Graylog configuration database (port 27017 internal)
│
├── Inputs listening:
│   ├── UDP 514  — Syslog from network devices
│   ├── TCP 514  — Syslog (reliable, for VMs/servers)
│   └── UDP/TCP 12201 — GELF (structured logs from VMs via Promtail/rsyslog)
│
└── Native: SNMP trap receiver
    └── UDP 162 — SNMP traps from network devices → Graylog

Inbound log flows:
  wan-r1/wan-r2   ──UDP 514──► Graylog (Cisco IOS syslog format)
  dist-01         ──UDP 514──► Graylog (Cisco IOS-XE syslog format)
  spine-01        ──UDP 514──► Graylog (NX-OS syslog format)
  leaf-01/leaf-02 ──UDP 514──► Graylog (Cisco IOS-XE syslog format)
  panos-fw01      ──UDP 514──► Graylog (PAN-OS syslog format)
  All VMs         ──TCP 514──► Graylog (rsyslog from harden.yml Part 34)

Processing pipeline:
  Ingest → Input (syslog/GELF) → Extractor (field split)
  → Pipeline rules (parse, enrich, normalise)
  → Stream routing (Network Devices / BGP Events / Config Changes)
  → OpenSearch index → Search / Alerts

Netbox enrichment:
  Pipeline rule queries Netbox API to resolve source IP → device hostname
  Adds: device_name, device_role, device_platform fields to every message
```

---

## 39.3 — Deploy with Docker Compose

```bash
# SSH to logging VM
ssh ansible@172.16.0.60

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker ansible
newgrp docker

# Create directories
sudo mkdir -p /opt/logging/{opensearch,graylog,mongodb}
sudo mkdir -p /opt/logging/graylog/{data,journal,config}
sudo chown -R ansible:ansible /opt/logging

# OpenSearch requires vm.max_map_count — without this it will crash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Verify the setting persisted
sysctl vm.max_map_count
```

### Generate secrets

```bash
# Graylog requires a password secret (min 16 chars) and a SHA2 of the admin password
GRAYLOG_PASSWORD_SECRET=$(openssl rand -base64 32 | tr -d '\n')
GRAYLOG_ADMIN_PASSWORD="GraylogAdmin#2025"
GRAYLOG_ROOT_PASSWORD_SHA2=$(echo -n "${GRAYLOG_ADMIN_PASSWORD}" | sha256sum | awk '{print $1}')
MONGODB_ROOT_PASSWORD=$(openssl rand -base64 24 | tr -d '\n')

mkdir -p ~/logging

cat > ~/logging/.env << EOF
GRAYLOG_PASSWORD_SECRET=${GRAYLOG_PASSWORD_SECRET}
GRAYLOG_ROOT_PASSWORD_SHA2=${GRAYLOG_ROOT_PASSWORD_SHA2}
MONGODB_ROOT_PASSWORD=${MONGODB_ROOT_PASSWORD}
GRAYLOG_ADMIN_PASSWORD=${GRAYLOG_ADMIN_PASSWORD}
EOF
chmod 600 ~/logging/.env

echo "Add to Ansible vault:"
echo "  vault_graylog_password_secret: ${GRAYLOG_PASSWORD_SECRET}"
echo "  vault_graylog_root_password_sha2: ${GRAYLOG_ROOT_PASSWORD_SHA2}"
echo "  vault_graylog_admin_password: ${GRAYLOG_ADMIN_PASSWORD}"
echo "  vault_mongodb_root_password: ${MONGODB_ROOT_PASSWORD}"
```

### Docker Compose file

```bash
cat > ~/logging/docker-compose.yml << 'EOF'
---
version: "3.8"

services:

  mongodb:
    image: mongo:6.0
    container_name: graylog-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: graylog
      MONGO_INITDB_ROOT_PASSWORD: "${MONGODB_ROOT_PASSWORD}"
    volumes:
      - /opt/logging/mongodb:/data/db
    networks:
      - graylog-backend
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  opensearch:
    image: opensearchproject/opensearch:2.12.0
    container_name: graylog-opensearch
    restart: unless-stopped
    environment:
      OPENSEARCH_JAVA_OPTS: "-Xms4g -Xmx4g"
      bootstrap.memory_lock: "true"
      cluster.name: graylog
      node.name: graylog-os-node-01
      discovery.type: single-node
      plugins.security.disabled: "true"       # Internal cluster — no TLS
      action.auto_create_index: "false"       # Graylog manages its own indices
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /opt/logging/opensearch:/usr/share/opensearch/data
    networks:
      - graylog-backend
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9200/_cluster/health | grep -qv '\"status\":\"red\"'"]
      interval: 20s
      timeout: 10s
      retries: 10

  graylog:
    image: graylog/graylog:6.0
    container_name: graylog
    restart: unless-stopped
    depends_on:
      mongodb:
        condition: service_healthy
      opensearch:
        condition: service_healthy
    environment:
      GRAYLOG_NODE_ID_FILE: "/usr/share/graylog/data/node-id"
      GRAYLOG_PASSWORD_SECRET: "${GRAYLOG_PASSWORD_SECRET}"
      GRAYLOG_ROOT_PASSWORD_SHA2: "${GRAYLOG_ROOT_PASSWORD_SHA2}"
      GRAYLOG_HTTP_BIND_ADDRESS: "0.0.0.0:9000"
      GRAYLOG_HTTP_EXTERNAL_URI: "http://172.16.0.60:9000/"
      GRAYLOG_MONGODB_URI: "mongodb://graylog:${MONGODB_ROOT_PASSWORD}@mongodb:27017/graylog?authSource=admin"
      GRAYLOG_ELASTICSEARCH_HOSTS: "http://opensearch:9200"
      GRAYLOG_REPORT_DISABLE_SANDBOX: "true"
      # Index settings
      GRAYLOG_ELASTICSEARCH_INDEX_PREFIX: "graylog"
      GRAYLOG_ELASTICSEARCH_MAX_NUMBER_OF_INDICES: "20"
      GRAYLOG_ELASTICSEARCH_RETENTION_STRATEGY: "delete"
      GRAYLOG_ELASTICSEARCH_MAX_SIZE_PER_INDEX: "1073741824"  # 1GB per index
      # Performance
      GRAYLOG_PROCESSBUFFER_PROCESSORS: 5
      GRAYLOG_OUTPUTBUFFER_PROCESSORS: 3
      GRAYLOG_MESSAGE_JOURNAL_MAX_SIZE: "4gb"
    volumes:
      - /opt/logging/graylog/data:/usr/share/graylog/data
      - /opt/logging/graylog/journal:/usr/share/graylog/data/journal
    ports:
      - "9000:9000"         # Graylog web UI + REST API
      - "514:514/udp"       # Syslog UDP
      - "514:514/tcp"       # Syslog TCP
      - "12201:12201/udp"   # GELF UDP
      - "12201:12201/tcp"   # GELF TCP
      - "162:162/udp"       # SNMP traps
    networks:
      - graylog-backend
      - graylog-frontend
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9000/api/system/lbstatus | grep -q ALIVE"]
      interval: 30s
      timeout: 10s
      retries: 15
      start_period: 120s    # Graylog takes ~2 minutes to initialise

networks:
  graylog-backend:
    driver: bridge
  graylog-frontend:
    driver: bridge
EOF
```

### Start the stack

```bash
cd ~/logging

# Pull images (OpenSearch is large — 600MB+, takes several minutes)
docker compose pull

# Start — OpenSearch initialisation takes 60-90 seconds
docker compose up -d

# Watch Graylog initialisation — wait for "Graylog server up and running"
docker compose logs -f graylog | grep -E "up and running|ERROR|WARN" &

# Verify health
docker compose ps

# Test web UI and API
curl -sf http://172.16.0.60:9000/api/system/lbstatus
# Expected: ALIVE

# Test API with credentials
curl -sf \
  -u "admin:GraylogAdmin#2025" \
  -H "X-Requested-By: ansible" \
  "http://172.16.0.60:9000/api/system" \
  | python3 -m json.tool | grep -E '"version"|"hostname"'
```

---

## 39.4 — Ansible Playbook: Deploy Graylog

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/graylog/templates

cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/deploy.yml << 'EOF'
---
# Deploy Graylog + OpenSearch + MongoDB on the logging VM
# Usage: ansible-playbook playbooks/infrastructure/graylog/deploy.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Deploy logging stack"
  hosts: logging_hosts
  gather_facts: true
  become: true
  tags: [graylog, deploy]

  vars:
    graylog_dir: /home/ansible/logging
    graylog_data_dir: /opt/logging

  tasks:

    - name: "Kernel | Set vm.max_map_count for OpenSearch"
      ansible.posix.sysctl:
        name: vm.max_map_count
        value: "262144"
        state: present
        reload: true

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

    - name: "Dirs | Create data directories"
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: ansible
        mode: '0755'
      loop:
        - "{{ graylog_dir }}"
        - "{{ graylog_data_dir }}/mongodb"
        - "{{ graylog_data_dir }}/opensearch"
        - "{{ graylog_data_dir }}/graylog/data"
        - "{{ graylog_data_dir }}/graylog/journal"

    - name: "Compose | Deploy docker-compose.yml"
      ansible.builtin.template:
        src: docker-compose.yml.j2
        dest: "{{ graylog_dir }}/docker-compose.yml"
        owner: ansible
        mode: '0644'

    - name: "Compose | Deploy environment file"
      ansible.builtin.template:
        src: graylog.env.j2
        dest: "{{ graylog_dir }}/.env"
        owner: ansible
        mode: '0600'

    - name: "UFW | Open Graylog ports"
      community.general.ufw:
        rule: allow
        port: "{{ item.port }}"
        proto: "{{ item.proto }}"
        src: "{{ item.src }}"
        comment: "{{ item.comment }}"
      loop:
        - { port: "9000",  proto: tcp, src: "172.16.0.0/24",    comment: "Graylog web UI + API" }
        - { port: "9000",  proto: tcp, src: "192.168.100.0/24", comment: "Graylog API from telemetry" }
        - { port: "514",   proto: udp, src: "any",              comment: "Syslog UDP from devices" }
        - { port: "514",   proto: tcp, src: "192.168.100.0/24", comment: "Syslog TCP from VMs" }
        - { port: "12201", proto: udp, src: "192.168.100.0/24", comment: "GELF UDP" }
        - { port: "12201", proto: tcp, src: "192.168.100.0/24", comment: "GELF TCP" }
        - { port: "162",   proto: udp, src: "any",              comment: "SNMP traps" }

    - name: "Docker | Start logging stack"
      community.docker.docker_compose_v2:
        project_src: "{{ graylog_dir }}"
        pull: always
        state: present
      become: false

    - name: "Graylog | Wait for API to be ready (up to 3 minutes)"
      ansible.builtin.uri:
        url: "http://172.16.0.60:9000/api/system/lbstatus"
        status_code: 200
        timeout: 10
      register: graylog_health
      retries: 18
      delay: 10
      until: "'ALIVE' in graylog_health.content"
      delegate_to: localhost

    - name: "Graylog | Report deployment"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════"
          - " Graylog is running"
          - " Web UI: http://172.16.0.60:9000"
          - " Login:  admin / (see vault_graylog_admin_password)"
          - " API:    http://172.16.0.60:9000/api/"
          - "════════════════════════════════════════"
EOF
```

---

## 39.5 — Configure Inputs via Graylog API

All Graylog configuration is done via the REST API — inputs, streams, pipelines, alerts. Ansible drives this entirely through the `ansible.builtin.uri` module. No clicking through the web UI required (though the UI callouts show what was created and why).

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/configure_inputs.yml << 'EOF'
---
# Create Graylog inputs for syslog and GELF
# Usage: ansible-playbook graylog/configure_inputs.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Configure inputs"
  hosts: localhost
  gather_facts: false
  tags: [graylog, inputs]

  vars:
    graylog_url: "http://172.16.0.60:9000/api"
    graylog_user: admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    graylog_headers:
      X-Requested-By: ansible
      Content-Type: application/json

    inputs:
      - title: "Syslog UDP — Network Devices"
        type: "org.graylog2.inputs.syslog.udp.SyslogUDPInput"
        global: true
        configuration:
          bind_address: "0.0.0.0"
          port: 514
          recv_buffer_size: 1048576
          override_source: ""
          force_rdns: false
          allow_override_date: true
          store_full_message: true
          expand_structured_data: false

      - title: "Syslog TCP — Infrastructure VMs"
        type: "org.graylog2.inputs.syslog.tcp.SyslogTCPInput"
        global: true
        configuration:
          bind_address: "0.0.0.0"
          port: 514
          recv_buffer_size: 1048576
          max_message_size: 65536
          override_source: ""
          force_rdns: false
          allow_override_date: true
          store_full_message: true
          expand_structured_data: false
          tcp_keepalive: true

      - title: "GELF UDP — Structured Logs"
        type: "org.graylog2.inputs.gelf.udp.GELFUDPInput"
        global: true
        configuration:
          bind_address: "0.0.0.0"
          port: 12201
          recv_buffer_size: 1048576
          decompress_size_limit: 8388608

      - title: "GELF TCP — Structured Logs"
        type: "org.graylog2.inputs.gelf.tcp.GELFTCPInput"
        global: true
        configuration:
          bind_address: "0.0.0.0"
          port: 12201
          recv_buffer_size: 1048576
          max_message_size: 2097152
          tcp_keepalive: true

  tasks:

    - name: "Inputs | Get existing inputs"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/inputs"
        method: GET
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
      register: existing_inputs

    - name: "Inputs | Build list of existing input titles"
      ansible.builtin.set_fact:
        existing_titles: >-
          {{ existing_inputs.json.inputs | map(attribute='title') | list }}

    - name: "Inputs | Create input: {{ item.title }}"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/inputs"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "{{ item.title }}"
          type: "{{ item.type }}"
          global: "{{ item.global }}"
          configuration: "{{ item.configuration }}"
          node: null
        status_code: 201
      loop: "{{ inputs }}"
      loop_control:
        label: "{{ item.title }}"
      when: item.title not in existing_titles
      register: input_results
      changed_when: true

    - name: "Inputs | Verify all inputs running"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/inputs"
        method: GET
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
      register: final_inputs

    - name: "Inputs | Report"
      ansible.builtin.debug:
        msg: "{{ item.title }} — {{ item.state }}"
      loop: "{{ final_inputs.json.inputs }}"
      loop_control:
        label: "{{ item.title }}"
EOF
```

### ### ℹ️ UI callout: Inputs in Graylog

```
After configure_inputs.yml runs, open the Graylog web UI:
http://172.16.0.60:9000  → System → Inputs

Each input shows:
  - State: RUNNING (green) or FAILED (red)
  - Messages received counter — starts at 0, increments as devices send logs
  - Throughput: real-time messages/second
  - "Show received messages" button — live view of incoming messages

The four inputs and what connects to each:
  Syslog UDP 514:  All 7 network devices → log to this input
  Syslog TCP 514:  6 infrastructure VMs → rsyslog from harden.yml
  GELF UDP 12201:  Future use (structured application logs)
  GELF TCP 12201:  Future use (structured application logs)
```

---

## 39.6 — Configure Syslog on Network Devices

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/configure_syslog.yml << 'EOF'
---
# Configure syslog on all network devices pointing to Graylog
# Usage: ansible-playbook graylog/configure_syslog.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

# ── Cisco IOS-XE ────────────────────────────────────────────────────
- name: "Syslog | Configure Cisco IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: [syslog, cisco_ios]

  vars:
    graylog_ip: "172.16.0.60"
    syslog_facility: local6
    syslog_severity: informational

  tasks:

    - name: "Syslog | Configure logging to Graylog"
      cisco.ios.ios_config:
        lines:
          # Set logging level — informational captures interface/BGP/config events
          - "logging {{ graylog_ip }}"
          - "logging trap {{ syslog_severity }}"
          - "logging facility {{ syslog_facility }}"
          - "logging on"
          # Critical: include timestamps and sequence numbers for correlation
          - "service timestamps log datetime msec show-timezone"
          - "service sequence-numbers"
          # Enable specific event categories relevant to network operations
          - "logging buffered 64000 informational"
          # BGP logging — needed for BGP state change messages
          - "router bgp {{ bgp_as | default('65001') }}"
          - "  bgp log-neighbor-changes"
        save_when: changed
      register: syslog_config

    - name: "Verify | Check logging config"
      cisco.ios.ios_command:
        commands:
          - show logging | include Logging to
      register: logging_verify

    - name: "Verify | Display logging config"
      ansible.builtin.debug:
        msg: "{{ logging_verify.stdout[0] }}"

# ── Cisco NX-OS ─────────────────────────────────────────────────────
- name: "Syslog | Configure Cisco NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tags: [syslog, cisco_nxos]

  vars:
    graylog_ip: "172.16.0.60"

  tasks:

    - name: "Syslog | Configure NX-OS logging"
      cisco.nxos.nxos_config:
        lines:
          - "logging server {{ graylog_ip }} 6 use-vrf management facility local6"
          - "logging timestamp milliseconds"
          - "logging monitor 6"
          - "logging level bgp 5"
          - "logging level ospf 5"
          - "logging level interface-vlan 5"
        save_when: changed

    - name: "Verify | Show NX-OS logging"
      cisco.nxos.nxos_command:
        commands: [show logging server]
      register: nxos_log_verify

    - name: "Verify | Display"
      ansible.builtin.debug:
        msg: "{{ nxos_log_verify.stdout[0] }}"

# ── Palo Alto PAN-OS ─────────────────────────────────────────────────
- name: "Syslog | Configure PAN-OS syslog"
  hosts: panos_devices
  gather_facts: false
  tags: [syslog, panos]

  vars:
    graylog_ip: "172.16.0.60"

  tasks:

    - name: "Syslog | Create syslog server profile"
      paloaltonetworks.panos.panos_config_element:
        provider: "{{ panos_provider }}"
        xpath: "/config/devices/entry[@name='localhost.localdomain']/deviceconfig/system/syslog/entry[@name='graylog']"
        element: |
          <server>
            <entry name="graylog-udp">
              <server>{{ graylog_ip }}</server>
              <transport>UDP</transport>
              <port>514</port>
              <format>BSD</format>
              <facility>LOG_LOCAL0</facility>
            </entry>
          </server>

    - name: "Syslog | Configure log forwarding for system logs"
      paloaltonetworks.panos.panos_config_element:
        provider: "{{ panos_provider }}"
        xpath: "/config/devices/entry[@name='localhost.localdomain']/deviceconfig/system"
        element: |
          <logging>
            <syslog>graylog</syslog>
          </logging>

    - name: "PAN-OS | Commit syslog changes"
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"

# ── Verify: Test log reception ───────────────────────────────────────
- name: "Verify | Confirm Graylog is receiving messages"
  hosts: localhost
  gather_facts: false
  tags: [syslog, verify]

  tasks:

    - name: "Graylog | Wait 30 seconds for first messages to arrive"
      ansible.builtin.pause:
        seconds: 30
        prompt: "Waiting for syslog messages from devices..."

    - name: "Graylog | Query recent messages"
      ansible.builtin.uri:
        url: >
          http://172.16.0.60:9000/api/search/universal/relative?query=*&range=60&limit=20&fields=source,message,facility,level
        method: GET
        user: admin
        password: "{{ vault_graylog_admin_password }}"
        force_basic_auth: true
        headers:
          X-Requested-By: ansible
          Accept: application/json
      register: recent_messages

    - name: "Graylog | Show message sources"
      ansible.builtin.debug:
        msg: >
          {{ recent_messages.json.messages |
             map(attribute='message') |
             map(attribute='source') |
             list | unique }}
EOF
```

---

## 39.7 — Streams

Streams route messages to named buckets based on field matching rules. Every message that hits the default stream also gets evaluated against all other stream rules — messages can match multiple streams.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/configure_streams.yml << 'EOF'
---
# Create Graylog streams for log routing
# Usage: ansible-playbook graylog/configure_streams.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Configure streams"
  hosts: localhost
  gather_facts: false
  tags: [graylog, streams]

  vars:
    graylog_url: "http://172.16.0.60:9000/api"
    graylog_user: admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    graylog_headers:
      X-Requested-By: ansible
      Content-Type: application/json

    streams:

      - title: "Network Devices"
        description: "All syslog messages from network infrastructure"
        matching_type: OR
        rules:
          - field: source
            value: "172.16.0.1[0-9]|172.16.0.2[0-9]|172.16.0.3[0-9]|172.16.0.5[0-9]"
            type: 2           # 2 = regex match
            inverted: false
        index_set_id: ""      # Will use default index set

      - title: "BGP Events"
        description: "BGP neighbor state changes and routing events"
        matching_type: OR
        rules:
          - field: message
            value: "BGP|bgp|neighbor.*[Dd]own|neighbor.*[Uu]p|ADJCHANGE"
            type: 2
            inverted: false

      - title: "Interface Events"
        description: "Interface state changes — up/down transitions"
        matching_type: OR
        rules:
          - field: message
            value: "UPDOWN|LINEPROTO|changed state to|link.*[Uu]p|link.*[Dd]own|Line protocol"
            type: 2
            inverted: false

      - title: "Configuration Changes"
        description: "Device configuration modifications"
        matching_type: OR
        rules:
          - field: message
            value: "CONFIG_I|SYS-5-CONFIG|Configured from|commit.*by|configure.*terminal"
            type: 2
            inverted: false

      - title: "Authentication Events"
        description: "Login, logout, and authentication failures"
        matching_type: OR
        rules:
          - field: message
            value: "LOGIN|LOGOUT|authentication.*fail|SSH.*accept|SSH.*close|AAA"
            type: 2
            inverted: false

      - title: "Infrastructure VMs"
        description: "Syslog from Proxmox guest VMs"
        matching_type: AND
        rules:
          - field: source
            value: "172.16.0.(2|3|4|5|6)[0-9]"
            type: 2
            inverted: false

  tasks:

    - name: "Index | Get default index set ID"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/indices/index_sets?stats=false"
        method: GET
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
      register: index_sets

    - name: "Index | Extract default index set ID"
      ansible.builtin.set_fact:
        default_index_set_id: >-
          {{ index_sets.json.index_sets |
             selectattr('default', 'equalto', true) |
             map(attribute='id') | first }}

    - name: "Streams | Get existing streams"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/streams"
        method: GET
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
      register: existing_streams

    - name: "Streams | Build existing stream title list"
      ansible.builtin.set_fact:
        existing_stream_titles: >-
          {{ existing_streams.json.streams | map(attribute='title') | list }}

    - name: "Streams | Create stream: {{ item.title }}"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/streams"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "{{ item.title }}"
          description: "{{ item.description }}"
          matching_type: "{{ item.matching_type }}"
          rules: "{{ item.rules }}"
          index_set_id: "{{ default_index_set_id }}"
          remove_matches_from_default_stream: false
        status_code: 201
      loop: "{{ streams }}"
      loop_control:
        label: "{{ item.title }}"
      when: item.title not in existing_stream_titles
      register: stream_results
      changed_when: true

    - name: "Streams | Resume (start) all created streams"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/streams/{{ item.json.stream_id }}/resume"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        status_code: 204
      loop: "{{ stream_results.results | selectattr('changed') | list }}"
      loop_control:
        label: "stream {{ item.json.stream_id | default('') }}"
      when: item.changed | default(false)
EOF
```

---

## 39.8 — Pipeline Rules: Parsing and Enrichment

Pipelines process messages after they're received. Each pipeline has one or more stages, and each stage has rules. Rules use Graylog's processing language to parse fields, set values, and call functions.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/configure_pipelines.yml << 'EOF'
---
# Create Graylog pipelines for log parsing and Netbox enrichment
# Usage: ansible-playbook graylog/configure_pipelines.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Configure processing pipelines"
  hosts: localhost
  gather_facts: false
  tags: [graylog, pipelines]

  vars:
    graylog_url: "http://172.16.0.60:9000/api"
    graylog_user: admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    graylog_headers:
      X-Requested-By: ansible
      Content-Type: application/json
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"

  tasks:

    # ── Pipeline rules ────────────────────────────────────────────────
    # Rules are created first, then assembled into a pipeline

    - name: "Rules | Create Cisco IOS syslog parser rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Parse Cisco IOS Syslog"
          description: >
            Extracts facility, severity, mnemonic, and message body
            from standard Cisco IOS syslog format:
            %FACILITY-SEVERITY-MNEMONIC: message body
          source: |
            rule "Parse Cisco IOS Syslog"
            when
              has_field("message") AND
              contains(to_string($message.message), "%")
            then
              // Cisco syslog format: %FACILITY-SEVERITY-MNEMONIC: description
              // Example: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1, changed state to down
              let raw = to_string($message.message);
              let match = grok("%{CISCO_SYSLOG}", raw, true);

              if (!is_null(match["CISCO_FACILITY"])) {
                set_field("cisco_facility", match["CISCO_FACILITY"]);
              }
              if (!is_null(match["CISCO_SEVERITY"])) {
                set_field("cisco_severity_code", to_long(match["CISCO_SEVERITY"]));
                // Map severity code to human-readable name
                let sev_map = create_map(["0","emergency","1","alert","2","critical",
                                          "3","error","4","warning","5","notice",
                                          "6","informational","7","debugging"]);
                set_field("cisco_severity_name",
                  key_value(sev_map, to_string(match["CISCO_SEVERITY"])));
              }
              if (!is_null(match["CISCO_MNEMONIC"])) {
                set_field("cisco_mnemonic", match["CISCO_MNEMONIC"]);
              }
              if (!is_null(match["CISCO_MSG"])) {
                set_field("cisco_message_body", match["CISCO_MSG"]);
              }
              // Tag for stream routing
              set_field("platform", "cisco_ios");
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create NX-OS syslog parser rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Parse Cisco NX-OS Syslog"
          description: "Extracts structured fields from NX-OS syslog messages"
          source: |
            rule "Parse Cisco NX-OS Syslog"
            when
              has_field("message") AND
              (contains(to_string($message.message), "%") OR
               contains(to_string($message.message), "NXOS"))
            then
              let raw = to_string($message.message);
              // NX-OS format: YYYY MMM DD HH:MM:SS.sss %FACILITY-SEVERITY-MNEMONIC: message
              let match = grok("%{YEAR}.*?%%{WORD:nxos_facility}-%{INT:nxos_severity}-%{WORD:nxos_mnemonic}:%{GREEDYDATA:nxos_body}", raw, true);

              if (!is_null(match["nxos_facility"])) {
                set_field("cisco_facility",  match["nxos_facility"]);
                set_field("cisco_mnemonic",  match["nxos_mnemonic"]);
                set_field("cisco_severity_code", to_long(match["nxos_severity"]));
                set_field("cisco_message_body", match["nxos_body"]);
              }
              set_field("platform", "cisco_nxos");
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create PAN-OS syslog parser rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Parse PAN-OS Syslog"
          description: "Parses PAN-OS comma-delimited syslog format for traffic, threat, and system logs"
          source: |
            rule "Parse PAN-OS Syslog"
            when
              has_field("source") AND
              to_string($message.source) == "172.16.0.51"
            then
              let raw = to_string($message.message);
              // PAN-OS CSV format fields differ by log type
              // System log: FUTURE_USE,Receive Time,Serial,Type,Subtype,...,EventID,Object,Description
              let fields = split(",", raw);

              if (length(fields) > 3) {
                set_field("panos_log_type", fields[3]);
                set_field("panos_subtype",  fields[4]);
              }
              if (length(fields) > 5) {
                set_field("panos_time",     fields[1]);
                set_field("panos_serial",   fields[2]);
              }
              // Tag events of interest
              if (contains(to_string($message.message), "link-change") OR
                  contains(to_string($message.message), "intf-status")) {
                set_field("event_type", "interface_change");
              }
              set_field("platform", "panos");
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create interface down detector rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Detect Interface Down Events"
          description: "Adds event_type=interface_down tag to interface-down syslog messages"
          source: |
            rule "Detect Interface Down Events"
            when
              has_field("message") AND
              (contains(to_string($message.message), "changed state to down") OR
               contains(to_string($message.message), "UPDOWN") AND
               contains(to_string($message.message), "down") OR
               contains(to_string($message.message), "link-status: change to down"))
            then
              set_field("event_type", "interface_down");
              set_field("alert_category", "network");
              // Extract interface name using regex
              let msg = to_string($message.message);
              let intf_match = regex("Interface\\s+(\\S+),", msg);
              if (!is_null(intf_match["0"])) {
                set_field("interface_name", intf_match["0"]);
              }
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create BGP down detector rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Detect BGP Neighbor Down"
          description: "Adds event_type=bgp_neighbor_down and extracts peer IP"
          source: |
            rule "Detect BGP Neighbor Down"
            when
              has_field("message") AND
              (contains(to_string($message.message), "ADJCHANGE") OR
               contains(to_string($message.message), "BGP") OR
               contains(to_string($message.message), "bgp")) AND
              (contains(to_string($message.message), "Down") OR
               contains(to_string($message.message), "down") OR
               contains(to_string($message.message), "went down"))
            then
              set_field("event_type", "bgp_neighbor_down");
              set_field("alert_category", "network");
              // Extract peer IP address
              let msg = to_string($message.message);
              let ip_match = regex("neighbor\\s+([0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+)", msg);
              if (!is_null(ip_match["0"])) {
                set_field("bgp_peer_ip", ip_match["0"]);
              }
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create config change detector rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Detect Configuration Changes"
          description: "Tags configuration change events and extracts user who made the change"
          source: |
            rule "Detect Configuration Changes"
            when
              has_field("message") AND
              (contains(to_string($message.message), "SYS-5-CONFIG_I") OR
               contains(to_string($message.message), "Configured from") OR
               contains(to_string($message.message), "commit.*by") OR
               contains(to_string($message.message), "CONFIG_I"))
            then
              set_field("event_type", "config_change");
              set_field("alert_category", "change_management");
              // Extract configuring user/source
              let msg = to_string($message.message);
              let user_match = regex("by\\s+(\\S+)\\s+on", msg);
              if (!is_null(user_match["0"])) {
                set_field("config_changed_by", user_match["0"]);
              }
              let console_match = regex("from\\s+(console|vty\\d+|ssh)", msg);
              if (!is_null(console_match["0"])) {
                set_field("config_source", console_match["0"]);
              }
            end
        status_code: [200, 400]
      changed_when: false

    - name: "Rules | Create Netbox IP enrichment rule"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/rule"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Enrich Source IP from Netbox Lookup Table"
          description: >
            Resolves source IP to device name, role, and platform
            using the Netbox lookup table. Adds device_name,
            device_role, device_platform fields.
          source: |
            rule "Enrich Source IP from Netbox"
            when
              has_field("source")
            then
              // Look up the source IP in the Netbox lookup table
              let lookup_result = lookup("netbox-devices", to_string($message.source));
              if (!is_null(lookup_result)) {
                set_field("device_name",     lookup_result["name"]);
                set_field("device_role",     lookup_result["role"]);
                set_field("device_platform", lookup_result["platform"]);
                set_field("device_site",     lookup_result["site"]);
              }
            end
        status_code: [200, 400]
      changed_when: false

    # ── Create the pipeline ───────────────────────────────────────────
    - name: "Pipeline | Create Network Device Processing pipeline"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/pipeline"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Network Device Processing"
          description: >
            Processes syslog from network devices.
            Stage 0: Platform detection and field parsing.
            Stage 1: Event classification (interface, BGP, config).
            Stage 2: Netbox IP enrichment.
          source: |
            pipeline "Network Device Processing"
            stage 0 match either
              rule "Parse Cisco IOS Syslog"
              rule "Parse Cisco NX-OS Syslog"
              rule "Parse PAN-OS Syslog"
            stage 1 match either
              rule "Detect Interface Down Events"
              rule "Detect BGP Neighbor Down"
              rule "Detect Configuration Changes"
            stage 2 match all
              rule "Enrich Source IP from Netbox"
            end
        status_code: [200, 400]
      register: pipeline_result
      changed_when: false

    - name: "Pipeline | Connect pipeline to Network Devices stream"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/pipelines/connections/to_stream"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          stream_id: "{{ stream_id_network_devices }}"   # Set by a lookup task
          pipeline_ids: ["{{ pipeline_result.json.id | default('') }}"]
        status_code: [200, 400]
      changed_when: false
EOF
```

---

## 39.9 — Netbox Lookup Table

The lookup table is what transforms a source IP like `172.16.0.11` into structured fields like `device_name: wan-r1`, `device_role: wan-router`, `device_platform: ios-xe`. It's populated by an Ansible playbook that queries Netbox and uploads the result to Graylog.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/netbox_lookup.yml << 'EOF'
---
# Build and upload Netbox device lookup table to Graylog
# Run on demand or after device changes in Netbox
# Usage: ansible-playbook graylog/netbox_lookup.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Create Netbox IP lookup table"
  hosts: localhost
  gather_facts: false
  tags: [graylog, lookup, netbox]

  vars:
    graylog_url: "http://172.16.0.60:9000/api"
    graylog_user: admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    graylog_headers:
      X-Requested-By: ansible
      Content-Type: application/json
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"

  tasks:

    - name: "Netbox | Fetch all devices with IPs"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/dcim/devices/?limit=200&has_primary_ip=true"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: netbox_devices

    - name: "Lookup | Build IP → device mapping"
      ansible.builtin.set_fact:
        device_map: >-
          {{ dict(
               netbox_devices.json.results |
               selectattr('primary_ip', 'ne', none) |
               map(attribute='primary_ip.address') |
               map('regex_replace', '/[0-9]+$', '') |
               zip(
                 netbox_devices.json.results |
                 selectattr('primary_ip', 'ne', none) |
                 map('combine', {}) |
                 list
               )
             )
          }}

    - name: "Lookup | Display device map summary"
      ansible.builtin.debug:
        msg: "Built lookup table with {{ device_map | length }} device entries"

    # ── Create CSV data adapter ───────────────────────────────────────
    # Graylog lookup tables use data adapters — CSV file is the simplest
    - name: "Lookup | Write CSV file on logging VM"
      ansible.builtin.template:
        src: netbox_lookup.csv.j2
        dest: /opt/logging/graylog/data/netbox_devices.csv
        mode: '0644'
      delegate_to: logging

    - name: "Lookup | Create CSV data adapter in Graylog"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/lookup/adapters"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          name: "netbox-device-csv"
          title: "Netbox Device CSV"
          description: "IP to device name/role/platform lookup from Netbox"
          config:
            type: "csvfile"
            path: "/usr/share/graylog/data/netbox_devices.csv"
            separator: ","
            quotechar: '"'
            key_column: "ip_address"
            check_interval: 60
        status_code: [200, 400]
      changed_when: false
      register: adapter_result

    - name: "Lookup | Create lookup cache"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/lookup/caches"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          name: "netbox-cache"
          title: "Netbox Device Cache"
          description: "Cache for Netbox device lookups"
          config:
            type: "guava_cache"
            max_size: 1000
            expire_after_access: 3600
            expire_after_write: 3600
        status_code: [200, 400]
      changed_when: false
      register: cache_result

    - name: "Lookup | Create lookup table"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/system/lookup/tables"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          name: "netbox-devices"
          title: "Netbox Devices"
          description: "Resolves source IP to device metadata from Netbox"
          cache_id: "{{ cache_result.json.id | default('') }}"
          data_adapter_id: "{{ adapter_result.json.id | default('') }}"
          default_single_value: ""
          default_single_value_type: "NULL"
          default_multi_value: ""
          default_multi_value_type: "NULL"
        status_code: [200, 400]
      changed_when: false
EOF
```

### CSV template for lookup table

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/templates/netbox_lookup.csv.j2 << 'EOF'
ip_address,name,role,platform,site
{% for ip, device in device_map.items() %}
{{ ip }},{{ device.name }},{{ device.device_role.slug | default('unknown') }},{{ device.platform.slug | default('unknown') }},{{ device.site.slug | default('lab') }}
{% endfor %}
EOF
```

---

## 39.10 — Graylog Alerts

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/graylog/configure_alerts.yml << 'EOF'
---
# Configure Graylog alert conditions and notification actions
# Usage: ansible-playbook graylog/configure_alerts.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Graylog | Configure alerts"
  hosts: localhost
  gather_facts: false
  tags: [graylog, alerts]

  vars:
    graylog_url: "http://172.16.0.60:9000/api"
    graylog_user: admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    graylog_headers:
      X-Requested-By: ansible
      Content-Type: application/json

  tasks:

    - name: "Notifications | Create Slack notification"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/events/notifications"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Slack — Network Alerts"
          description: "Send network event alerts to #network-alerts Slack channel"
          config:
            type: "http-notification-v1"
            url: "{{ vault_slack_webhook_url | default('https://hooks.slack.com/PLACEHOLDER') }}"
            method: "POST"
            content_type: "application/json"
            body_template: |
              {
                "text": "*[Graylog Alert]* ${event_definition_title}",
                "attachments": [{
                  "color": "danger",
                  "fields": [
                    {"title": "Device", "value": "${source}", "short": true},
                    {"title": "Event Type", "value": "${event.fields.event_type}", "short": true},
                    {"title": "Message", "value": "${event.message}", "short": false},
                    {"title": "Time", "value": "${event.timestamp}", "short": true}
                  ]
                }]
              }
        status_code: [200, 400]
      register: slack_notification
      changed_when: false

    # ── Alert: Interface Down ─────────────────────────────────────────
    - name: "Alert | Create interface down event definition"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/events/definitions"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Network Interface Down"
          description: "Fires when a network device logs an interface down event"
          priority: 3
          alert: true
          config:
            type: "aggregation-v1"
            query: 'event_type:interface_down'
            query_parameters: []
            streams: []
            group_by: [source]
            series:
              - id: "count-of-events"
                function: count
                field: ""
            conditions:
              expression:
                expr: ">"
                left:
                  expr: "number-ref"
                  ref: "count-of-events"
                right:
                  expr: "number"
                  value: 0
            execute_every_ms: 60000     # Evaluate every minute
            search_within_ms: 120000    # Look back 2 minutes
          field_spec:
            event_type:
              data_type: string
              providers:
                - type: template-v1
                  template: "${source.event_type}"
                  require_values: false
            device_name:
              data_type: string
              providers:
                - type: template-v1
                  template: "${source.device_name}"
                  require_values: false
          key_spec: [source]
          notification_settings:
            grace_period_ms: 300000    # 5-minute grace period between repeated alerts
            backlog_size: 5
          notifications:
            - notification_id: "{{ slack_notification.json.id | default('') }}"
        status_code: [200, 400]
      changed_when: false

    # ── Alert: BGP Neighbor Down ──────────────────────────────────────
    - name: "Alert | Create BGP neighbor down event definition"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/events/definitions"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "BGP Neighbor Down"
          description: "Fires when a BGP neighbor adjacency drops on any device"
          priority: 4     # Higher priority than interface down
          alert: true
          config:
            type: "aggregation-v1"
            query: 'event_type:bgp_neighbor_down'
            group_by: [source, bgp_peer_ip]
            series:
              - id: "count-bgp-events"
                function: count
                field: ""
            conditions:
              expression:
                expr: ">"
                left:
                  expr: "number-ref"
                  ref: "count-bgp-events"
                right:
                  expr: "number"
                  value: 0
            execute_every_ms: 60000
            search_within_ms: 180000    # 3-minute lookback (matches Zabbix Part 37)
          key_spec: [source, bgp_peer_ip]
          notification_settings:
            grace_period_ms: 300000
            backlog_size: 10
          notifications:
            - notification_id: "{{ slack_notification.json.id | default('') }}"
        status_code: [200, 400]
      changed_when: false

    # ── Alert: Configuration Change ───────────────────────────────────
    - name: "Alert | Create config change event definition"
      ansible.builtin.uri:
        url: "{{ graylog_url }}/events/definitions"
        method: POST
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers: "{{ graylog_headers }}"
        body_format: json
        body:
          title: "Device Configuration Changed"
          description: >
            Fires when any device logs a configuration change event.
            Critical for change management — unexpected config changes
            are a security and stability concern.
          priority: 3
          alert: true
          config:
            type: "aggregation-v1"
            query: 'event_type:config_change'
            group_by: [source, config_changed_by]
            series:
              - id: "count-config-events"
                function: count
                field: ""
            conditions:
              expression:
                expr: ">"
                left:
                  expr: "number-ref"
                  ref: "count-config-events"
                right:
                  expr: "number"
                  value: 0
            execute_every_ms: 60000
            search_within_ms: 120000
          key_spec: [source]
          notification_settings:
            grace_period_ms: 60000     # 1-minute grace — config changes are important
            backlog_size: 5
          notifications:
            - notification_id: "{{ slack_notification.json.id | default('') }}"
        status_code: [200, 400]
      changed_when: false

    - name: "Alerts | Report configuration complete"
      ansible.builtin.debug:
        msg:
          - "Three alert event definitions created:"
          - "  - Network Interface Down (1-minute evaluation)"
          - "  - BGP Neighbor Down (3-minute lookback)"
          - "  - Device Configuration Changed (1-minute grace period)"
          - "Notifications: Slack webhook (configure vault_slack_webhook_url)"
          - "View alerts: http://172.16.0.60:9000/alerts"
EOF
```

---

## 39.11 — Checklist

```
[ ] System prerequisites:
    [ ] vm.max_map_count=262144 set in /etc/sysctl.conf
    [ ] sysctl vm.max_map_count returns 262144 after reboot
    [ ] Docker and docker-compose-plugin installed
    [ ] /opt/logging/{opensearch,graylog/{data,journal},mongodb} created

[ ] Stack deployed:
    [ ] ~/logging/docker-compose.yml created with correct image versions
    [ ] ~/logging/.env created with generated secrets
    [ ] Secrets added to Ansible vault:
        vault_graylog_password_secret
        vault_graylog_root_password_sha2
        vault_graylog_admin_password
        vault_mongodb_root_password
    [ ] docker compose ps — all 3 containers running:
        [ ] graylog-mongodb  — healthy
        [ ] graylog-opensearch — healthy
        [ ] graylog            — healthy
    [ ] curl http://172.16.0.60:9000/api/system/lbstatus returns ALIVE
    [ ] Web UI at http://172.16.0.60:9000 loads and admin login works

[ ] Inputs configured:
    [ ] configure_inputs.yml ran successfully
    [ ] Graylog UI System → Inputs shows 4 inputs, all RUNNING:
        [ ] Syslog UDP 514
        [ ] Syslog TCP 514
        [ ] GELF UDP 12201
        [ ] GELF TCP 12201

[ ] Syslog from network devices:
    [ ] configure_syslog.yml ran against all devices
    [ ] IOS-XE devices (wan-r1, wan-r2, dist-01, leaf-01/02):
        [ ] "logging 172.16.0.60" in running config
        [ ] "service timestamps log datetime msec" enabled
        [ ] "bgp log-neighbor-changes" enabled
    [ ] NX-OS (spine-01): logging server configured with use-vrf management
    [ ] PAN-OS (panos-fw01): syslog server profile committed
    [ ] Graylog Inputs show rising message count after ~30 seconds
    [ ] Search query: source:172.16.0.11 returns wan-r1 messages

[ ] Streams configured:
    [ ] configure_streams.yml ran successfully
    [ ] 6 streams visible in Streams menu, all RUNNING:
        [ ] Network Devices
        [ ] BGP Events
        [ ] Interface Events
        [ ] Configuration Changes
        [ ] Authentication Events
        [ ] Infrastructure VMs
    [ ] Test: shut interface on wan-r1 → message appears in Interface Events stream
    [ ] Test: restore interface → recovery message appears

[ ] Pipelines configured:
    [ ] configure_pipelines.yml ran successfully
    [ ] Pipeline "Network Device Processing" visible in System → Pipelines
    [ ] 6 rules created:
        [ ] Parse Cisco IOS Syslog
        [ ] Parse Cisco NX-OS Syslog
        [ ] Parse PAN-OS Syslog
        [ ] Detect Interface Down Events
        [ ] Detect BGP Neighbor Down
        [ ] Detect Configuration Changes
        [ ] Enrich Source IP from Netbox
    [ ] Test: message from wan-r1 shows cisco_facility, cisco_mnemonic fields
    [ ] Test: interface down message shows event_type=interface_down field
    [ ] Pipeline connected to Network Devices stream

[ ] Netbox lookup table:
    [ ] netbox_lookup.yml ran successfully
    [ ] /opt/logging/graylog/data/netbox_devices.csv exists with correct entries
    [ ] CSV contains correct IP, name, role, platform columns
    [ ] Lookup table "netbox-devices" visible in System → Lookup Tables
    [ ] Test: message from 172.16.0.11 shows device_name=wan-r1,
        device_role=wan-router, device_platform=ios-xe fields

[ ] Alerts configured:
    [ ] configure_alerts.yml ran successfully
    [ ] 3 event definitions visible in Alerts → Event Definitions:
        [ ] Network Interface Down
        [ ] BGP Neighbor Down
        [ ] Device Configuration Changed
    [ ] Slack notification configured
    [ ] Test end-to-end:
        [ ] Shut interface on wan-r1
        [ ] Message appears in Interface Events stream within 30s
        [ ] event_type=interface_down field present
        [ ] "Network Interface Down" alert fires within 1-2 minutes
        [ ] Slack message received (if webhook configured)
        [ ] Restore interface → alert resolves

[ ] UFW rules verified:
    [ ] Port 9000 TCP from 172.16.0.0/24 (web UI and API)
    [ ] Port 514 UDP from any (syslog from devices)
    [ ] Port 514 TCP from 192.168.100.0/24 (syslog from VMs)
    [ ] Port 12201 UDP/TCP from 192.168.100.0/24 (GELF)
    [ ] Port 162 UDP from any (SNMP traps)
    [ ] Port 9200 NOT accessible externally (OpenSearch internal only)

[ ] Snapshot taken:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=106 snap_name=post-graylog \
          snap_description='Graylog deployed, pipelines configured, alerts active Part 39'"

[ ] All playbooks committed to Gitea:
    [ ] playbooks/infrastructure/graylog/deploy.yml
    [ ] playbooks/infrastructure/graylog/configure_inputs.yml
    [ ] playbooks/infrastructure/graylog/configure_syslog.yml
    [ ] playbooks/infrastructure/graylog/configure_streams.yml
    [ ] playbooks/infrastructure/graylog/configure_pipelines.yml
    [ ] playbooks/infrastructure/graylog/netbox_lookup.yml
    [ ] playbooks/infrastructure/graylog/configure_alerts.yml
    [ ] templates/netbox_lookup.csv.j2
```

---

*Every log line from every device lands in Graylog, gets parsed into structured fields, enriched with device metadata from Netbox, routed into the correct stream, and evaluated against alert conditions that fire within a minute of the triggering event. Cisco IOS syslog messages arrive with `cisco_facility`, `cisco_mnemonic`, and `event_type` fields already extracted — no manual searching through raw message text. An interface down event on wan-r1 generates a structured log entry saying `event_type=interface_down device_name=wan-r1 device_role=wan-router interface_name=GigabitEthernet2` before alerting. Part 40 closes the loop — unifying Zabbix, Prometheus, Graylog, and Netbox into a single operational picture.*

*Next up: **Part 40 — Integrating the Stack (Closing the Loop)***
