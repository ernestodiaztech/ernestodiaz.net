---
draft: true
title: '40 - Integrating the Stack'
description: "Part 40 of my Ansible learning geared towards Network Engineering."
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

## Part 40: Integrating the Stack — Closing the Loop

> *Parts 37, 38, and 39 built four independent tools. Zabbix polls devices. Prometheus scrapes exporters. Graylog collects syslog. Grafana shows dashboards. Each one works, and each one is useful on its own. But tools in isolation are just complexity — the value comes from the connections between them. A device gets added to Netbox and it should automatically appear in Zabbix and Prometheus without anyone touching a config file. Graylog detects a BGP failure and it should automatically trigger an Ansible playbook that backs up the device, diagnoses the failure, and notifies the team with context already assembled. Grafana should show the whole picture — metrics, logs, and alerts — in a single view. This part builds those connections. By the end, the stack behaves as a system, not a collection of tools.*

---

## 40.1 — The Integration Map

Before writing any code, the full integration picture in one diagram:

```
                         ┌──────────────────────────────────┐
                         │           NETBOX                 │
                         │     Source of Truth              │
                         │  (172.16.0.20)                   │
                         └──────┬───────────────────────────┘
                                │
              ┌─────────────────┼──────────────────────┐
              │  Webhook on     │  Scheduled poll       │
              │  device change  │  (safety net)         │
              ▼                 ▼                       │
    ┌─────────────────┐  ┌─────────────────┐           │
    │  Zabbix         │  │  Prometheus      │           │
    │  Monitoring     │  │  Observability   │           │
    │  (172.16.0.40)  │  │  (172.16.0.50)  │           │
    └────────┬────────┘  └────────┬────────┘           │
             │                    │                     │
             └──────────┬─────────┘                     │
                        │                               │
                        ▼                               │
              ┌─────────────────────────────────────────┴───┐
              │              GRAFANA                        │
              │    Unified Operations Dashboard             │
              │  Sources: Prometheus + Loki + Zabbix API    │
              └──────────────────────────────────────────────┘
                                ▲
                                │
              ┌─────────────────┴───────────────────────────┐
              │             GRAYLOG                         │
              │    Logging + Event Detection                │
              │  (172.16.0.60)                              │
              └──────────────────────┬──────────────────────┘
                                     │
                           Alert fires webhook
                                     │
                                     ▼
                         ┌───────────────────────┐
                         │        AWX            │
                         │  Automation Engine    │
                         │  (172.16.0.30)        │
                         │                       │
                         │  Workflow:            │
                         │  1. Backup device     │
                         │  2. Diagnose failure  │
                         │  3. Notify with       │
                         │     context           │
                         └───────────────────────┘

Before Part 40:  Each box works. No arrows between them.
After Part 40:   Every arrow is a live, tested integration.
```

---

## 40.2 — Integration 1: Netbox → Zabbix Sync

### Before

```
Device added to Netbox → nothing happens in Zabbix.
Engineer manually registers device in Zabbix.
Device gets monitored eventually — if the engineer remembers.
```

### After

```
Device added/updated in Netbox
  → Netbox webhook fires to AWX (Part 36 configured this)
  → AWX runs netbox_to_zabbix_sync.yml
  → Device registered in Zabbix with correct template and SNMP config
  → Zabbix starts polling within seconds
  → Scheduled poll runs nightly to catch anything the webhook missed
```

```bash
cat > ~/projects/ansible-network/playbooks/integration/netbox_to_zabbix_sync.yml << 'EOF'
---
# Sync a device from Netbox into Zabbix
# Triggered by: Netbox webhook (event-driven) OR AWX schedule (poll)
# Usage:
#   Event-driven (from Netbox webhook via AWX extra_vars):
#   ansible-playbook integration/netbox_to_zabbix_sync.yml
#     -e "netbox_device_id=8 netbox_event=created"
#
#   Scheduled full sync:
#   ansible-playbook integration/netbox_to_zabbix_sync.yml
#     -e "sync_all=true"

- name: "Integration | Sync Netbox devices to Zabbix"
  hosts: localhost
  gather_facts: false
  tags: [integration, netbox, zabbix]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"
    zabbix_url: "http://172.16.0.40"
    zabbix_user: Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"
    netbox_device_id: ~
    netbox_event: ~
    sync_all: false

    # Map Netbox platform slug → Zabbix template name
    platform_template_map:
      ios-xe: "Cisco IOS by SNMP"
      nxos:   "Cisco NX-OS by SNMP"
      panos:  "Palo Alto PA-Series by SNMP"
      linux:  "Linux by Zabbix agent"

    # Map Netbox device role slug → Zabbix host group
    role_group_map:
      wan-router:    ["Network Devices", "WAN Routers", "Cisco IOS-XE"]
      distribution:  ["Network Devices", "Distribution", "Cisco IOS-XE"]
      spine-switch:  ["Network Devices", "Spine Switches", "Cisco NX-OS"]
      leaf-switch:   ["Network Devices", "Leaf Switches", "Cisco IOS-XE"]
      firewall:      ["Network Devices", "Firewalls", "Palo Alto"]

  tasks:

    # ── Fetch device(s) from Netbox ───────────────────────────────────
    - name: "Netbox | Fetch single device (event-driven)"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/dcim/devices/{{ netbox_device_id }}/"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: single_device
      when: netbox_device_id is not none and not sync_all | bool

    - name: "Netbox | Fetch all active devices (scheduled poll)"
      ansible.builtin.uri:
        url: "{{ netbox_url }}/api/dcim/devices/?status=active&limit=200"
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: all_devices
      when: sync_all | bool

    - name: "Integration | Normalise device list"
      ansible.builtin.set_fact:
        devices_to_sync: >-
          {{ [single_device.json] if (netbox_device_id is not none and not sync_all | bool)
             else all_devices.json.results | default([]) }}

    - name: "Integration | Report sync scope"
      ansible.builtin.debug:
        msg: "Syncing {{ devices_to_sync | length }} device(s) to Zabbix"

    # ── Authenticate to Zabbix ────────────────────────────────────────
    - name: "Zabbix | Authenticate"
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
      register: zbx_auth
      no_log: true

    - name: "Zabbix | Set auth token"
      ansible.builtin.set_fact:
        zbx_token: "{{ zbx_auth.json.result }}"
      no_log: true

    # ── Sync each device ──────────────────────────────────────────────
    - name: "Zabbix | Register/update device: {{ item.name }}"
      community.zabbix.zabbix_host:
        server_url: "{{ zabbix_url }}"
        login_user: "{{ zabbix_user }}"
        login_password: "{{ zabbix_password }}"
        host_name: "{{ item.name }}"
        host_groups: >-
          {{ role_group_map[item.device_role.slug] | default(['Network Devices']) }}
        link_templates: >-
          {{ [platform_template_map[item.platform.slug | default('linux')]] }}
        status: >-
          {{ 'enabled' if item.status.value == 'active' else 'disabled' }}
        state: present
        interfaces:
          - type: 2
            main: 1
            useip: 1
            ip: "{{ item.primary_ip.address | ipaddr('address') }}"
            dns: ""
            port: "161"
            details:
              version: 2
              bulk: 1
              community: "{{ vault_snmp_community_ro }}"
        macros:
          - macro: "{$NETBOX_ID}"
            value: "{{ item.id | string }}"
          - macro: "{$PLATFORM}"
            value: "{{ item.platform.slug | default('unknown') }}"
          - macro: "{$DEVICE_ROLE}"
            value: "{{ item.device_role.slug | default('unknown') }}"
        inventory_mode: automatic
        inventory:
          location: "{{ item.site.name | default('Lab') }}"
          contact: "admin@lab.local"
      loop: "{{ devices_to_sync }}"
      loop_control:
        label: "{{ item.name }}"
      when: item.primary_ip is not none

    - name: "Integration | Report Netbox → Zabbix sync complete"
      ansible.builtin.debug:
        msg:
          - "Netbox → Zabbix sync complete"
          - "Devices processed: {{ devices_to_sync | length }}"
          - "Trigger: {{ 'event-driven (device_id=' + netbox_device_id|string + ')' if netbox_device_id is not none else 'scheduled poll' }}"
          - "View in Zabbix: http://172.16.0.40/zabbix/hosts.php"
EOF
```

