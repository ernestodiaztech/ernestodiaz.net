---
draft: true
title: '38 - Grafana'
description: "Part 38 of my Ansible learning geared towards Network Engineering."
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

## Part 38: Modern Observability — Prometheus + Grafana + Loki

> *Part 37 gave us Zabbix — polling-based, SNMP-native, the standard in enterprise network operations. This part builds the modern observability stack alongside it. The word "observability" is precise: it means being able to ask arbitrary questions about system state without having predicted in advance which questions you'd need to ask. Zabbix answers the questions its templates were built for. Prometheus answers any question you can express in PromQL — including ones you didn't know you'd need until an outage at 2am. This part deploys Prometheus, Grafana, Loki, and AlertManager on the observability VM, installs exporters on every VM as native binaries managed by Ansible, and builds dashboards that show the lab fabric from perspectives Zabbix can't express.*

---

## 38.1 — How Prometheus Works (Conceptual Foundation)

### The pull model and why it matters

```
Prometheus scrape cycle:

  Every scrape_interval (default 15s):
  Prometheus ──HTTP GET http://target:port/metrics──► Exporter
                                                       Returns all
                                                       metrics as
                                                       text/plain

  Prometheus parses the response, timestamps each sample,
  stores in its local time-series database (TSDB).

  Key insight: If Prometheus can't reach a target, it records
  that as a scrape failure — itself a metric. The absence of
  data IS data in Prometheus. Zabbix needs a trigger to know
  a device is unreachable. Prometheus surfaces it automatically
  via up{job="network-devices"} == 0.

Prometheus data model:
  Every metric is a time series identified by:
    metric_name{label1="value1", label2="value2"} value timestamp

  Examples:
    node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
    ifOperStatus{ifDescr="GigabitEthernet1",instance="172.16.0.11"} 1
    up{job="snmp",instance="172.16.0.11"} 1

  Labels are the power — they let you slice and aggregate
  across any dimension without pre-defining the queries.
  "Show me all interfaces that are down across all devices"
  is a single PromQL expression, not a template per device.
```

### Prometheus vs Zabbix at a glance

```
                    Zabbix              Prometheus
─────────────────────────────────────────────────────────
Data collection     Poll (push/pull)    Pull (scrape)
Protocol            SNMP, agent, HTTP   HTTP /metrics
Network device      Native SNMP         SNMP Exporter (proxy)
Query language      None (UI only)      PromQL
Retention           Configurable DB     Local TSDB (default 15d)
Alerting            Triggers + Actions  AlertManager
Dashboards          Built-in            Grafana
Best for            SNMP/agent polling  Custom metrics + PromQL
When to use both    Always — they complement each other
```

### The SNMP Exporter bridge

```
Network devices don't expose HTTP /metrics endpoints.
The Prometheus SNMP Exporter solves this:

  Prometheus ──HTTP GET /snmp?target=172.16.0.11&module=cisco_ios──►
    SNMP Exporter (runs on observability VM)
      ──SNMP GET──► wan-r1 (172.16.0.11)
        ◄── SNMP response ──
    Translates SNMP OIDs → Prometheus metrics format
  ◄── HTTP 200 with /metrics text ──

  The SNMP Exporter is a translation layer. One instance
  on the observability VM polls all network devices.
  No software on the network devices themselves.
```

---

## 38.2 — Architecture

```
observability VM (172.16.0.50 / 192.168.100.50)
│
├── Docker: prometheus      — TSDB + scrape engine (port 9090)
├── Docker: grafana         — Dashboards + alerting UI (port 3000)
├── Docker: loki            — Log aggregation (port 3100)
├── Docker: alertmanager    — Alert routing (port 9093)
└── Docker: snmp-exporter   — SNMP → Prometheus bridge (port 9116)

Native binaries on each VM (managed by Ansible):
  network-lab    192.168.100.10  — node_exporter :9100
  netbox         192.168.100.20  — node_exporter :9100
  automation     192.168.100.30  — node_exporter :9100
  monitoring     192.168.100.40  — node_exporter :9100
  observability  192.168.100.50  — node_exporter :9100
  logging        192.168.100.60  — node_exporter :9100

  All VMs also run:
  promtail (systemd service) — tails /var/log/* → Loki :3100

Scrape flows:
  prometheus ──scrape :9100──► node_exporter on each VM (direct)
  prometheus ──scrape :9116?target=172.16.0.11──► snmp_exporter
                                  ──SNMP──► wan-r1
  prometheus ──scrape :9116?target=172.16.0.12──► snmp_exporter
                                  ──SNMP──► wan-r2
  ... (all 7 network devices)

Log flows:
  promtail on each VM ──push──► loki :3100

Access:
  Grafana:      http://172.16.0.50:3000
  Prometheus:   http://172.16.0.50:9090
  AlertManager: http://172.16.0.50:9093
  Loki API:     http://172.16.0.50:3100
```

---

## 38.3 — Deploy the Observability Stack

### Directory structure

```bash
# SSH to observability VM
ssh ansible@172.16.0.50

# Install Docker
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker ansible
newgrp docker

# Create directory structure
sudo mkdir -p /opt/observability/{prometheus,grafana,loki,alertmanager}
sudo mkdir -p /opt/observability/prometheus/rules
sudo mkdir -p /opt/observability/grafana/{dashboards,provisioning/datasources,provisioning/dashboards}
sudo chown -R ansible:ansible /opt/observability
```

### Docker Compose file