### Schedule the sync as an AWX job

```yaml
# AWX job template configuration (created via AWX API or UI in Part 42)
# Name: Netbox → Zabbix Sync (Scheduled)
# Playbook: playbooks/integration/netbox_to_zabbix_sync.yml
# Extra vars:
#   sync_all: true
# Schedule: Daily at 02:00
# Description: Nightly catch-all sync — ensures Zabbix reflects Netbox

# AWX job template for event-driven sync (triggered by Netbox webhook):
# Name: Netbox → Zabbix Sync (Device Change)
# Playbook: playbooks/integration/netbox_to_zabbix_sync.yml
# Extra vars: (passed by Netbox webhook)
#   netbox_device_id: (from webhook payload)
#   netbox_event: (created/updated)
# Webhook: enabled — Netbox webhook from Part 36 points here
```

### Test the event-driven sync

```bash
# From the control node — simulate adding a new device in Netbox
# Step 1: Add device in Netbox UI (http://172.16.0.20)
#         Devices → Add Device:
#           Name: test-switch-01
#           Role: Leaf Switch
#           Platform: ios-xe
#           Primary IPv4: 172.16.0.35/24
#           Status: Active

# Step 2: Watch for Netbox webhook to fire to AWX
# (Check AWX Jobs — a new job should appear within seconds)

# Step 3: Verify device appeared in Zabbix
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
      "output": ["hostid","host","status"],
      "filter": {"host": ["test-switch-01"]}
    },
    "auth": "'"${ZBX_TOKEN}"'",
    "id": 1
  }' \
  http://172.16.0.40/api_jsonrpc.php \
  | python3 -m json.tool | grep -E '"host"|"status"'
# Expected: "host": "test-switch-01", "status": "0" (enabled)
```

---

## 40.3 — Integration 2: Netbox → Prometheus Scrape Targets

### Before

```
prometheus.yml is static — generated from Ansible inventory at deploy time.
New device added to Netbox is not in inventory.
Prometheus doesn't know about the device until:
  1. inventory/hosts.yml is manually updated
  2. deploy.yml is re-run to regenerate prometheus.yml
  3. Prometheus reloads config
```

### After

```
Device added/updated in Netbox
  → Netbox webhook fires to AWX
  → AWX runs netbox_to_prometheus_sync.yml
  → Prometheus file-based service discovery files updated
  → Prometheus picks up new target within 15 seconds (no reload needed)
```

File-based service discovery (`file_sd_configs`) is the right tool here — Prometheus watches a directory for JSON target files and automatically updates scrape targets when files change, without a config reload.

```bash
# Add file_sd_configs to prometheus.yml (update the template from Part 38)
cat >> /opt/observability/prometheus/prometheus.yml << 'EOF'

  # ── File-based service discovery ──────────────────────────────────
  # Prometheus watches this directory — targets update automatically
  # when Ansible writes new files. No reload needed.
  - job_name: netbox_network_devices
    metrics_path: /snmp
    params:
      module: [cisco_ios]
      auth: [lab_v2c]
    file_sd_configs:
      - files:
          - /etc/prometheus/sd_targets/network_devices_*.json
        refresh_interval: 15s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
EOF

# Create the service discovery targets directory
sudo mkdir -p /opt/observability/prometheus/sd_targets
```

```bash
cat > ~/projects/ansible-network/playbooks/integration/netbox_to_prometheus_sync.yml << 'EOF'
---
# Sync Netbox devices to Prometheus file-based service discovery
# Triggered by: Netbox webhook OR scheduled poll
# Prometheus picks up file changes within 15 seconds — no reload needed

- name: "Integration | Sync Netbox devices to Prometheus SD targets"
  hosts: localhost
  gather_facts: false
  tags: [integration, netbox, prometheus]

  vars:
    netbox_url: "http://172.16.0.20"
    netbox_token: "{{ vault_netbox_api_token }}"
    prometheus_sd_dir: "/opt/observability/prometheus/sd_targets"
    netbox_device_id: ~
    sync_all: false

    # Map platform slug → SNMP module name in snmp.yml
    platform_module_map:
      ios-xe: cisco_ios
      nxos:   cisco_nxos
      panos:  paloalto_panos

  tasks:

    - name: "Netbox | Fetch device(s)"
      ansible.builtin.uri:
        url: >-
          {{ netbox_url }}/api/dcim/devices/
          {{ netbox_device_id ~ '/' if netbox_device_id is not none and not sync_all | bool else '?status=active&limit=200' }}
        method: GET
        headers:
          Authorization: "Token {{ netbox_token }}"
      register: netbox_response

    - name: "Integration | Normalise response"
      ansible.builtin.set_fact:
        devices: >-
          {{ [netbox_response.json] if (netbox_device_id is not none and not sync_all | bool)
             else netbox_response.json.results | default([]) }}

    - name: "Integration | Filter to devices with primary IPs"
      ansible.builtin.set_fact:
        addressable_devices: >-
          {{ devices | selectattr('primary_ip', 'ne', None) | list }}

    # ── Group devices by platform for separate SD files ──────────────
    - name: "SD | Build per-platform target lists"
      ansible.builtin.set_fact:
        sd_ios: >-
          {{ addressable_devices | selectattr('platform.slug', 'eq', 'ios-xe') | list }}
        sd_nxos: >-
          {{ addressable_devices | selectattr('platform.slug', 'eq', 'nxos') | list }}
        sd_panos: >-
          {{ addressable_devices | selectattr('platform.slug', 'eq', 'panos') | list }}

    # ── Write SD target files ─────────────────────────────────────────
    - name: "SD | Write Cisco IOS-XE targets file"
      ansible.builtin.copy:
        content: |
          {{ sd_ios | map(attribute='primary_ip.address') |
             map('regex_replace', '/[0-9]+$', '') |
             list |
             zip(sd_ios | map(attribute='name')) |
             map('community.general.dict_zip', ['__address__', 'device']) |
             map(attribute='__address__') |
             list | to_nice_json }}
        dest: "{{ prometheus_sd_dir }}/network_devices_ios.json"
        mode: '0644'
      delegate_to: observability
      become: true
      when: sd_ios | length > 0

    - name: "SD | Write SD targets file (correct format)"
      ansible.builtin.copy:
        content: >
          {{
            [
              {
                "targets": sd_ios | map(attribute='primary_ip.address') |
                           map('regex_replace', '/[0-9]+$', '') | list,
                "labels": {
                  "module": "cisco_ios",
                  "category": "network",
                  "platform": "ios-xe"
                }
              }
            ] | to_nice_json
          }}
        dest: "{{ prometheus_sd_dir }}/network_devices_ios.json"
        mode: '0644'
      delegate_to: observability
      become: true
      when: sd_ios | length > 0

    - name: "SD | Write Cisco NX-OS targets file"
      ansible.builtin.copy:
        content: >
          {{
            [
              {
                "targets": sd_nxos | map(attribute='primary_ip.address') |
                           map('regex_replace', '/[0-9]+$', '') | list,
                "labels": {
                  "module": "cisco_nxos",
                  "category": "network",
                  "platform": "nxos"
                }
              }
            ] | to_nice_json
          }}
        dest: "{{ prometheus_sd_dir }}/network_devices_nxos.json"
        mode: '0644'
      delegate_to: observability
      become: true
      when: sd_nxos | length > 0

    - name: "SD | Write PAN-OS targets file"
      ansible.builtin.copy:
        content: >
          {{
            [
              {
                "targets": sd_panos | map(attribute='primary_ip.address') |
                           map('regex_replace', '/[0-9]+$', '') | list,
                "labels": {
                  "module": "paloalto_panos",
                  "category": "network",
                  "platform": "panos"
                }
              }
            ] | to_nice_json
          }}
        dest: "{{ prometheus_sd_dir }}/network_devices_panos.json"
        mode: '0644'
      delegate_to: observability
      become: true
      when: sd_panos | length > 0

    - name: "Integration | Verify Prometheus picked up new targets"
      ansible.builtin.uri:
        url: "http://172.16.0.50:9090/api/v1/targets?state=active"
        method: GET
      register: prom_targets
      delay: 20     # Give Prometheus 15s to reload file_sd, plus buffer

    - name: "Integration | Report"
      ansible.builtin.debug:
        msg:
          - "Netbox → Prometheus SD sync complete"
          - "IOS-XE devices: {{ sd_ios | length }}"
          - "NX-OS devices:  {{ sd_nxos | length }}"
          - "PAN-OS devices: {{ sd_panos | length }}"
          - "Active Prometheus targets: {{ prom_targets.json.data.activeTargets | length }}"
          - "SD files written to: {{ prometheus_sd_dir }}"
          - "Prometheus picks up changes automatically within 15 seconds"
EOF
```

---

## 40.4 — Integration 3: Graylog Alert → AWX Self-Healing

### Before

```
BGP neighbor drops on wan-r1.
Graylog detects it, fires a Slack alert.
Network engineer sees the alert, SSHs to the device, runs diagnostics.
Engineer checks BGP neighbours, interface states, routing table.
Engineer decides whether to escalate.
Total time from alert to diagnosis: 10-20 minutes minimum.
```

### After

```
BGP neighbor drops on wan-r1.
Graylog detects it, fires Slack alert AND webhook to AWX.
AWX runs three-job workflow:
  1. Backup: capture device config and show output before anything changes
  2. Diagnose: collect BGP neighbors, interface states, routing table,
               format into a structured report
  3. Notify: post the diagnosis report to Slack with all context
             — engineer reads the full picture before touching anything
Total time from alert to engineer having full diagnosis: < 2 minutes.
Self-healing (optional): if BGP is down due to interface being
  administratively shut, playbook brings it back up and notifies.
```

### Step 1: Simple pattern — alert → diagnose → notify

```bash
mkdir -p ~/projects/ansible-network/playbooks/remediation

cat > ~/projects/ansible-network/playbooks/remediation/diagnose_bgp.yml << 'EOF'
---
# Triggered by Graylog BGP Neighbor Down alert via AWX webhook
# Extra vars from Graylog webhook (configured in Part 39):
#   graylog_device_name: wan-r1
#   graylog_bgp_peer_ip: 10.0.0.1
#   graylog_event_id: <graylog event ID for cross-reference>

- name: "Remediation | Diagnose BGP neighbor failure"
  hosts: "{{ graylog_device_name | default('wan-r1') }}"
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: [remediation, bgp, diagnose]

  vars:
    device_name: "{{ graylog_device_name | default(inventory_hostname) }}"
    bgp_peer_ip: "{{ graylog_bgp_peer_ip | default('unknown') }}"
    slack_webhook: "{{ vault_slack_webhook_url }}"
    netbox_url: "http://172.16.0.20"
    graylog_event_id: "{{ graylog_event_id | default('N/A') }}"
    diagnosis_timestamp: "{{ lookup('pipe', 'date +%Y-%m-%d\\ %H:%M:%S\\ UTC') }}"

  tasks:

    # ── Stage 1: Collect diagnostics ─────────────────────────────────
    - name: "Diagnose | Collect BGP neighbor summary"
      cisco.ios.ios_command:
        commands:
          - "show bgp summary"
          - "show bgp neighbors {{ bgp_peer_ip }}"
      register: bgp_output

    - name: "Diagnose | Collect interface states"
      cisco.ios.ios_command:
        commands:
          - "show interfaces summary"
          - "show ip interface brief"
      register: interface_output

    - name: "Diagnose | Collect routing table (last 20 routes)"
      cisco.ios.ios_command:
        commands:
          - "show ip route | head 25"
      register: route_output

    - name: "Diagnose | Collect recent syslog from device buffer"
      cisco.ios.ios_command:
        commands:
          - "show logging | tail 20"
      register: logging_output

    # ── Stage 2: Determine likely root cause ─────────────────────────
    - name: "Diagnose | Analyse BGP output for root cause"
      ansible.builtin.set_fact:
        root_cause: >-
          {% if 'Idle' in bgp_output.stdout[0] %}
          BGP session idle — peer unreachable or TCP not established
          {% elif 'Active' in bgp_output.stdout[0] %}
          BGP session active — attempting to connect, TCP failing
          {% elif 'Connect' in bgp_output.stdout[0] %}
          BGP session in Connect state — TCP SYN sent, awaiting response
          {% elif bgp_peer_ip not in bgp_output.stdout[0] %}
          BGP peer {{ bgp_peer_ip }} not found in neighbor table
          {% else %}
          BGP session state change detected — see raw output below
          {% endif %}

    - name: "Diagnose | Check if interface is administratively down"
      ansible.builtin.set_fact:
        interface_admin_down: >-
          {{ 'administratively down' in interface_output.stdout[0] }}

    # ── Stage 3: Attempt remediation (self-healing) ──────────────────
    - name: "Self-heal | Bring up admin-down interface if detected"
      cisco.ios.ios_config:
        lines:
          - "no shutdown"
        parents: "interface {{ item }}"
        save_when: changed
      loop: "{{ interface_output.stdout[0] | regex_findall('(\\S+) .* administratively down') }}"
      when: interface_admin_down | bool
      register: remediation_result

    - name: "Self-heal | Re-check BGP after remediation"
      cisco.ios.ios_command:
        commands: ["show bgp summary"]
      register: post_remediation_bgp
      when: interface_admin_down | bool
      delay: 30     # Wait for BGP to reconverge

    - name: "Self-heal | Determine if remediation succeeded"
      ansible.builtin.set_fact:
        remediation_status: >-
          {{ 'SUCCESS — BGP reconverged after bringing up interface'
             if (interface_admin_down | bool and 'Established' in post_remediation_bgp.stdout[0] | default(''))
             else ('ATTEMPTED — Interface brought up, BGP not yet reconverged'
                   if interface_admin_down | bool
                   else 'NOT ATTEMPTED — Root cause requires manual investigation') }}

    # ── Stage 4: Notify with full context ────────────────────────────
    - name: "Notify | Build diagnosis report"
      ansible.builtin.set_fact:
        diagnosis_report:
          timestamp: "{{ diagnosis_timestamp }}"
          device: "{{ device_name }}"
          bgp_peer: "{{ bgp_peer_ip }}"
          graylog_event: "{{ graylog_event_id }}"
          root_cause: "{{ root_cause }}"
          remediation: "{{ remediation_status }}"
          bgp_summary: "{{ bgp_output.stdout[0][:500] }}"
          interface_brief: "{{ interface_output.stdout[1][:500] }}"
          recent_logs: "{{ logging_output.stdout[0][:500] }}"
          graylog_link: "http://172.16.0.60:9000/search?q=source:{{ ansible_host }}&rangetype=relative&relative=600"
          grafana_link: "http://172.16.0.50:3000/d/lab-network-fabric?var-device={{ device_name }}"

    - name: "Notify | Post diagnosis to Slack"
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "🔴 *BGP Neighbor Down — Automated Diagnosis Report*"
          attachments:
            - color: "{{ 'good' if 'SUCCESS' in remediation_status else 'danger' }}"
              title: "{{ device_name }} — BGP Peer {{ bgp_peer_ip }}"
              fields:
                - title: "Root Cause Analysis"
                  value: "{{ root_cause }}"
                  short: false
                - title: "Remediation Status"
                  value: "{{ remediation_status }}"
                  short: false
                - title: "BGP Summary (excerpt)"
                  value: "```{{ bgp_output.stdout[0][:300] }}```"
                  short: false
                - title: "Recent Device Logs (excerpt)"
                  value: "```{{ logging_output.stdout[0][-300:] }}```"
                  short: false
              footer: "Triggered by Graylog event {{ graylog_event_id }}"
              actions:
                - type: button
                  text: "View in Graylog"
                  url: "{{ diagnosis_report.graylog_link }}"
                - type: button
                  text: "View in Grafana"
                  url: "{{ diagnosis_report.grafana_link }}"
      delegate_to: localhost

    - name: "Remediation | Write diagnosis to file for AWX artifact"
      ansible.builtin.copy:
        content: "{{ diagnosis_report | to_nice_json }}"
        dest: "/tmp/bgp_diagnosis_{{ device_name }}_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.json"
        mode: '0644'
      delegate_to: localhost

    - name: "Remediation | Final status"
      ansible.builtin.debug:
        msg:
          - "BGP diagnosis complete for {{ device_name }}"
          - "Root cause: {{ root_cause }}"
          - "Remediation: {{ remediation_status }}"
          - "Slack notification sent with full context"
EOF
```