```bash
mkdir -p ~/observability
cat > ~/observability/docker-compose.yml << 'EOF'
---
version: "3.8"

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'          # Enables POST /-/reload
      - '--web.enable-admin-api'
      - '--log.level=info'
    volumes:
      - /opt/observability/prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - observability
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9090/-/healthy"]
      interval: 15s
      timeout: 5s
      retries: 5

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: "${GRAFANA_ADMIN_PASSWORD}"
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_DOMAIN: "172.16.0.50"
      GF_SERVER_ROOT_URL: "http://172.16.0.50:3000"
      GF_ALERTING_ENABLED: "true"
      GF_UNIFIED_ALERTING_ENABLED: "true"
    volumes:
      - grafana-data:/var/lib/grafana
      - /opt/observability/grafana/provisioning:/etc/grafana/provisioning
      - /opt/observability/grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    networks:
      - observability
    depends_on:
      - prometheus
      - loki

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    command: -config.file=/etc/loki/loki-config.yml
    volumes:
      - /opt/observability/loki:/etc/loki
      - loki-data:/loki
    ports:
      - "3100:3100"
    networks:
      - observability
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3100/ready"]
      interval: 15s
      timeout: 5s
      retries: 5

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.listen-address=:9093'
    volumes:
      - /opt/observability/alertmanager:/etc/alertmanager
      - alertmanager-data:/alertmanager
    ports:
      - "9093:9093"
    networks:
      - observability

  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    restart: unless-stopped
    command:
      - '--config.file=/etc/snmp-exporter/snmp.yml'
    volumes:
      - /opt/observability/snmp-exporter:/etc/snmp-exporter
    ports:
      - "9116:9116"
    networks:
      - observability

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  alertmanager-data:

networks:
  observability:
    driver: bridge
EOF
```

### Configuration files

```bash
# Loki config
mkdir -p /opt/observability/loki
cat > /opt/observability/loki/loki-config.yml << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
EOF

# AlertManager config
mkdir -p /opt/observability/alertmanager
cat > /opt/observability/alertmanager/alertmanager.yml << 'EOF'
# Managed by Ansible — regenerated by deploy playbook
global:
  resolve_timeout: 5m
  slack_api_url: '${SLACK_WEBHOOK_URL}'

route:
  group_by: ['alertname', 'job', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: critical-alerts
      repeat_interval: 1h
    - match:
        severity: warning
      receiver: default
      repeat_interval: 4h

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#network-alerts'
        title: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Host:* {{ .Labels.instance }}
          *Alert:* {{ .Labels.alertname }}
          *Severity:* {{ .Labels.severity }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
        send_resolved: true

  - name: 'critical-alerts'
    slack_configs:
      - channel: '#network-critical'
        title: '🚨 CRITICAL: {{ .CommonLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Host:* {{ .Labels.instance }}
          *Alert:* {{ .Labels.alertname }}
          *Description:* {{ .Annotations.description }}
          {{ end }}
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['alertname', 'instance']
EOF

# Environment file
GRAFANA_ADMIN_PASSWORD=$(openssl rand -base64 16)
cat > ~/observability/.env << EOF
GRAFANA_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
SLACK_WEBHOOK_URL=${SLACK_WEBHOOK_URL:-https://hooks.slack.com/services/placeholder}
EOF
chmod 600 ~/observability/.env
echo "Add to vault: vault_grafana_admin_password: ${GRAFANA_ADMIN_PASSWORD}"
```

---

## 38.4 — Ansible Playbook: Deploy Observability Stack

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/observability/templates

cat > ~/projects/ansible-network/playbooks/infrastructure/observability/deploy.yml << 'EOF'
---
# Deploy Prometheus + Grafana + Loki + AlertManager + SNMP Exporter
# Usage: ansible-playbook playbooks/infrastructure/observability/deploy.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Observability | Deploy stack on observability VM"
  hosts: observability_hosts
  gather_facts: true
  become: true
  tags: [observability, deploy]

  vars:
    obs_dir: /home/ansible/observability
    obs_data_dir: /opt/observability

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

    - name: "Dirs | Create observability directories"
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
      loop:
        - "{{ obs_dir }}"
        - "{{ obs_data_dir }}/prometheus/rules"
        - "{{ obs_data_dir }}/grafana/provisioning/datasources"
        - "{{ obs_data_dir }}/grafana/provisioning/dashboards"
        - "{{ obs_data_dir }}/grafana/dashboards"
        - "{{ obs_data_dir }}/loki"
        - "{{ obs_data_dir }}/alertmanager"
        - "{{ obs_data_dir }}/snmp-exporter"

    # ── Generate prometheus.yml from Ansible inventory ────────────────
    - name: "Prometheus | Generate prometheus.yml from inventory"
      ansible.builtin.template:
        src: prometheus.yml.j2
        dest: "{{ obs_data_dir }}/prometheus/prometheus.yml"
        owner: ansible
        mode: '0644'
      notify: Reload Prometheus

    # ── Alert rules ───────────────────────────────────────────────────
    - name: "Prometheus | Deploy alert rules"
      ansible.builtin.template:
        src: "{{ item }}"
        dest: "{{ obs_data_dir }}/prometheus/rules/{{ item | basename | replace('.j2', '') }}"
        owner: ansible
        mode: '0644'
      loop:
        - templates/rules/network_alerts.yml.j2
        - templates/rules/vm_alerts.yml.j2
      notify: Reload Prometheus

    # ── Loki config ───────────────────────────────────────────────────
    - name: "Loki | Deploy config"
      ansible.builtin.template:
        src: loki-config.yml.j2
        dest: "{{ obs_data_dir }}/loki/loki-config.yml"
        owner: ansible
        mode: '0644'

    # ── AlertManager config ───────────────────────────────────────────
    - name: "AlertManager | Deploy config"
      ansible.builtin.template:
        src: alertmanager.yml.j2
        dest: "{{ obs_data_dir }}/alertmanager/alertmanager.yml"
        owner: ansible
        mode: '0644'

    # ── SNMP Exporter config ──────────────────────────────────────────
    - name: "SNMP Exporter | Deploy snmp.yml"
      ansible.builtin.copy:
        src: files/snmp.yml
        dest: "{{ obs_data_dir }}/snmp-exporter/snmp.yml"
        owner: ansible
        mode: '0644'

    # ── Grafana provisioning ──────────────────────────────────────────
    - name: "Grafana | Deploy datasource provisioning"
      ansible.builtin.template:
        src: grafana-datasources.yml.j2
        dest: "{{ obs_data_dir }}/grafana/provisioning/datasources/datasources.yml"
        owner: ansible
        mode: '0644'

    - name: "Grafana | Deploy dashboard provisioning"
      ansible.builtin.template:
        src: grafana-dashboards-provisioning.yml.j2
        dest: "{{ obs_data_dir }}/grafana/provisioning/dashboards/dashboards.yml"
        owner: ansible
        mode: '0644'

    # ── UFW rules ─────────────────────────────────────────────────────
    - name: "UFW | Allow Grafana"
      community.general.ufw:
        rule: allow
        port: "3000"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Grafana web UI"

    - name: "UFW | Allow Prometheus"
      community.general.ufw:
        rule: allow
        port: "9090"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "Prometheus web UI"

    - name: "UFW | Allow AlertManager"
      community.general.ufw:
        rule: allow
        port: "9093"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "AlertManager web UI"

    - name: "UFW | Allow Loki from telemetry network (Promtail push)"
      community.general.ufw:
        rule: allow
        port: "3100"
        proto: tcp
        src: "192.168.100.0/24"
        comment: "Loki log ingestion from Promtail"

    - name: "UFW | Allow node_exporter scrapes from observability VM"
      community.general.ufw:
        rule: allow
        port: "9100"
        proto: tcp
        src: "192.168.100.50"
        comment: "Prometheus node_exporter scrapes"

    # ── Start stack ───────────────────────────────────────────────────
    - name: "Compose | Deploy environment file"
      ansible.builtin.template:
        src: observability.env.j2
        dest: "{{ obs_dir }}/.env"
        owner: ansible
        mode: '0600'

    - name: "Compose | Deploy docker-compose.yml"
      ansible.builtin.template:
        src: docker-compose.yml.j2
        dest: "{{ obs_dir }}/docker-compose.yml"
        owner: ansible
        mode: '0644'

    - name: "Docker | Start observability stack"
      community.docker.docker_compose_v2:
        project_src: "{{ obs_dir }}"
        pull: always
        state: present
      become: false

    - name: "Grafana | Wait for web UI"
      ansible.builtin.uri:
        url: "http://172.16.0.50:3000/api/health"
        status_code: 200
      register: grafana_health
      retries: 20
      delay: 10
      until: grafana_health.status == 200
      delegate_to: localhost

    - name: "Prometheus | Wait for health endpoint"
      ansible.builtin.uri:
        url: "http://172.16.0.50:9090/-/healthy"
        status_code: 200
      register: prom_health
      retries: 12
      delay: 5
      until: prom_health.status == 200
      delegate_to: localhost

    - name: "Stack | Report"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════════"
          - " Observability stack running"
          - " Grafana:      http://172.16.0.50:3000"
          - " Prometheus:   http://172.16.0.50:9090"
          - " AlertManager: http://172.16.0.50:9093"
          - " Loki:         http://172.16.0.50:3100"
          - "════════════════════════════════════════════"

  handlers:
    - name: Reload Prometheus
      ansible.builtin.uri:
        url: "http://172.16.0.50:9090/-/reload"
        method: POST
        status_code: 200
      delegate_to: localhost
      failed_when: false
EOF
```

---

## 38.5 — Prometheus Configuration: Dynamic from Inventory

The key differentiator from a static `prometheus.yml` — Ansible generates the scrape targets directly from the inventory, so adding a new VM to `hosts.yml` automatically includes it in Prometheus scraping on the next deploy.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/templates/prometheus.yml.j2 << 'EOF'
# Managed by Ansible — regenerated from inventory on each deploy
# To add a new scrape target: add host to inventory, re-run deploy.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    environment: 'lab'
    datacenter: 'proxmox-home-lab'

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

scrape_configs:

  # ── Prometheus itself ───────────────────────────────────────────────
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          role: 'observability'

  # ── Node Exporter on all infrastructure VMs ─────────────────────────
  # Dynamically built from inventory group: linux_vms
  - job_name: 'node'
    static_configs:
      - targets:
{% for host in groups['linux_vms'] %}
          - '{{ hostvars[host]["ansible_host"] | regex_replace("^172\\.16\\.0\\.", "192.168.100.") }}:9100'
{% endfor %}
        labels:
          job: 'node-exporter'

    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+).*'
        replacement: '${1}'

  # ── SNMP Exporter — Cisco IOS-XE devices ───────────────────────────
  # Dynamically built from inventory group: cisco_ios
  - job_name: 'snmp_ios'
    static_configs:
      - targets:
{% for host in groups['cisco_ios'] %}
          - '{{ hostvars[host]["ansible_host"] }}'
{% endfor %}
    metrics_path: /snmp
    params:
      module: [cisco_ios]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
      - source_labels: [instance]
        target_label: device_type
        replacement: 'cisco_ios'

  # ── SNMP Exporter — Cisco NX-OS devices ────────────────────────────
  - job_name: 'snmp_nxos'
    static_configs:
      - targets:
{% for host in groups['cisco_nxos'] %}
          - '{{ hostvars[host]["ansible_host"] }}'
{% endfor %}
    metrics_path: /snmp
    params:
      module: [cisco_nxos]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
      - source_labels: [instance]
        target_label: device_type
        replacement: 'cisco_nxos'

  # ── SNMP Exporter — PAN-OS devices ─────────────────────────────────
  - job_name: 'snmp_panos'
    static_configs:
      - targets:
{% for host in groups['panos_devices'] %}
          - '{{ hostvars[host]["ansible_host"] }}'
{% endfor %}
    metrics_path: /snmp
    params:
      module: [paloalto_panos]
      auth: [public_v2]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
      - source_labels: [instance]
        target_label: device_type
        replacement: 'panos'

  # ── SNMP Exporter scrape health ─────────────────────────────────────
  - job_name: 'snmp_exporter'
    static_configs:
      - targets: ['snmp-exporter:9116']
EOF
```