### Step 2: AWX workflow — three chained job templates

```bash
cat > ~/projects/ansible-network/playbooks/remediation/backup_before_remediation.yml << 'EOF'
---
# Stage 1 of the AWX workflow — capture device state before any changes
# Runs before diagnose_bgp.yml so there is a pre-change snapshot

- name: "Workflow Stage 1 | Backup device before remediation"
  hosts: "{{ graylog_device_name | default('wan-r1') }}"
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: [remediation, backup]

  vars:
    backup_dir: "/home/ansible/backups/remediation"
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
    device_name: "{{ graylog_device_name | default(inventory_hostname) }}"

  tasks:

    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Capture running configuration"
      cisco.ios.ios_command:
        commands: ["show running-config"]
      register: running_config

    - name: "Backup | Capture full show tech-support"
      cisco.ios.ios_command:
        commands: ["show tech-support"]
        wait_for: null
        timeout: 120
      register: tech_support
      failed_when: false    # Don't fail the workflow if show tech is slow

    - name: "Backup | Write backup files"
      ansible.builtin.copy:
        content: "{{ item.content }}"
        dest: "{{ backup_dir }}/{{ device_name }}_{{ timestamp }}_{{ item.name }}.txt"
        mode: '0644'
      loop:
        - { name: running_config, content: "{{ running_config.stdout[0] }}" }
        - { name: tech_support,   content: "{{ tech_support.stdout[0] | default('') }}" }
      delegate_to: localhost

    - name: "Backup | Commit backup to Gitea"
      ansible.builtin.shell: |
        cd /home/ansible/backups/remediation && \
        git add . && \
        git commit -m "auto-backup: {{ device_name }} pre-remediation {{ timestamp }}" && \
        git push origin main || echo "Git push failed — backup saved locally"
      delegate_to: localhost
      changed_when: true
      failed_when: false

    - name: "Backup | Set backup artifact fact for downstream jobs"
      ansible.builtin.set_fact:
        backup_timestamp: "{{ timestamp }}"
        backup_path: "{{ backup_dir }}/{{ device_name }}_{{ timestamp }}_running_config.txt"
EOF
```

```bash
cat > ~/projects/ansible-network/playbooks/remediation/notify_escalation.yml << 'EOF'
---
# Stage 3 of the AWX workflow — escalation notification
# Runs after diagnose_bgp.yml — posts a final status with all context

- name: "Workflow Stage 3 | Send escalation notification"
  hosts: localhost
  gather_facts: false
  tags: [remediation, notify]

  vars:
    slack_webhook: "{{ vault_slack_webhook_url }}"
    device_name: "{{ graylog_device_name | default('unknown') }}"
    bgp_peer_ip: "{{ graylog_bgp_peer_ip | default('unknown') }}"

  tasks:

    - name: "Notify | Final escalation message"
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "📋 *Automated Remediation Workflow Complete*"
          attachments:
            - color: "#439FE0"
              fields:
                - title: "Device"
                  value: "{{ device_name }}"
                  short: true
                - title: "BGP Peer"
                  value: "{{ bgp_peer_ip }}"
                  short: true
                - title: "Workflow Steps"
                  value: "✅ Backup captured\n✅ Diagnosis run\n✅ Escalation sent"
                  short: false
                - title: "Next Steps"
                  value: >-
                    Review the diagnosis report in Slack. If auto-remediation
                    did not resolve the issue, manual investigation is required.
                    Backup is committed to Gitea.
                  short: false
              footer: "AWX Automated Remediation Workflow"

    - name: "Notify | Create Netbox journal entry for the event"
      ansible.builtin.uri:
        url: "http://172.16.0.20/api/extras/journal-entries/"
        method: POST
        headers:
          Authorization: "Token {{ vault_netbox_api_token }}"
          Content-Type: application/json
        body_format: json
        body:
          assigned_object_type: "dcim.device"
          assigned_object_id: "{{ netbox_device_id | default(1) }}"
          kind: "warning"
          comments: |
            **Automated BGP Remediation Event**
            - Time: {{ lookup('pipe', 'date') }}
            - BGP peer {{ bgp_peer_ip }} went down
            - AWX automated workflow ran: backup → diagnose → notify
            - See Graylog event {{ graylog_event_id | default('N/A') }}
        status_code: [201, 400]
      failed_when: false    # Journal entry is best-effort — don't fail if Netbox unavailable
EOF
```

### AWX workflow configuration

```bash
cat > ~/projects/ansible-network/playbooks/integration/awx_workflow_setup.yml << 'EOF'
---
# Create the BGP Remediation AWX workflow
# Usage: ansible-playbook integration/awx_workflow_setup.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt
# Requires: AWX deployed (Part 38 sets this up)

- name: "AWX | Create BGP remediation workflow"
  hosts: localhost
  gather_facts: false
  tags: [awx, workflow, integration]

  vars:
    awx_url: "http://172.16.0.30"
    awx_token: "{{ vault_awx_api_token }}"
    gitea_project_url: "http://172.16.0.30:3000/network-automation/ansible-network.git"

  tasks:

    # ── Create job templates ──────────────────────────────────────────
    - name: "AWX | Create job template: Backup Before Remediation"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/job_templates/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          name: "Remediation — Backup Device"
          description: "Stage 1: Capture device state before remediation"
          job_type: run
          playbook: "playbooks/remediation/backup_before_remediation.yml"
          ask_variables_on_launch: true
          extra_vars: |
            graylog_device_name: ""
            graylog_bgp_peer_ip: ""
            graylog_event_id: ""
        status_code: [201, 400]
      register: jt_backup

    - name: "AWX | Create job template: Diagnose BGP"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/job_templates/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          name: "Remediation — Diagnose BGP"
          description: "Stage 2: Collect diagnostics, attempt self-healing, notify"
          job_type: run
          playbook: "playbooks/remediation/diagnose_bgp.yml"
          ask_variables_on_launch: true
          ask_limit_on_launch: true
        status_code: [201, 400]
      register: jt_diagnose

    - name: "AWX | Create job template: Notify Escalation"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/job_templates/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          name: "Remediation — Notify Escalation"
          description: "Stage 3: Final status notification and Netbox journal entry"
          job_type: run
          playbook: "playbooks/remediation/notify_escalation.yml"
          ask_variables_on_launch: true
        status_code: [201, 400]
      register: jt_notify

    # ── Create workflow template ──────────────────────────────────────
    - name: "AWX | Create workflow template"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_templates/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          name: "BGP Failure — Automated Remediation"
          description: >
            Triggered by Graylog BGP Neighbor Down alert.
            Stage 1: Backup device state.
            Stage 2: Diagnose and attempt self-healing.
            Stage 3: Notify with full context.
          ask_variables_on_launch: true
          webhook_service: "github"   # Graylog uses HTTP POST — generic webhook
          variables: |
            graylog_device_name: ""
            graylog_bgp_peer_ip: ""
            graylog_event_id: ""
        status_code: [201, 400]
      register: wf_template

    # ── Link workflow nodes ───────────────────────────────────────────
    - name: "AWX | Add Stage 1 node (Backup)"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_template_nodes/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          workflow_job_template: "{{ wf_template.json.id }}"
          unified_job_template: "{{ jt_backup.json.id }}"
          all_parents_must_converge: false
        status_code: [201, 400]
      register: node_backup

    - name: "AWX | Add Stage 2 node (Diagnose)"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_template_nodes/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          workflow_job_template: "{{ wf_template.json.id }}"
          unified_job_template: "{{ jt_diagnose.json.id }}"
        status_code: [201, 400]
      register: node_diagnose

    - name: "AWX | Add Stage 3 node (Notify)"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_template_nodes/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          workflow_job_template: "{{ wf_template.json.id }}"
          unified_job_template: "{{ jt_notify.json.id }}"
        status_code: [201, 400]
      register: node_notify

    - name: "AWX | Chain: Backup → Diagnose (on success)"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_template_nodes/{{ node_backup.json.id }}/success_nodes/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          id: "{{ node_diagnose.json.id }}"
        status_code: [201, 204]

    - name: "AWX | Chain: Diagnose → Notify (on success AND failure)"
      ansible.builtin.uri:
        url: "{{ awx_url }}/api/v2/workflow_job_template_nodes/{{ node_diagnose.json.id }}/{{ item }}_nodes/"
        method: POST
        headers:
          Authorization: "Bearer {{ awx_token }}"
          Content-Type: application/json
        body_format: json
        body:
          id: "{{ node_notify.json.id }}"
        status_code: [201, 204]
      loop: [success, failure]
      # Notify runs whether diagnosis succeeded or failed —
      # the team needs to know either way

    - name: "AWX | Report workflow created"
      ansible.builtin.debug:
        msg:
          - "AWX workflow created: BGP Failure — Automated Remediation"
          - "Workflow ID: {{ wf_template.json.id }}"
          - "Webhook URL: {{ awx_url }}/api/v2/workflow_job_templates/{{ wf_template.json.id }}/launch/"
          - "Add this URL to Graylog BGP Neighbor Down alert notification"
          - ""
          - "Workflow: Backup → Diagnose (self-heal if possible) → Notify"
          - "Notify stage runs on BOTH success and failure of Diagnose"
EOF
```

### Wire Graylog to the AWX workflow

```bash
# Update the Graylog BGP alert notification to call the AWX workflow
# (updates the notification created in Part 39)

AWX_WORKFLOW_URL="http://172.16.0.30/api/v2/workflow_job_templates/${WORKFLOW_ID}/launch/"
AWX_WEBHOOK_TOKEN="${VAULT_AWX_API_TOKEN}"

# Create AWX HTTP notification in Graylog
curl -s -X POST \
  -u "admin:${GRAYLOG_ADMIN_PASSWORD}" \
  -H "X-Requested-By: ansible" \
  -H "Content-Type: application/json" \
  "http://172.16.0.60:9000/api/events/notifications" \
  -d "{
    \"title\": \"AWX — BGP Remediation Workflow\",
    \"description\": \"Triggers AWX self-healing workflow on BGP failure\",
    \"config\": {
      \"type\": \"http-notification-v1\",
      \"url\": \"${AWX_WORKFLOW_URL}\",
      \"method\": \"POST\",
      \"content_type\": \"application/json\",
      \"headers\": {\"Authorization\": \"Bearer ${AWX_WEBHOOK_TOKEN}\"},
      \"body_template\": \"{\\\"extra_vars\\\": {\\\"graylog_device_name\\\": \\\"\${source.device_name}\\\", \\\"graylog_bgp_peer_ip\\\": \\\"\${source.bgp_peer_ip}\\\", \\\"graylog_event_id\\\": \\\"\${event.id}\\\"}}\"
    }
  }"
```

---

## 40.5 — Integration 4: Unified Grafana Dashboard

### Before