### ### ℹ️ How the template renders

```
For a group cisco_ios containing wan-r1 (172.16.0.11) and wan-r2 (172.16.0.12),
the Jinja2 loop produces:

  - job_name: 'snmp_ios'
    static_configs:
      - targets:
          - '172.16.0.11'
          - '172.16.0.12'
          - '172.16.0.21'
          - '172.16.0.33'
          - '172.16.0.34'

Adding leaf-03 to [leaf_switches] in hosts.yml and re-running deploy.yml
automatically adds 172.16.0.35 to this scrape job — no manual prometheus.yml
editing required. This is the operational advantage of dynamic config generation.
```

---

## 38.6 — Node Exporter on All VMs (Native Binary)

Node Exporter runs as a native binary and systemd service on each VM — not in Docker. This gives it access to the host's full filesystem, network stats, and hardware metrics that Docker can't see cleanly.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/install_exporters.yml << 'EOF'
---
# Install Node Exporter and Promtail as native binaries on all VMs
# Usage: ansible-playbook playbooks/infrastructure/observability/install_exporters.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Exporters | Install Node Exporter on all VMs"
  hosts: linux_vms
  gather_facts: true
  become: true
  tags: [exporters, node_exporter]

  vars:
    node_exporter_version: "1.8.2"
    node_exporter_user: node_exporter
    node_exporter_port: 9100
    node_exporter_url: >-
      https://github.com/prometheus/node_exporter/releases/download/
      v{{ node_exporter_version }}/
      node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz

  tasks:

    - name: "NodeExporter | Create dedicated system user"
      ansible.builtin.user:
        name: "{{ node_exporter_user }}"
        system: true
        shell: /bin/false
        create_home: false
        comment: "Node Exporter service user"

    - name: "NodeExporter | Check installed version"
      ansible.builtin.command:
        cmd: /usr/local/bin/node_exporter --version
      register: current_version
      failed_when: false
      changed_when: false

    - name: "NodeExporter | Download and install binary"
      when: >
        current_version.rc != 0 or
        node_exporter_version not in current_version.stderr
      block:
        - name: "Download node_exporter archive"
          ansible.builtin.get_url:
            url: "{{ node_exporter_url | replace('\n', '') | replace(' ', '') }}"
            dest: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
            mode: '0644'
            timeout: 60

        - name: "Extract archive"
          ansible.builtin.unarchive:
            src: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
            dest: /tmp/
            remote_src: true

        - name: "Install binary"
          ansible.builtin.copy:
            src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
            dest: /usr/local/bin/node_exporter
            owner: root
            group: root
            mode: '0755'
            remote_src: true

        - name: "Cleanup"
          ansible.builtin.file:
            path: "{{ item }}"
            state: absent
          loop:
            - "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
            - "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64"

    - name: "NodeExporter | Create systemd unit"
      ansible.builtin.copy:
        content: |
          [Unit]
          Description=Prometheus Node Exporter
          Documentation=https://prometheus.io/docs/guides/node-exporter/
          After=network-online.target

          [Service]
          User={{ node_exporter_user }}
          Group={{ node_exporter_user }}
          Type=simple
          ExecStart=/usr/local/bin/node_exporter \
            --web.listen-address=:{{ node_exporter_port }} \
            --collector.systemd \
            --collector.processes \
            --collector.interrupts
          Restart=on-failure
          RestartSec=5s
          NoNewPrivileges=true
          ProtectSystem=strict
          ProtectHome=true
          PrivateTmp=true

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/node_exporter.service
        mode: '0644'
      notify:
        - Reload systemd
        - Restart node_exporter

    - name: "NodeExporter | Enable and start service"
      ansible.builtin.service:
        name: node_exporter
        state: started
        enabled: true

    - name: "NodeExporter | Verify metrics endpoint"
      ansible.builtin.uri:
        url: "http://{{ ansible_host | regex_replace('^172\\.16\\.0\\.', '192.168.100.') }}:9100/metrics"
        return_content: false
        status_code: 200
      delegate_to: localhost
      register: exporter_check
      retries: 5
      delay: 3
      until: exporter_check.status == 200

  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true

    - name: Restart node_exporter
      ansible.builtin.service:
        name: node_exporter
        state: restarted
EOF
```

---

## 38.7 — Promtail on All VMs

```bash
cat >> ~/projects/ansible-network/playbooks/infrastructure/observability/install_exporters.yml << 'EOF'