```
Grafana has separate dashboards:
  - Node Exporter Full (VM metrics)
  - Lab Network Fabric (network device metrics — Part 38)
  - SNMP Stats (imported community dashboard)
  - Loki Logs (imported community dashboard)

Finding the full picture requires navigating between four dashboards.
Correlating a log event with a metric spike requires manual mental mapping.
```

### After

```
One dashboard — "Lab Operations Centre" — six rows:
  Row 1: Stack health (are all monitoring services up?)
  Row 2: Network device health (Prometheus SNMP metrics)
  Row 3: BGP and routing state table
  Row 4: Infrastructure VM metrics (Node Exporter)
  Row 5: Recent alerts (from Prometheus AlertManager API)
  Row 6: Log stream (Loki — correlated with metric rows above)
```

```bash
cat > ~/projects/ansible-network/playbooks/integration/grafana_ops_dashboard.yml << 'EOF'
---
# Create the unified Lab Operations Centre dashboard
# Usage: ansible-playbook integration/grafana_ops_dashboard.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Grafana | Create unified operations dashboard"
  hosts: localhost
  gather_facts: false
  tags: [grafana, integration, dashboard]

  vars:
    grafana_url: "http://172.16.0.50:3000"
    grafana_user: admin
    grafana_password: "{{ vault_grafana_admin_password }}"

  tasks:

    - name: "Grafana | Create Lab Operations Centre dashboard"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/dashboards/db"
        method: POST
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
        body_format: json
        body:
          overwrite: true
          dashboard:
            title: "Lab Operations Centre"
            uid: "lab-ops-centre"
            tags: [lab, operations, unified]
            refresh: "30s"
            time:
              from: now-3h
              to: now

            templating:
              list:
                - name: device
                  type: query
                  datasource: Prometheus
                  query: "label_values(up{category='network'}, device)"
                  multi: true
                  includeAll: true

            panels:

              # ── Row 1: Stack health ─────────────────────────────────
              - type: row
                title: "🔭 Monitoring Stack Health"
                gridPos: { x: 0, y: 0, w: 24, h: 1 }
                collapsed: false

              - type: stat
                title: "Prometheus"
                gridPos: { x: 0, y: 1, w: 3, h: 3 }
                targets:
                  - expr: "up{job='prometheus'}"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: red,   value: 0 }
                      - { color: green, value: 1 }
                  mappings:
                    - type: value
                      options:
                        "0": { text: "DOWN", color: red }
                        "1": { text: "UP",   color: green }

              - type: stat
                title: "Zabbix"
                gridPos: { x: 3, y: 1, w: 3, h: 3 }
                targets:
                  - expr: "up{job='node_exporter', device='monitoring'}"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: red, value: 0 }
                      - { color: green, value: 1 }

              - type: stat
                title: "Graylog"
                gridPos: { x: 6, y: 1, w: 3, h: 3 }
                targets:
                  - expr: "up{job='node_exporter', device='logging'}"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: red, value: 0 }
                      - { color: green, value: 1 }

              - type: stat
                title: "Netbox"
                gridPos: { x: 9, y: 1, w: 3, h: 3 }
                targets:
                  - expr: "up{job='node_exporter', device='netbox'}"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: red, value: 0 }
                      - { color: green, value: 1 }

              - type: stat
                title: "Active Alerts"
                gridPos: { x: 12, y: 1, w: 4, h: 3 }
                targets:
                  - expr: "count(ALERTS{alertstate='firing'}) or vector(0)"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: green,  value: 0 }
                      - { color: yellow, value: 1 }
                      - { color: red,    value: 5 }

              - type: stat
                title: "Network Devices Up"
                gridPos: { x: 16, y: 1, w: 4, h: 3 }
                targets:
                  - expr: "count(up{category='network'} == 1) or vector(0)"
                    legendFormat: ""
                options:
                  colorMode: background
                  thresholds:
                    steps:
                      - { color: red,   value: 0 }
                      - { color: yellow, value: 5 }
                      - { color: green, value: 7 }

              # ── Row 2: Network device metrics ───────────────────────
              - type: row
                title: "🌐 Network Device Metrics"
                gridPos: { x: 0, y: 4, w: 24, h: 1 }

              - type: timeseries
                title: "Interface Traffic — $device (bps)"
                gridPos: { x: 0, y: 5, w: 14, h: 7 }
                targets:
                  - expr: "rate(ifHCInOctets{category='network', device=~'$device'}[5m]) * 8"
                    legendFormat: "{{device}} {{ifName}} ▼"
                  - expr: "rate(ifHCOutOctets{category='network', device=~'$device'}[5m]) * 8"
                    legendFormat: "{{device}} {{ifName}} ▲"
                fieldConfig:
                  defaults:
                    unit: bps

              - type: table
                title: "Device CPU % (5-min avg)"
                gridPos: { x: 14, y: 5, w: 10, h: 7 }
                targets:
                  - expr: "avgBusy5{category='network'}"
                    legendFormat: "{{device}}"
                    instant: true
                transformations:
                  - id: organize
                    options:
                      renameByName:
                        device: Device
                        Value: CPU %
                fieldConfig:
                  defaults:
                    thresholds:
                      steps:
                        - { color: green,  value: 0 }
                        - { color: yellow, value: 75 }
                        - { color: red,    value: 90 }

              # ── Row 3: BGP state ────────────────────────────────────
              - type: row
                title: "🔁 BGP and Routing"
                gridPos: { x: 0, y: 12, w: 24, h: 1 }

              - type: table
                title: "BGP Peer States (6=Established)"
                gridPos: { x: 0, y: 13, w: 12, h: 6 }
                targets:
                  - expr: "bgpPeerState{category='network'}"
                    legendFormat: "{{device}} → {{bgpPeerRemoteAddr}}"
                    instant: true
                fieldConfig:
                  defaults:
                    thresholds:
                      steps:
                        - { color: red,   value: 0 }
                        - { color: green, value: 6 }

              - type: stat
                title: "BGP Sessions Established"
                gridPos: { x: 12, y: 13, w: 6, h: 6 }
                targets:
                  - expr: "count(bgpPeerState{category='network'} == 6) or vector(0)"
                    legendFormat: ""
                options:
                  colorMode: background

              # ── Row 4: VM health ────────────────────────────────────
              - type: row
                title: "🖥️ Infrastructure VM Metrics"
                gridPos: { x: 0, y: 19, w: 24, h: 1 }

              - type: timeseries
                title: "VM Memory Utilisation %"
                gridPos: { x: 0, y: 20, w: 12, h: 6 }
                targets:
                  - expr: "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100"
                    legendFormat: "{{device}}"
                fieldConfig:
                  defaults:
                    unit: percent
                    max: 100
                    thresholds:
                      steps:
                        - { color: green,  value: 0 }
                        - { color: yellow, value: 80 }
                        - { color: red,    value: 90 }

              - type: timeseries
                title: "VM Disk Usage % (root filesystem)"
                gridPos: { x: 12, y: 20, w: 12, h: 6 }
                targets:
                  - expr: "(1 - (node_filesystem_avail_bytes{mountpoint='/'} / node_filesystem_size_bytes{mountpoint='/'})) * 100"
                    legendFormat: "{{device}}"
                fieldConfig:
                  defaults:
                    unit: percent
                    max: 100
                    thresholds:
                      steps:
                        - { color: green,  value: 0 }
                        - { color: yellow, value: 75 }
                        - { color: red,    value: 85 }

              # ── Row 5: Active alerts summary ────────────────────────
              - type: row
                title: "🚨 Active Alerts"
                gridPos: { x: 0, y: 26, w: 24, h: 1 }

              - type: table
                title: "Firing Alerts"
                gridPos: { x: 0, y: 27, w: 24, h: 5 }
                targets:
                  - expr: "ALERTS{alertstate='firing'}"
                    legendFormat: "{{alertname}} | {{device}} | {{severity}}"
                    instant: true
                transformations:
                  - id: organize
                    options:
                      renameByName:
                        alertname: Alert
                        device: Device
                        severity: Severity
                        Value: Active

              # ── Row 6: Log stream ───────────────────────────────────
              - type: row
                title: "📋 Recent Log Events"
                gridPos: { x: 0, y: 32, w: 24, h: 1 }

              - type: logs
                title: "Network Device Syslog (last 30 min)"
                gridPos: { x: 0, y: 33, w: 24, h: 7 }
                datasource: Loki
                targets:
                  - expr: '{job="syslog", category="infrastructure"}'
                    legendFormat: ""
                options:
                  showTime: true
                  showLabels: true
                  wrapLogMessage: false
                  sortOrder: Descending

        status_code: 200
      register: dashboard_result

    - name: "Dashboard | Report"
      ansible.builtin.debug:
        msg:
          - "Lab Operations Centre dashboard created"
          - "URL: http://172.16.0.50:3000/d/lab-ops-centre"
          - ""
          - "Dashboard has 6 rows:"
          - "  Row 1: Monitoring stack health (Prometheus/Zabbix/Graylog/Netbox)"
          - "  Row 2: Network device interface traffic and CPU table"
          - "  Row 3: BGP peer state table"
          - "  Row 4: VM memory and disk utilisation"
          - "  Row 5: Active alerts from Prometheus AlertManager"
          - "  Row 6: Real-time syslog from Loki"
EOF
```

---

## 40.6 — End-to-End Integration Test

With all four integrations in place, run the full test scenario — a simulated network failure that exercises every integration simultaneously.

```bash
cat > ~/projects/ansible-network/playbooks/integration/test_full_stack.yml << 'EOF'
---
# Full-stack integration test
# Simulates a BGP failure and validates every integration fires correctly
# Usage: ansible-playbook integration/test_full_stack.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Integration Test | Full stack failure simulation"
  hosts: localhost
  gather_facts: false
  tags: [integration, test]

  vars:
    test_device: "wan-r1"
    test_device_ip: "172.16.0.11"

  tasks:

    - name: "Test | Pre-check — verify all integrations are reachable"
      ansible.builtin.uri:
        url: "{{ item.url }}"
        status_code: 200
        timeout: 5
      loop:
        - { name: Netbox,      url: "http://172.16.0.20/api/" }
        - { name: Zabbix,      url: "http://172.16.0.40/" }
        - { name: Prometheus,  url: "http://172.16.0.50:9090/-/healthy" }
        - { name: Grafana,     url: "http://172.16.0.50:3000/api/health" }
        - { name: Graylog,     url: "http://172.16.0.60:9000/api/system/lbstatus" }
        - { name: AWX,         url: "http://172.16.0.30/api/v2/ping/" }
      loop_control:
        label: "{{ item.name }}"

    - name: "Test | Record baseline — BGP state before test"
      ansible.builtin.uri:
        url: "http://172.16.0.50:9090/api/v1/query?query=bgpPeerState%7Bdevice%3D%22wan-r1%22%7D"
      register: bgp_baseline

    - name: "Test | Display baseline BGP state"
      ansible.builtin.debug:
        msg: "Baseline BGP peers: {{ bgp_baseline.json.data.result | map(attribute='metric.bgpPeerRemoteAddr') | list }}"

    - name: "Test | Step 1 — shut BGP neighbor on wan-r1"
      cisco.ios.ios_config:
        lines:
          - "neighbor 10.0.0.1 shutdown"
        parents: "router bgp 65001"
      delegate_to: "{{ test_device }}"
      connection: network_cli

    - name: "Test | Wait 90 seconds for events to propagate through stack"
      ansible.builtin.pause:
        seconds: 90
        prompt: |
          Waiting for events to propagate:
            t+0s:  BGP neighbor shutdown on wan-r1
            t+10s: Cisco IOS logs BGP ADJCHANGE message
            t+15s: Graylog receives syslog, pipeline detects BGP down
            t+30s: Prometheus scrape detects bgpPeerState != 6
            t+60s: Graylog alert evaluates BGP neighbor down condition
            t+90s: AWX workflow should have started
          Press ENTER to continue validation checks

    # ── Validate each integration fired ──────────────────────────────
    - name: "Validate | Check Graylog received BGP event"
      ansible.builtin.uri:
        url: >
          http://172.16.0.60:9000/api/search/universal/relative
          ?query=event_type:bgp_neighbor_down+AND+source:{{ test_device_ip }}
          &range=120&limit=5
        method: GET
        user: admin
        password: "{{ vault_graylog_admin_password }}"
        force_basic_auth: true
        headers:
          X-Requested-By: ansible
      register: graylog_events

    - name: "Validate | Graylog result"
      ansible.builtin.debug:
        msg: >
          Graylog BGP events found: {{ graylog_events.json.total_results }}
          (expected: >= 1)

    - name: "Validate | Check Prometheus detected BGP peer down"
      ansible.builtin.uri:
        url: >
          http://172.16.0.50:9090/api/v1/query
          ?query=bgpPeerState%7Bdevice%3D%22wan-r1%22%7D%20!%3D%206
      register: prom_bgp

    - name: "Validate | Prometheus result"
      ansible.builtin.debug:
        msg: "Prometheus BGP peer != 6: {{ prom_bgp.json.data.result | length }} peer(s) (expected: >= 1)"

    - name: "Validate | Check Prometheus alert is firing"
      ansible.builtin.uri:
        url: "http://172.16.0.50:9090/api/v1/alerts"
      register: prom_alerts

    - name: "Validate | Alert status"
      ansible.builtin.debug:
        msg: >
          Firing alerts:
          {{ prom_alerts.json.data.alerts
             | selectattr('state', 'eq', 'firing')
             | map(attribute='labels.alertname')
             | list }}

    - name: "Validate | Check AWX workflow was triggered"
      ansible.builtin.uri:
        url: "http://172.16.0.30/api/v2/workflow_jobs/?order_by=-created&page_size=5"
        headers:
          Authorization: "Bearer {{ vault_awx_api_token }}"
      register: awx_jobs

    - name: "Validate | AWX workflow result"
      ansible.builtin.debug:
        msg: >
          Recent AWX workflows:
          {{ awx_jobs.json.results
             | map(attribute='name')
             | list }}

    # ── Restore ───────────────────────────────────────────────────────
    - name: "Test | Step 2 — restore BGP neighbor"
      cisco.ios.ios_config:
        lines:
          - "no neighbor 10.0.0.1 shutdown"
        parents: "router bgp 65001"
      delegate_to: "{{ test_device }}"
      connection: network_cli

    - name: "Test | Wait 60 seconds for BGP to reconverge"
      ansible.builtin.pause:
        seconds: 60

    - name: "Test | Verify BGP reconverged"
      ansible.builtin.uri:
        url: >
          http://172.16.0.50:9090/api/v1/query
          ?query=bgpPeerState%7Bdevice%3D%22wan-r1%22%7D
      register: bgp_post_restore

    - name: "Test | Final status"
      ansible.builtin.debug:
        msg:
          - "════════════════════════════════════════════════════════════"
          - " Full Stack Integration Test Complete"
          - "════════════════════════════════════════════════════════════"
          - " BGP failure detected by Graylog: {{ graylog_events.json.total_results >= 1 }}"
          - " Prometheus alert fired:          {{ prom_alerts.json.data.alerts | selectattr('state','eq','firing') | list | length > 0 }}"
          - " AWX workflow triggered:          {{ awx_jobs.json.results | length > 0 }}"
          - " BGP reconverged after restore:   {{ bgp_post_restore.json.data.result | length > 0 }}"
          - ""
          - " Dashboard: http://172.16.0.50:3000/d/lab-ops-centre"
          - " Graylog:   http://172.16.0.60:9000"
          - " AWX Jobs:  http://172.16.0.30/#/jobs"
          - "════════════════════════════════════════════════════════════"
EOF
```