- name: "Promtail | Install Promtail log shipper on all VMs"
  hosts: linux_vms
  gather_facts: true
  become: true
  tags: [exporters, promtail]

  vars:
    promtail_version: "3.0.0"
    loki_url: "http://192.168.100.50:3100"
    promtail_url: >-
      https://github.com/grafana/loki/releases/download/
      v{{ promtail_version }}/
      promtail-linux-amd64.zip

  tasks:

    - name: "Promtail | Install unzip"
      ansible.builtin.apt:
        name: unzip
        state: present

    - name: "Promtail | Check installed version"
      ansible.builtin.command:
        cmd: /usr/local/bin/promtail --version
      register: current_version
      failed_when: false
      changed_when: false

    - name: "Promtail | Download and install binary"
      when: >
        current_version.rc != 0 or
        promtail_version not in current_version.stdout
      block:
        - name: "Download promtail"
          ansible.builtin.get_url:
            url: "{{ promtail_url | replace('\n', '') | replace(' ', '') }}"
            dest: "/tmp/promtail-linux-amd64.zip"
            mode: '0644'
            timeout: 120

        - name: "Extract and install"
          ansible.builtin.unarchive:
            src: /tmp/promtail-linux-amd64.zip
            dest: /usr/local/bin/
            remote_src: true
          notify: Restart promtail

        - name: "Set permissions"
          ansible.builtin.file:
            path: /usr/local/bin/promtail-linux-amd64
            owner: root
            mode: '0755'

        - name: "Create symlink"
          ansible.builtin.file:
            src: /usr/local/bin/promtail-linux-amd64
            dest: /usr/local/bin/promtail
            state: link

    - name: "Promtail | Create config directory"
      ansible.builtin.file:
        path: /etc/promtail
        state: directory
        mode: '0755'

    - name: "Promtail | Deploy config"
      ansible.builtin.template:
        src: promtail-config.yml.j2
        dest: /etc/promtail/config.yml
        mode: '0644'
      notify: Restart promtail

    - name: "Promtail | Create systemd unit"
      ansible.builtin.copy:
        content: |
          [Unit]
          Description=Promtail log shipper
          After=network-online.target

          [Service]
          Type=simple
          ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
          Restart=on-failure
          RestartSec=10s

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/promtail.service
        mode: '0644'
      notify:
        - Reload systemd
        - Restart promtail

    - name: "Promtail | Enable and start"
      ansible.builtin.service:
        name: promtail
        state: started
        enabled: true

  handlers:
    - name: Reload systemd
      ansible.builtin.systemd:
        daemon_reload: true
    - name: Restart promtail
      ansible.builtin.service:
        name: promtail
        state: restarted
EOF
```

### Promtail config template

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/templates/promtail-config.yml.j2 << 'EOF'
# Managed by Ansible — regenerated on each deploy
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: {{ loki_url }}/loki/api/v1/push
    tenant_id: lab

scrape_configs:

  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          host: {{ inventory_hostname }}
          env: lab
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets:
          - localhost
        labels:
          job: auth
          host: {{ inventory_hostname }}
          env: lab
          __path__: /var/log/auth.log

  - job_name: ansible
    static_configs:
      - targets:
          - localhost
        labels:
          job: ansible
          host: {{ inventory_hostname }}
          env: lab
          __path__: /var/log/ansible.log

  {% if 'docker' in ansible_facts.packages | default({}) %}
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          host: {{ inventory_hostname }}
          env: lab
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
      - labels:
          stream:
      - output:
          source: output
  {% endif %}
EOF
```

---

## 38.8 — Grafana Datasource Provisioning

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/templates/grafana-datasources.yml.j2 << 'EOF'
# Managed by Ansible — provisioned automatically at Grafana startup
apiVersion: 1

datasources:

  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: loki

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
    jsonData:
      maxLines: 1000
      derivedFields:
        - name: TraceID
          matcherRegex: "traceID=(\\w+)"
          url: "${__value.raw}"

  - name: AlertManager
    type: alertmanager
    access: proxy
    url: http://alertmanager:9093
    editable: false
    jsonData:
      handleGrafanaManagedAlerts: false
      implementation: prometheus
EOF
```

---

## 38.9 — Grafana Dashboards

### Import community dashboards via Ansible

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/configure_grafana.yml << 'EOF'
---
# Configure Grafana: import dashboards and set up folders
# Usage: ansible-playbook observability/configure_grafana.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Grafana | Configure dashboards and folders"
  hosts: localhost
  gather_facts: false
  tags: [grafana, dashboards]

  vars:
    grafana_url: "http://172.16.0.50:3000"
    grafana_user: admin
    grafana_password: "{{ vault_grafana_admin_password }}"

    # Community dashboard IDs from grafana.com/grafana/dashboards
    community_dashboards:
      - id: 1860        # Node Exporter Full — most popular VM dashboard
        folder: "Infrastructure"
        title: "Node Exporter Full"
      - id: 11074       # Node Exporter for Prometheus — simpler variant
        folder: "Infrastructure"
        title: "Node Exporter Basic"
      - id: 14981       # SNMP Interface Traffic — network device traffic
        folder: "Network Devices"
        title: "SNMP Interface Traffic"
      - id: 9526        # Loki + Prometheus — log rate dashboards
        folder: "Logs"
        title: "Loki Dashboard Quick Search"

  tasks:

    # ── Folders ───────────────────────────────────────────────────────
    - name: "Folders | Create dashboard folders"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/folders"
        method: POST
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
        body_format: json
        body:
          title: "{{ item }}"
        status_code: [200, 412]     # 412 = folder already exists
      loop:
        - Infrastructure
        - Network Devices
        - Logs
        - Lab Overview

    # ── Import community dashboards ───────────────────────────────────
    - name: "Dashboards | Get folder UIDs"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/folders"
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
      register: folders_result

    - name: "Dashboards | Build folder UID map"
      ansible.builtin.set_fact:
        folder_uid_map: >-
          {{ dict(folders_result.json
             | map(attribute='title')
             | zip(folders_result.json | map(attribute='uid'))) }}

    - name: "Dashboards | Import community dashboard {{ item.title }}"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/dashboards/import"
        method: POST
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
        body_format: json
        body:
          dashboardId: "{{ item.id }}"
          overwrite: true
          folderId: 0
          folderUid: "{{ folder_uid_map[item.folder] }}"
          inputs:
            - name: DS_PROMETHEUS
              type: datasource
              pluginId: prometheus
              value: Prometheus
            - name: DS_LOKI
              type: datasource
              pluginId: loki
              value: Loki
        status_code: 200
      loop: "{{ community_dashboards }}"
      loop_control:
        label: "{{ item.title }} (id: {{ item.id }})"

    # ── Custom network fabric dashboard ───────────────────────────────
    - name: "Dashboards | Deploy custom fabric dashboard"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/dashboards/db"
        method: POST
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
        body_format: json
        body:
          overwrite: true
          folderUid: "{{ folder_uid_map['Network Devices'] }}"
          dashboard: "{{ lookup('file', 'files/fabric_dashboard.json') | from_json }}"
        status_code: 200
      register: fabric_dash

    - name: "Dashboards | Report"
      ansible.builtin.debug:
        msg:
          - "Community dashboards imported: {{ community_dashboards | length }}"
          - "Custom fabric dashboard: {{ fabric_dash.json.url }}"
          - "Grafana: http://172.16.0.50:3000"
EOF
```