---

## 40.7 — Checklist

```
[ ] Integration 1: Netbox → Zabbix sync
    [ ] netbox_to_zabbix_sync.yml created and tested:
        [ ] Event-driven: -e "netbox_device_id=X netbox_event=created"
            registers device in Zabbix within seconds
        [ ] Scheduled: -e "sync_all=true" syncs all active devices
        [ ] Device in Zabbix has correct template from platform_template_map
        [ ] Device in Zabbix has correct host groups from role_group_map
        [ ] Netbox custom field $NETBOX_ID stored as Zabbix host macro
    [ ] AWX job templates created:
        [ ] "Netbox → Zabbix Sync (Device Change)" — webhook-enabled
        [ ] "Netbox → Zabbix Sync (Scheduled)" — nightly at 02:00
    [ ] Netbox webhook from Part 36 updated to call AWX job template ID
    [ ] End-to-end test: add device in Netbox, verify it appears in
        Zabbix within 30 seconds without any manual steps

[ ] Integration 2: Netbox → Prometheus SD
    [ ] file_sd_configs section added to prometheus.yml
    [ ] /opt/observability/prometheus/sd_targets/ directory created
    [ ] netbox_to_prometheus_sync.yml created and tested:
        [ ] Writes network_devices_ios.json, _nxos.json, _panos.json
        [ ] JSON files contain correct target IP arrays with labels
    [ ] Prometheus picks up new targets within 15 seconds of file change
        (verify via Prometheus UI → Targets)
    [ ] AWX job templates created:
        [ ] "Netbox → Prometheus Sync (Device Change)" — webhook-enabled
        [ ] "Netbox → Prometheus Sync (Scheduled)" — nightly at 02:05

[ ] Integration 3: Graylog → AWX self-healing
    [ ] Three remediation playbooks created:
        [ ] backup_before_remediation.yml
        [ ] diagnose_bgp.yml
        [ ] notify_escalation.yml
    [ ] AWX job templates created (3 templates):
        [ ] "Remediation — Backup Device"
        [ ] "Remediation — Diagnose BGP"
        [ ] "Remediation — Notify Escalation"
    [ ] AWX workflow created: "BGP Failure — Automated Remediation"
        [ ] Workflow has 3 nodes chained in order
        [ ] Notify node linked from BOTH success and failure of Diagnose
    [ ] Graylog BGP Neighbor Down alert updated with AWX webhook URL
    [ ] Test: shut BGP on wan-r1 → AWX workflow launches automatically
        [ ] Stage 1 completes: backup committed to Gitea
        [ ] Stage 2 completes: Slack message with BGP summary received
        [ ] Stage 3 completes: Netbox journal entry created
        [ ] Restore BGP → workflow does not re-fire (grace period working)

[ ] Integration 4: Unified Grafana dashboard
    [ ] grafana_ops_dashboard.yml ran successfully
    [ ] "Lab Operations Centre" dashboard accessible at
        http://172.16.0.50:3000/d/lab-ops-centre
    [ ] Row 1 panels showing correct green/red status:
        [ ] Prometheus: green
        [ ] Zabbix (monitoring VM): green
        [ ] Graylog (logging VM): green
        [ ] Netbox: green
        [ ] Active Alerts: correct count
        [ ] Network Devices Up: 7 (or current count)
    [ ] Row 2: Interface traffic timeseries has data for wan-r1
    [ ] Row 3: BGP peer table populated with all established sessions
    [ ] Row 4: VM memory and disk panels showing data for all 5 VMs
    [ ] Row 5: Firing alerts table (empty when lab is healthy)
    [ ] Row 6: Loki log panel showing recent syslog entries

[ ] End-to-end integration test
    [ ] test_full_stack.yml ran to completion
    [ ] Pre-check: all 6 services reachable
    [ ] BGP shutdown on wan-r1 triggers events in:
        [ ] Graylog (event_type=bgp_neighbor_down in search)
        [ ] Prometheus (BGPPeerDown alert firing)
        [ ] AlertManager (alert visible at :9093)
        [ ] AWX (workflow job visible in Jobs list)
    [ ] Slack notification received with diagnosis content
    [ ] BGP restore: all alerts clear within 3 minutes
    [ ] Grafana Operations Centre shows clean state after restore

[ ] All playbooks committed to Gitea:
    [ ] playbooks/integration/netbox_to_zabbix_sync.yml
    [ ] playbooks/integration/netbox_to_prometheus_sync.yml
    [ ] playbooks/integration/awx_workflow_setup.yml
    [ ] playbooks/integration/grafana_ops_dashboard.yml
    [ ] playbooks/integration/test_full_stack.yml
    [ ] playbooks/remediation/backup_before_remediation.yml
    [ ] playbooks/remediation/diagnose_bgp.yml
    [ ] playbooks/remediation/notify_escalation.yml

[ ] Final snapshot of all VMs:
    ansible-playbook lifecycle/snapshot.yml \
      -e "snap_name=post-integration \
          snap_description='Full stack integrated and tested Part 40'"
```

---

*The stack is now a system. A device added to Netbox appears in Zabbix and Prometheus within seconds. A BGP failure triggers an Ansible workflow that backs up the device, diagnoses the failure, attempts remediation, and posts a fully contextual Slack notification — before most engineers would have finished opening their SSH client. Grafana shows the complete operational picture: metrics from Prometheus, logs from Loki, and alert state from AlertManager in a single view. Each tool still works independently, and each integration makes the whole more capable than the sum of its parts. Parts 41 and 42 take this fully operational stack and put it through its paces against the actual Containerlab fabric before the capstone project ties everything together from scratch.*

*Next up: **Part 41 — Monitoring the Containerlab Fabric***