### Custom network fabric dashboard (PromQL)

The custom dashboard visualises the lab fabric with panels that Zabbix can't express — cross-device comparisons, rate calculations, and multi-dimensional filtering.

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/observability/files

cat > ~/projects/ansible-network/playbooks/infrastructure/observability/files/fabric_dashboard.json << 'DASHBOARD_EOF'
{
  "title": "Lab Network Fabric",
  "uid": "lab-fabric-v1",
  "schemaVersion": 38,
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  "panels": [

    {
      "title": "Device Reachability",
      "type": "stat",
      "gridPos": {"x": 0, "y": 0, "w": 24, "h": 4},
      "options": {
        "colorMode": "background",
        "reduceOptions": {"calcs": ["lastNotNull"]}
      },
      "targets": [{
        "expr": "up{job=~\"snmp.*\"}",
        "legendFormat": "{{instance}}",
        "instant": true
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"color": "red", "value": 0},
              {"color": "green", "value": 1}
            ]
          },
          "mappings": [
            {"options": {"0": {"text": "DOWN"}}, "type": "value"},
            {"options": {"1": {"text": "UP"}}, "type": "value"}
          ]
        }
      }
    },

    {
      "title": "Interface Status — All Devices",
      "type": "table",
      "gridPos": {"x": 0, "y": 4, "w": 12, "h": 8},
      "targets": [{
        "expr": "ifOperStatus{job=~\"snmp.*\"}",
        "instant": true,
        "legendFormat": "{{instance}} — {{ifDescr}}"
      }],
      "transformations": [
        {"id": "sortBy", "options": {"fields": [{"displayName": "Value", "desc": true}]}}
      ],
      "fieldConfig": {
        "defaults": {
          "custom": {"displayMode": "color-background"},
          "thresholds": {
            "steps": [
              {"color": "red", "value": 0},
              {"color": "red", "value": 2},
              {"color": "green", "value": 1}
            ]
          },
          "mappings": [
            {"options": {"1": {"text": "up", "color": "green"}}, "type": "value"},
            {"options": {"2": {"text": "down", "color": "red"}}, "type": "value"}
          ]
        }
      }
    },

    {
      "title": "Interface Traffic In — All Devices (bits/s)",
      "type": "timeseries",
      "gridPos": {"x": 12, "y": 4, "w": 12, "h": 8},
      "targets": [{
        "expr": "rate(ifHCInOctets{job=~\"snmp.*\",ifOperStatus=\"1\"}[5m]) * 8",
        "legendFormat": "{{instance}} {{ifDescr}} in"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "bps",
          "custom": {"lineInterpolation": "smooth"}
        }
      }
    },

    {
      "title": "BGP Peer State",
      "type": "stat",
      "gridPos": {"x": 0, "y": 12, "w": 12, "h": 4},
      "options": {"colorMode": "background"},
      "targets": [{
        "expr": "bgpPeerState{job=~\"snmp.*\"}",
        "legendFormat": "{{instance}} → {{bgpPeerRemoteAddr}}",
        "instant": true
      }],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              {"color": "red", "value": 0},
              {"color": "orange", "value": 3},
              {"color": "green", "value": 6}
            ]
          },
          "mappings": [
            {"options": {
              "1": {"text": "idle"},
              "2": {"text": "connect"},
              "3": {"text": "active"},
              "6": {"text": "established", "color": "green"}
            }, "type": "value"}
          ]
        }
      }
    },

    {
      "title": "VM CPU Usage %",
      "type": "timeseries",
      "gridPos": {"x": 0, "y": 16, "w": 12, "h": 8},
      "targets": [{
        "expr": "100 - (avg by(instance)(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
        "legendFormat": "{{instance}}"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 70},
              {"color": "red", "value": 90}
            ]
          }
        }
      }
    },

    {
      "title": "VM Memory Usage %",
      "type": "timeseries",
      "gridPos": {"x": 12, "y": 16, "w": 12, "h": 8},
      "targets": [{
        "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
        "legendFormat": "{{instance}}"
      }],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 80},
              {"color": "red", "value": 90}
            ]
          }
        }
      }
    },

    {
      "title": "Recent Logs (Loki)",
      "type": "logs",
      "gridPos": {"x": 0, "y": 24, "w": 24, "h": 8},
      "options": {
        "showTime": true,
        "showLabels": true,
        "wrapLogMessage": false,
        "sortOrder": "Descending"
      },
      "targets": [{
        "expr": "{env=\"lab\"} |= ``",
        "datasource": {"type": "loki", "uid": "loki"}
      }]
    }

  ]
}
DASHBOARD_EOF
```

### PromQL explained — the key queries

```
Query: up{job=~"snmp.*"}
  Meaning: "Is each SNMP scrape target reachable?"
  up = 1 → Prometheus successfully scraped the target
  up = 0 → Scrape failed (device unreachable, SNMP timeout)
  job=~"snmp.*" → regex match — includes snmp_ios, snmp_nxos, snmp_panos

Query: rate(ifHCInOctets{job=~"snmp.*"}[5m]) * 8
  Meaning: "Inbound interface traffic in bits per second, 5-minute average"
  ifHCInOctets → 64-bit octet counter (HCInOctets = High Capacity)
  rate([5m]) → per-second rate of increase over last 5 minutes
  * 8 → convert bytes to bits

Query: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
  Meaning: "CPU utilisation percentage per VM"
  node_cpu_seconds_total{mode="idle"} → time spent idle
  rate([5m]) → rate of idle time increase (between 0 and 1)
  avg by(instance) → average across all CPU cores for the VM
  100 - (... * 100) → invert: idle% becomes busy%

Query: bgpPeerState{job=~"snmp.*"}
  Meaning: "BGP FSM state for each peer on each device"
  Value 6 = Established, anything else = problem
  Labels include instance (device IP) and bgpPeerRemoteAddr (peer IP)
  The stat panel maps value 6 → green, others → red
```

---

## 38.10 — Alert Rules

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/observability/templates/rules/network_alerts.yml.j2 << 'EOF'
# Network device alert rules — managed by Ansible
groups:
  - name: network_devices
    interval: 30s
    rules:

      - alert: DeviceDown
        expr: up{job=~"snmp.*"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Network device {{ $labels.instance }} is unreachable"
          description: >
            Prometheus cannot scrape {{ $labels.instance }}
            (job: {{ $labels.job }}).
            SNMP polling has failed for more than 2 minutes.
            Check device connectivity and SNMP configuration.

      - alert: InterfaceDown
        expr: ifOperStatus{job=~"snmp.*"} == 2
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Interface {{ $labels.ifDescr }} down on {{ $labels.instance }}"
          description: >
            Interface {{ $labels.ifDescr }} on {{ $labels.instance }}
            has been operationally down for more than 1 minute.

      - alert: BGPPeerDown
        expr: bgpPeerState{job=~"snmp.*"} != 6
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "BGP peer {{ $labels.bgpPeerRemoteAddr }} not established on {{ $labels.instance }}"
          description: >
            BGP peer {{ $labels.bgpPeerRemoteAddr }} on {{ $labels.instance }}
            is not in Established state (current state: {{ $value }}).
            States: 1=idle 2=connect 3=active 4=opensent 5=openconfirm 6=established

      - alert: HighInterfaceErrorRate
        expr: rate(ifInErrors{job=~"snmp.*"}[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.ifDescr }} at {{ $labels.instance }}"
          description: >
            Interface {{ $labels.ifDescr }} on {{ $labels.instance }}
            is receiving more than 10 errors per second.
            Investigate physical layer or duplex mismatch.
EOF

cat > ~/projects/ansible-network/playbooks/infrastructure/observability/templates/rules/vm_alerts.yml.j2 << 'EOF'
# Infrastructure VM alert rules — managed by Ansible
groups:
  - name: infrastructure_vms
    interval: 30s
    rules:

      - alert: VMDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "VM {{ $labels.instance }} is unreachable"
          description: >
            Node Exporter on {{ $labels.instance }} is not responding.
            The VM may be down or node_exporter service may have failed.

      - alert: HighCPU
        expr: >
          100 - (avg by(instance)
          (rate(node_cpu_seconds_total{mode="idle",job="node"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}: {{ $value | printf \"%.1f\" }}%"
          description: >
            CPU utilisation on {{ $labels.instance }} has been above
            85% for more than 5 minutes.

      - alert: HighMemory
        expr: >
          (1 - (node_memory_MemAvailable_bytes{job="node"}
          / node_memory_MemTotal_bytes{job="node"})) * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }}: {{ $value | printf \"%.1f\" }}%"
          description: >
            Memory utilisation on {{ $labels.instance }} has exceeded
            90% for more than 5 minutes.

      - alert: DiskSpaceLow
        expr: >
          (node_filesystem_avail_bytes{job="node",fstype!~"tmpfs|overlay"}
          / node_filesystem_size_bytes{job="node",fstype!~"tmpfs|overlay"}) * 100 < 15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }} ({{ $labels.mountpoint }})"
          description: >
            Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }}
            has less than 15% free space remaining.
EOF
```

---

## 38.11 — Zabbix vs Prometheus: The Comparison

After deploying both, the architectural difference becomes concrete rather than theoretical.

```
Scenario: BGP peer goes down between wan-r1 and panos-fw01

Zabbix response:
  1. Next poll cycle runs (default 1 min)
  2. bgpPeerState OID returns != 6
  3. Trigger expression evaluates — fires if state was 6 last poll
  4. Action sends Slack message
  5. Message: "BGP peer state not established on wan-r1"
  Total latency: up to 1 minute + alert processing time

Prometheus response:
  1. Next scrape of snmp_ios job (default 15s)
  2. bgpPeerState metric value changes to != 6
  3. BGPPeerDown rule evaluates (30s interval) — pending
  4. for: 3m fires AlertManager after 3 minutes sustained
  5. AlertManager routes to Slack channel
  Total latency: 15s scrape + 3m for duration = ~3m 15s before alert

  BUT: Prometheus also records the exact moment the metric changed.
  PromQL: bgpPeerState{instance="172.16.0.11"}[30m]
  Shows the full state history with timestamps — when it went down,
  how long it stayed down, when it recovered. Zabbix shows this too
  in its event log, but Prometheus makes it queryable.

Scenario: "Show me interface utilisation for ALL devices together,
           sorted by highest traffic, over the last 6 hours"

Zabbix:
  - Navigate to each host individually
  - OR build a custom screen/dashboard (manual, per-host)
  - No cross-device aggregation in a single query

Prometheus/Grafana:
  topk(10, rate(ifHCInOctets{job=~"snmp.*"}[5m]) * 8)
  → Single query, all devices, sorted by traffic, any time range
  → Dashboard panel renders this in 2 seconds

When to use Zabbix:
  - Initial device monitoring — add host, apply template, done
  - SNMP trap reception (Prometheus has no native trap receiver)
  - Auto-discovery of network interfaces (Zabbix LLD is mature)
  - Compliance environments requiring agent-based monitoring

When to use Prometheus:
  - Custom metrics from applications (Netbox, AWX, Gitea all
    expose /metrics endpoints)
  - PromQL for complex cross-device queries
  - Grafana dashboards with rich visualisation
  - Correlation with log data (Loki) in the same UI

Both running together:
  The lab runs both because in most enterprise environments you'll
  encounter both — often Zabbix for network devices (legacy) and
  Prometheus for application/cloud infrastructure (modern).
  Knowing how to operate them simultaneously is the skill.
```

---

## 38.12 — Checklist

```
[ ] Observability stack deployed:
    [ ] ~/observability/docker-compose.yml created
    [ ] /opt/observability/{prometheus,grafana,loki,alertmanager,snmp-exporter}
        directories created
    [ ] Config files in place:
        [ ] /opt/observability/loki/loki-config.yml
        [ ] /opt/observability/alertmanager/alertmanager.yml
        [ ] /opt/observability/grafana/provisioning/datasources/datasources.yml
    [ ] vault_grafana_admin_password added to vault
    [ ] docker compose ps — all 5 containers running/healthy
    [ ] http://172.16.0.50:3000 — Grafana UI loads
    [ ] http://172.16.0.50:9090 — Prometheus UI loads
    [ ] http://172.16.0.50:9093 — AlertManager UI loads
    [ ] http://172.16.0.50:3100/ready — Loki returns ready

[ ] Prometheus config generated from inventory:
    [ ] /opt/observability/prometheus/prometheus.yml contains all
        cisco_ios, cisco_nxos, and panos_devices from inventory
    [ ] Prometheus UI → Targets — all targets listed
    [ ] SNMP targets show State: UP for all 7 devices
    [ ] Node targets show State: UP for all 6 VMs

[ ] Node Exporter installed on all VMs (native binary):
    [ ] /usr/local/bin/node_exporter present on each VM
    [ ] node_exporter.service enabled and running on each VM
    [ ] curl http://192.168.100.{20,30,40,50,60}:9100/metrics
        returns Prometheus text format
    [ ] Prometheus UI shows node job targets all UP

[ ] Promtail installed on all VMs (native binary):
    [ ] /usr/local/bin/promtail present on each VM
    [ ] promtail.service enabled and running
    [ ] Loki UI (via Grafana Explore) shows logs from all VMs

[ ] Grafana configured:
    [ ] Prometheus datasource — test returns success
    [ ] Loki datasource — test returns success
    [ ] AlertManager datasource — connected
    [ ] 4 community dashboards imported:
        [ ] Node Exporter Full (id: 1860) visible in Infrastructure folder
        [ ] Node Exporter Basic (id: 11074)
        [ ] SNMP Interface Traffic (id: 14981) in Network Devices folder
        [ ] Loki Quick Search (id: 9526) in Logs folder
    [ ] Custom "Lab Network Fabric" dashboard deployed
        [ ] Device Reachability panel — all devices green
        [ ] Interface Status table — showing all interfaces
        [ ] Interface Traffic panel — showing rate data
        [ ] BGP Peer State panel — all peers green (established)
        [ ] VM CPU panel — all VMs showing values
        [ ] VM Memory panel — all VMs showing values
        [ ] Logs panel — showing entries from Loki

[ ] Alert rules deployed:
    [ ] /opt/observability/prometheus/rules/network_alerts.yml present
    [ ] /opt/observability/prometheus/rules/vm_alerts.yml present
    [ ] Prometheus UI → Alerts — all rules listed (state: inactive)
    [ ] AlertManager UI — connected to Prometheus
    [ ] Test DeviceDown alert:
        [ ] Stop Containerlab on network-lab VM
        [ ] up{job="snmp_ios"} drops to 0 in Prometheus
        [ ] Alert moves to Pending → Firing after 2m
        [ ] AlertManager receives alert
        [ ] Restart Containerlab → alert resolves

[ ] Grafana alerting:
    [ ] At least one Grafana alert rule created (HighCPU panel)
    [ ] Alert state shows Normal in Grafana → Alerting

[ ] Zabbix vs Prometheus comparison understood:
    [ ] Both monitoring the same devices simultaneously
    [ ] Can articulate when each tool is the better choice

[ ] UFW rules on observability VM verified:
    [ ] Port 3000 (Grafana) from 172.16.0.0/24
    [ ] Port 9090 (Prometheus) from 172.16.0.0/24
    [ ] Port 9093 (AlertManager) from 172.16.0.0/24
    [ ] Port 3100 (Loki) from 192.168.100.0/24
    [ ] Port 9100 (node_exporter) from 192.168.100.50

[ ] Snapshot taken:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=105 snap_name=post-observability \
          snap_description='Prometheus Grafana Loki deployed Part 38'"

[ ] All playbooks committed and mirrored to Gitea:
    [ ] playbooks/infrastructure/observability/deploy.yml
    [ ] playbooks/infrastructure/observability/install_exporters.yml
    [ ] playbooks/infrastructure/observability/configure_grafana.yml
    [ ] playbooks/infrastructure/observability/templates/prometheus.yml.j2
    [ ] playbooks/infrastructure/observability/templates/promtail-config.yml.j2
    [ ] playbooks/infrastructure/observability/templates/rules/network_alerts.yml.j2
    [ ] playbooks/infrastructure/observability/templates/rules/vm_alerts.yml.j2
    [ ] playbooks/infrastructure/observability/files/fabric_dashboard.json
```

---

*Both monitoring stacks are running — Zabbix polling via SNMP, Prometheus scraping via exporters, Grafana showing both data sources in dashboards, Loki aggregating logs from every VM, AlertManager routing alerts. The lab fabric is fully observable. Part 39 adds the last piece: centralised logging for the network devices themselves — syslog from every router and switch, SNMP traps, and structured search across everything in Graylog.*

*Next up: **Part 39 — Logging: Graylog + OpenSearch***
