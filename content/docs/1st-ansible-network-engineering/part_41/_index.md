---
draft: true
title: '41 - Monitoring the Containerlab Fabric'
description: "Part 41 of my Ansible learning geared towards Network Engineering."
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

## Part 41: Monitoring the Containerlab Fabric

> *Parts 37 through 40 built the monitoring stack and proved the integrations work. Part 41 is where the two threads of this guide fully merge — the Containerlab network fabric from Parts 1 through 32 and the observability platform from Parts 37 through 40. The result is a lab where every device is monitored, every interface state change is detected within 60 seconds across three tools simultaneously, and a single Grafana dashboard shows the health of the entire fabric. This part focuses on what the previous parts didn't specifically cover: the Containerlab fabric has different characteristics than generic network devices, and those differences require specific configuration choices. Everything else is orchestration — running the right playbooks in the right order.*

---

## 41.1 — What Parts 37–40 Already Cover vs What's New Here

```
Already covered and working (run these, don't rewrite them):

  Part 37: configure_snmp.yml     — SNMP v2c + v3 on IOS-XE, NX-OS, PAN-OS
  Part 37: register_hosts.yml     — Zabbix host registration + template linking
  Part 38: deploy.yml             — Prometheus + SNMP Exporter running
  Part 38: install_exporters.yml  — Node Exporter on all VMs
  Part 39: configure_syslog.yml   — Syslog forwarding to Graylog
  Part 39: configure_streams.yml  — Graylog stream routing
  Part 39: configure_pipelines.yml — Graylog parsing + Netbox enrichment
  Part 40: netbox_to_zabbix_sync.yml — Netbox → Zabbix sync
  Part 40: netbox_to_prometheus_sync.yml — Netbox → Prometheus SD

Gaps specific to the Containerlab fabric:

  1. Containerlab device quirks — vrnetlab containers expose SNMP
     differently than physical/VM devices. Management interface naming,
     SNMP polling paths, and syslog source IPs need adjustment.

  2. BGP fabric topology awareness — the lab runs eBGP between
     specific AS numbers. Prometheus metrics and Grafana panels need
     to be aware of the expected BGP peer relationships so alerts
     only fire for unexpected drops, not planned state changes.

  3. Interface naming normalisation — IOS-XE uses GigabitEthernetX,
     NX-OS uses EthernetX/Y, PAN-OS uses ethernetX/Y. Grafana panels
     that filter by interface name need regex that covers all three.

  4. Fabric-specific Grafana panels — the Lab Operations Centre from
     Part 40 shows generic network device metrics. This part adds
     fabric-topology-aware panels: BGP AS path visibility,
     east-west traffic across the spine, and the specific
     dist-01 → leaf-01/leaf-02 interface pairs.

  5. Syslog rate baseline — a freshly started Containerlab topology
     generates significant syslog noise (interface flaps, BGP
     reconvergence). Graylog needs a suppression period rule and
     a baseline to distinguish normal startup noise from real alerts.

  6. Single onboarding playbook — a master playbook that brings a
     freshly started Containerlab topology to fully monitored state
     in one run, filling the gaps above alongside the Part 37–40 plays.
```

---

## 41.2 — Containerlab Fabric Topology Reminder

```
WAN Edge:
  wan-r1   (172.16.0.11)  AS 65001  eBGP → panos-fw01
  wan-r2   (172.16.0.12)  AS 65002  eBGP → panos-fw01

Firewall:
  panos-fw01 (172.16.0.51)          eBGP → wan-r1, wan-r2, dist-01

Distribution:
  dist-01  (172.16.0.21)  AS 65010  eBGP → panos-fw01
                                    iBGP → spine-01

Spine:
  spine-01 (172.16.0.31)  AS 65010  iBGP → dist-01
                                    eBGP → leaf-01, leaf-02

Leaf:
  leaf-01  (172.16.0.33)  AS 65020  eBGP → spine-01
  leaf-02  (172.16.0.34)  AS 65030  eBGP → spine-01

Expected BGP sessions (total: 8):
  wan-r1   ↔ panos-fw01
  wan-r2   ↔ panos-fw01
  panos-fw01 ↔ dist-01
  dist-01  ↔ spine-01  (iBGP)
  spine-01 ↔ leaf-01
  spine-01 ↔ leaf-02

Interface pairs to monitor (critical paths):
  dist-01  Gi1  ↔  panos-fw01 (uplink to firewall)
  dist-01  Gi2  ↔  spine-01   (distribution to spine)
  spine-01 Eth1/1 ↔ dist-01
  spine-01 Eth1/2 ↔ leaf-01
  spine-01 Eth1/3 ↔ leaf-02
```

---

## 41.3 — Gap 1: Containerlab SNMP Quirks

vrnetlab containers run inside Docker on the network-lab VM (`172.16.0.10`). From the perspective of the monitoring stack, SNMP polls go to the device management IPs (`172.16.0.11` etc.) — but the source IP that Graylog and Prometheus see for syslog and SNMP responses may need verifying.

```bash
# From the monitoring VM — verify SNMP is reachable and responding
# (run these before assuming the Part 37 config is complete)

for device_ip in 172.16.0.11 172.16.0.12 172.16.0.21 172.16.0.31 172.16.0.33 172.16.0.34 172.16.0.51; do
  echo -n "Testing SNMP on ${device_ip}: "
  result=$(docker exec zabbix-server snmpget \
    -v 2c \
    -c lab-monitor-ro \
    -t 5 \
    "${device_ip}" \
    1.3.6.1.2.1.1.1.0 2>&1)
  if echo "${result}" | grep -q "STRING"; then
    echo "OK — $(echo "${result}" | grep -o 'STRING:.*' | head -1)"
  else
    echo "FAIL — ${result}"
  fi
done

# Common Containerlab SNMP issues and fixes:

# Issue 1: CSR1000v (wan-r1, wan-r2, dist-01) management interface
# is GigabitEthernet1 — SNMP community ACL must NOT restrict to
# a specific interface or the poll will fail from the monitoring VM
# Fix: ensure snmp-server community lab-monitor-ro RO  (no ACL suffix)

# Issue 2: N9Kv (spine-01) SNMP uses management VRF
# snmpget must specify the VRF context — but the monitoring VM
# polls via the routed management IP (172.16.0.31) which does NOT
# use the management VRF. Verify the correct config:
ssh ansible@172.16.0.31  # spine-01
# show snmp host          → should show 172.16.0.40 (zabbix) in global VRF
# show run | inc snmp     → logging server line should NOT have use-vrf management
#                           if the Zabbix VM reaches spine-01 via global routing

# Issue 3: PAN-OS SNMP source IP
# PAN-OS sends SNMP responses from the management interface IP
# Verify the management profile allows SNMP from monitoring VM subnet:
# Device → Setup → Interfaces → Management → Management Profile
# Should include SNMP checked with Permitted IPs: 172.16.0.0/24
```

### Containerlab-specific SNMP fix playbook

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/zabbix/containerlab_snmp_fixes.yml << 'EOF'
---
# Containerlab-specific SNMP fixes not covered in Part 37
# Addresses vrnetlab container quirks for reliable SNMP polling

# ── CSR1000v SNMP ACL removal ────────────────────────────────────────
- name: "SNMP Fix | Ensure CSR1000v SNMP has no ACL restriction"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: [snmp, fix, ios]

  vars:
    snmp_community_ro: "{{ vault_snmp_community_ro }}"
    zabbix_ip: "172.16.0.40"
    prometheus_ip: "172.16.0.50"

  tasks:

    - name: "SNMP | Remove any ACL-restricted community and re-add clean"
      cisco.ios.ios_config:
        lines:
          # Remove the ACL-restricted version if it exists
          - "no snmp-server community {{ snmp_community_ro }} RO 10"
          - "no snmp-server community {{ snmp_community_ro }} RO SNMP_ACL"
          # Add clean version — no ACL, monitoring VMs reach device via routing
          - "snmp-server community {{ snmp_community_ro }} RO"
          # Explicit trap destination for both Zabbix and Prometheus snmptrapd
          - "snmp-server host {{ zabbix_ip }} version 2c {{ snmp_community_ro }}"
          # Ensure management source interface is not restricted
          - "no snmp-server source-interface"
        save_when: changed

    - name: "SNMP | Verify polling works from Zabbix"
      ansible.builtin.command:
        cmd: >
          docker exec zabbix-server
          snmpget -v 2c -c {{ snmp_community_ro }} -t 5
          {{ ansible_host }} 1.3.6.1.2.1.1.3.0
      delegate_to: monitoring
      register: snmp_test
      changed_when: false
      failed_when: "'Timeticks' not in snmp_test.stdout"

    - name: "SNMP | Report result"
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} SNMP OK — {{ snmp_test.stdout }}"

# ── N9Kv SNMP VRF fix ────────────────────────────────────────────────
- name: "SNMP Fix | N9Kv SNMP source routing"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tags: [snmp, fix, nxos]

  vars:
    snmp_community_ro: "{{ vault_snmp_community_ro }}"
    zabbix_ip: "172.16.0.40"

  tasks:

    - name: "SNMP | Check if snmp-server host uses management VRF"
      cisco.nxos.nxos_command:
        commands: ["show snmp host"]
      register: snmp_host_output

    - name: "SNMP | Remove management VRF restriction if present"
      cisco.nxos.nxos_config:
        lines:
          - "no snmp-server host {{ zabbix_ip }} traps version 2c {{ snmp_community_ro }} use-vrf management"
          - "snmp-server host {{ zabbix_ip }} traps version 2c {{ snmp_community_ro }}"
        save_when: changed
      when: "'use-vrf management' in snmp_host_output.stdout[0]"

    - name: "SNMP | Verify N9Kv polling"
      ansible.builtin.command:
        cmd: >
          docker exec zabbix-server
          snmpget -v 2c -c {{ snmp_community_ro }} -t 5
          {{ ansible_host }} 1.3.6.1.2.1.1.1.0
      delegate_to: monitoring
      register: snmp_nxos_test
      changed_when: false

    - name: "SNMP | Report"
      ansible.builtin.debug:
        msg: "spine-01 SNMP: {{ snmp_nxos_test.stdout[:80] }}"
EOF
```

---

## 41.4 — Gap 2: BGP Topology Awareness in Prometheus

The Part 38 alert rules fire when `bgpPeerState != 6` for any peer. In a freshly started or recently restarted Containerlab topology, BGP sessions take time to converge and will legitimately be in non-Established states. The alert rule needs a topology-aware `for` duration and an exclusion for planned maintenance windows.

More importantly, Prometheus should track the *expected* number of BGP sessions, not just the state of sessions it happens to find. If a session disappears entirely (peer removed from config), `bgpPeerState` won't appear in metrics at all — and a missing metric won't trigger an alert.

```bash
cat > /opt/observability/prometheus/rules/fabric_bgp.yml << 'EOF'
# Fabric-specific BGP alert rules
# Extends the generic network.yml rules from Part 38
# with topology-aware thresholds for the Containerlab fabric

groups:
  - name: containerlab_fabric_bgp
    interval: 60s
    rules:

      # ── Expected BGP session count ───────────────────────────────
      # The fabric should always have exactly 6 established sessions
      # across all devices. If the count drops, something is wrong.
      - alert: FabricBGPSessionCountLow
        expr: |
          count(bgpPeerState{category="network"} == 6) < 6
        for: 3m
        labels:
          severity: high
          category: network
          fabric: containerlab
        annotations:
          summary: "Fabric BGP session count below expected"
          description: >
            Expected 6 established BGP sessions across the fabric.
            Currently: {{ $value }} established.
            Missing sessions indicate a routing topology failure.
            Check: dist-01↔spine-01, spine-01↔leaf-01, spine-01↔leaf-02,
            wan-r1↔panos-fw01, wan-r2↔panos-fw01, panos-fw01↔dist-01.

      # ── Spine connectivity — critical path ────────────────────────
      # spine-01 has 3 eBGP sessions. If spine-01 loses ALL of them,
      # east-west traffic between leaves is completely broken.
      - alert: SpineTotalConnectivityLoss
        expr: |
          count(bgpPeerState{device="spine-01", category="network"} == 6) == 0
        for: 1m
        labels:
          severity: critical
          category: network
          fabric: containerlab
        annotations:
          summary: "spine-01 has lost ALL BGP sessions"
          description: >
            spine-01 has no established BGP sessions.
            East-west traffic between leaf switches is completely broken.
            Immediate investigation required.

      # ── Leaf isolation ────────────────────────────────────────────
      # If a leaf loses its only uplink to spine, it's isolated.
      - alert: LeafIsolated
        expr: |
          count by (device) (bgpPeerState{device=~"leaf-0[12]", category="network"} == 6) == 0
        for: 2m
        labels:
          severity: high
          category: network
          fabric: containerlab
        annotations:
          summary: "{{ $labels.device }} is isolated — no BGP uplinks"
          description: >
            {{ $labels.device }} has lost its BGP session to spine-01.
            All traffic in/out of this leaf segment is dropped.

      # ── Startup noise suppression ─────────────────────────────────
      # After Containerlab restart, BGP takes 30-90s to converge.
      # This recording rule tracks time since last topology start
      # (approximated by first SNMP data point after gap).
      # Alerts with 'for: 3m' above naturally suppress startup noise.

      # ── BGP reconvergence time tracking ──────────────────────────
      # Record how long BGP sessions spend in non-Established state.
      # Useful for SLA reporting and capacity planning.
      - record: fabric_bgp_convergence_seconds
        expr: |
          (time() - bgpPeerFsmEstablishedTime{category="network"})
          * on(device, bgpPeerRemoteAddr) (bgpPeerState == 6)
        labels:
          fabric: containerlab
EOF

# Reload Prometheus to pick up new rules
curl -s -X POST http://172.16.0.50:9090/-/reload
echo "Prometheus rules reloaded"

# Verify rules loaded
curl -s http://172.16.0.50:9090/api/v1/rules \
  | python3 -m json.tool \
  | grep -A2 '"name": "containerlab_fabric_bgp"'
```

---

## 41.5 — Gap 3: Interface Naming Normalisation

The three platforms in the lab use different interface naming conventions. Grafana panels and Graylog pipeline rules that parse interface names need to handle all three.

```
IOS-XE (CSR1000v):    GigabitEthernet1, GigabitEthernet2, GigabitEthernet3
NX-OS  (N9Kv):        Ethernet1/1, Ethernet1/2, Ethernet1/3
PAN-OS (PA-VM):       ethernet1/1, ethernet1/2, ethernet1/3
```

```bash
# Add a Graylog pipeline rule that normalises interface names
# across all three platforms into a consistent short form
# e.g. GigabitEthernet2 → Gi2, Ethernet1/2 → Eth1/2, ethernet1/1 → eth1/1

cat > /tmp/normalise_interface_rule.json << 'EOF'
{
  "title": "Normalise Interface Name",
  "description": "Creates interface_name_short from various platform naming conventions",
  "source": "rule \"Normalise Interface Name\"\nwhen\n  has_field(\"interface_name\")\nthen\n  let raw = to_string($message.interface_name);\n  // Cisco IOS-XE long form → short form\n  let short = replace(raw, \"GigabitEthernet\", \"Gi\");\n  let short = replace(short, \"FastEthernet\", \"Fa\");\n  let short = replace(short, \"TenGigabitEthernet\", \"Te\");\n  let short = replace(short, \"Loopback\", \"Lo\");\n  // NX-OS — already short (Ethernet1/1) — just lowercase E\n  let short = replace(short, \"Ethernet\", \"Eth\");\n  // PAN-OS — already lowercase, no change needed\n  set_field(\"interface_name_short\", short);\nend"
}
EOF

curl -s -X POST \
  -u "admin:${GRAYLOG_ADMIN_PASSWORD}" \
  -H "X-Requested-By: ansible" \
  -H "Content-Type: application/json" \
  "http://172.16.0.60:9000/api/system/pipelines/rule" \
  -d @/tmp/normalise_interface_rule.json \
  | python3 -m json.tool | grep -E '"id"|"title"'

# Add this rule to Stage 1 of the Network Device Processing pipeline
# (update the pipeline source to include this rule in stage 1)
```

### Grafana interface regex for cross-platform panels

```bash
# In Grafana panels that filter by interface name, use these regex patterns:
# Instead of exact interface name matching, use regex that covers all platforms

# Panel variable for interface name:
# Query: label_values(ifOperStatus{device=~"$device"}, ifName)
# Regex filter to exclude loopbacks and management interfaces:
#   /^(?!.*[Ll]o|.*[Mm]gmt|.*[Mm]anagement).*/

# For critical fabric interfaces only:
#   /(Gi[123]|Eth1\/[123]|ethernet1\/[123])/

# Example Grafana panel PromQL using regex:
# rate(ifHCInOctets{device=~"$device", ifName=~"(Gi[123]|Eth1\\/[123]|ethernet1\\/[123])"}[5m]) * 8
```

---

## 41.6 — Gap 4: Fabric-Specific Grafana Panels

Add a "Fabric Topology" row to the Lab Operations Centre dashboard that shows the specific interface pairs that matter for this topology — not all interfaces on all devices.

```bash
cat > ~/projects/ansible-network/playbooks/integration/fabric_grafana_panels.yml << 'EOF'
---
# Add fabric-topology-aware panels to the Lab Operations Centre dashboard
# Run after Part 40's grafana_ops_dashboard.yml has been run

- name: "Grafana | Add fabric topology panels"
  hosts: localhost
  gather_facts: false
  tags: [grafana, fabric, integration]

  vars:
    grafana_url: "http://172.16.0.50:3000"
    grafana_user: admin
    grafana_password: "{{ vault_grafana_admin_password }}"

  tasks:

    - name: "Grafana | Get existing dashboard"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/dashboards/uid/lab-ops-centre"
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
      register: existing_dashboard

    - name: "Grafana | Append fabric topology row to dashboard"
      ansible.builtin.uri:
        url: "{{ grafana_url }}/api/dashboards/db"
        method: POST
        user: "{{ grafana_user }}"
        password: "{{ grafana_password }}"
        force_basic_auth: true
        body_format: json
        body:
          overwrite: true
          dashboard: >-
            {{ existing_dashboard.json.dashboard | combine({
               'panels': existing_dashboard.json.dashboard.panels + [

                 {
                   'type': 'row',
                   'title': '🏗️ Fabric Topology',
                   'gridPos': {'x': 0, 'y': 39, 'w': 24, 'h': 1},
                   'collapsed': false
                 },

                 {
                   'type': 'timeseries',
                   'title': 'Spine Uplinks — dist-01 ↔ spine-01',
                   'description': 'Critical iBGP path — distribution to spine',
                   'gridPos': {'x': 0, 'y': 40, 'w': 12, 'h': 7},
                   'targets': [
                     {
                       'expr': 'rate(ifHCInOctets{device="dist-01", ifName=~"Gi2|GigabitEthernet2"}[5m]) * 8',
                       'legendFormat': 'dist-01 Gi2 ▼'
                     },
                     {
                       'expr': 'rate(ifHCInOctets{device="spine-01", ifName=~"Eth1/1|Ethernet1/1"}[5m]) * 8',
                       'legendFormat': 'spine-01 Eth1/1 ▼'
                     }
                   ],
                   'fieldConfig': {'defaults': {'unit': 'bps'}}
                 },

                 {
                   'type': 'timeseries',
                   'title': 'Leaf Uplinks — spine-01 ↔ leaf-01/leaf-02',
                   'description': 'East-west traffic paths from spine to leaves',
                   'gridPos': {'x': 12, 'y': 40, 'w': 12, 'h': 7},
                   'targets': [
                     {
                       'expr': 'rate(ifHCInOctets{device="spine-01", ifName=~"Eth1/2|Ethernet1/2"}[5m]) * 8',
                       'legendFormat': 'spine-01 → leaf-01 ▼'
                     },
                     {
                       'expr': 'rate(ifHCInOctets{device="spine-01", ifName=~"Eth1/3|Ethernet1/3"}[5m]) * 8',
                       'legendFormat': 'spine-01 → leaf-02 ▼'
                     },
                     {
                       'expr': 'rate(ifHCOutOctets{device="spine-01", ifName=~"Eth1/2|Ethernet1/2"}[5m]) * 8',
                       'legendFormat': 'spine-01 → leaf-01 ▲'
                     },
                     {
                       'expr': 'rate(ifHCOutOctets{device="spine-01", ifName=~"Eth1/3|Ethernet1/3"}[5m]) * 8',
                       'legendFormat': 'spine-01 → leaf-02 ▲'
                     }
                   ],
                   'fieldConfig': {'defaults': {'unit': 'bps'}}
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: wan-r1 ↔ panos-fw01',
                   'gridPos': {'x': 0, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="wan-r1", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]},
                     'mappings': [
                       {'type': 'value', 'options': {
                         '1': {'text': 'Idle'},
                         '2': {'text': 'Connect'},
                         '3': {'text': 'Active'},
                         '6': {'text': 'Established'}
                       }}
                     ]
                   }
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: wan-r2 ↔ panos-fw01',
                   'gridPos': {'x': 4, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="wan-r2", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]}
                   }
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: panos-fw01 ↔ dist-01',
                   'gridPos': {'x': 8, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="panos-fw01", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]}
                   }
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: dist-01 ↔ spine-01',
                   'gridPos': {'x': 12, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="dist-01", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]}
                   }
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: spine-01 ↔ leaf-01',
                   'gridPos': {'x': 16, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="spine-01", bgpPeerRemoteAddr="10.0.3.2", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]}
                   }
                 },

                 {
                   'type': 'stat',
                   'title': 'BGP: spine-01 ↔ leaf-02',
                   'gridPos': {'x': 20, 'y': 47, 'w': 4, 'h': 4},
                   'targets': [{
                     'expr': 'bgpPeerState{device="spine-01", bgpPeerRemoteAddr="10.0.4.2", category="network"}',
                     'legendFormat': ''
                   }],
                   'options': {
                     'colorMode': 'background',
                     'thresholds': {'steps': [
                       {'color': 'red', 'value': 0},
                       {'color': 'green', 'value': 6}
                     ]}
                   }
                 }
               ]
            }, recursive=True) }}
        status_code: 200

    - name: "Grafana | Dashboard updated"
      ansible.builtin.debug:
        msg:
          - "Fabric topology panels added to Lab Operations Centre"
          - "URL: http://172.16.0.50:3000/d/lab-ops-centre"
          - "New row: Fabric Topology (scroll to bottom)"
          - "  - Spine uplink traffic (dist-01 ↔ spine-01)"
          - "  - Leaf uplink traffic (spine-01 ↔ leaf-01/02)"
          - "  - Per-session BGP state stat panels for all 6 sessions"
EOF
```

---

## 41.7 — Gap 5: Syslog Noise Suppression at Startup

```bash
# Containerlab topologies generate significant syslog noise during startup:
# - Interface flap messages as virtual interfaces initialise
# - BGP reconvergence messages (ADJCHANGE) as sessions establish
# - OSPF/BGP SPF calculations
# - Line protocol messages

# Add a Graylog pipeline rule that suppresses startup noise
# by tagging messages as 'startup_noise' for the first 5 minutes
# after the first message from a device is seen

cat > /tmp/suppress_startup_noise.json << 'EOF'
{
  "title": "Tag Containerlab Startup Noise",
  "description": "Tags interface/BGP flap messages within 5 minutes of first message from device as startup_noise to suppress alerting",
  "source": "rule \"Tag Containerlab Startup Noise\"\nwhen\n  has_field(\"event_type\") AND\n  (to_string($message.event_type) == \"interface_down\" OR\n   to_string($message.event_type) == \"bgp_neighbor_down\") AND\n  has_field(\"device_name\")\nthen\n  // Events in first 5 minutes after device boots are startup noise\n  // Check if this device has been seen for < 5 minutes\n  // (Approximated by checking if uptime OID is < 300 seconds)\n  // Mark as potential noise — alert evaluation uses 'for: 3m' which\n  // naturally handles most startup noise without this rule\n  // This rule adds an explicit label for Graylog stream filtering\n  let uptime_seconds = to_long($message.snmp_uptime) / 100;\n  if (!is_null(uptime_seconds) AND uptime_seconds < 300) {\n    set_field(\"startup_noise\", true);\n    set_field(\"alert_suppressed\", \"Device uptime < 5 minutes — startup noise\");\n  }\nend"
}
EOF

curl -s -X POST \
  -u "admin:${GRAYLOG_ADMIN_PASSWORD}" \
  -H "X-Requested-By: ansible" \
  -H "Content-Type: application/json" \
  "http://172.16.0.60:9000/api/system/pipelines/rule" \
  -d @/tmp/suppress_startup_noise.json \
  | python3 -m json.tool | grep -E '"id"|"title"'

# Update Graylog alert conditions from Part 39 to exclude startup_noise
# The BGP and interface alert queries should add: AND NOT startup_noise:true
# Update via UI: Alerts → Event Definitions → edit each alert
# Change query from: event_type:interface_down
# To:               event_type:interface_down AND NOT startup_noise:true
```

---

## 41.8 — Master Onboarding Playbook

This is the single playbook that brings a freshly started Containerlab topology to fully monitored state, running the Part 37–40 plays in the correct order with the Containerlab-specific fixes from this part.

```bash
cat > ~/projects/ansible-network/playbooks/fabric_onboard.yml << 'EOF'
---
# Master playbook: Onboard Containerlab fabric to full monitoring
# Runs Part 37-40 playbooks in correct order + Part 41 fabric-specific fixes
#
# Usage:
#   Full onboard (first time or after topology restart):
#   ansible-playbook fabric_onboard.yml -i inventory/hosts.yml \
#     --vault-id lab@.vault/lab.txt
#
#   SNMP + syslog only (quick re-run after config change):
#   ansible-playbook fabric_onboard.yml -i inventory/hosts.yml \
#     --vault-id lab@.vault/lab.txt --tags device_config
#
#   Registration only (Zabbix + Prometheus targets):
#   ansible-playbook fabric_onboard.yml -i inventory/hosts.yml \
#     --vault-id lab@.vault/lab.txt --tags registration

- name: "Fabric Onboard | Wait for Containerlab topology to be ready"
  hosts: network_devices
  gather_facts: false
  tags: [always]

  tasks:

    - name: "Pre-check | Wait for SSH/management to be reachable"
      ansible.builtin.wait_for:
        host: "{{ ansible_host }}"
        port: 22
        timeout: 120
        sleep: 5
      delegate_to: localhost

    - name: "Pre-check | Test connectivity"
      cisco.ios.ios_command:
        commands: ["show version | head 3"]
      register: version_check
      failed_when: false
      when: ansible_network_os is defined and 'ios' in ansible_network_os

    - name: "Pre-check | Report device ready"
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} is reachable"

# ── Step 1: Configure SNMP on all devices ───────────────────────────
- name: "Step 1 | Configure SNMP v2c + v3"
  import_playbook: infrastructure/zabbix/configure_snmp.yml
  tags: [device_config, snmp]

# ── Step 2: Containerlab-specific SNMP fixes ────────────────────────
- name: "Step 2 | Apply Containerlab SNMP quirk fixes"
  import_playbook: infrastructure/zabbix/containerlab_snmp_fixes.yml
  tags: [device_config, snmp, fix]

# ── Step 3: Configure syslog on all devices ─────────────────────────
- name: "Step 3 | Configure syslog → Graylog"
  import_playbook: infrastructure/graylog/configure_syslog.yml
  tags: [device_config, syslog]

# ── Step 4: Register devices in Zabbix ──────────────────────────────
- name: "Step 4 | Register hosts in Zabbix"
  import_playbook: infrastructure/zabbix/register_hosts.yml
  tags: [registration, zabbix]

# ── Step 5: Sync to Prometheus file-based SD ────────────────────────
- name: "Step 5 | Sync Netbox → Prometheus scrape targets"
  import_playbook: integration/netbox_to_prometheus_sync.yml
  tags: [registration, prometheus]

# ── Step 6: Load fabric-specific Prometheus rules ───────────────────
- name: "Step 6 | Load fabric BGP alert rules"
  hosts: observability_hosts
  gather_facts: false
  become: true
  tags: [registration, prometheus, rules]

  tasks:
    - name: "Rules | Copy fabric BGP rules to Prometheus"
      ansible.builtin.copy:
        src: "{{ playbook_dir }}/../files/fabric_bgp.yml"
        dest: /opt/observability/prometheus/rules/fabric_bgp.yml
        mode: '0644'

    - name: "Rules | Reload Prometheus"
      ansible.builtin.uri:
        url: "http://localhost:9090/-/reload"
        method: POST
        status_code: 200
      delegate_to: localhost
      failed_when: false

# ── Step 7: Add Grafana fabric panels ────────────────────────────────
- name: "Step 7 | Add fabric topology panels to Grafana"
  import_playbook: integration/fabric_grafana_panels.yml
  tags: [grafana, fabric]

# ── Step 8: Verify full onboarding ───────────────────────────────────
- name: "Step 8 | Run end-state verification"
  import_playbook: fabric_verify.yml
  tags: [verify, always]
EOF
```

---

## 41.9 — Simulated Failure: Interface Shutdown on dist-01

This is the core exercise of Part 41. Shut `GigabitEthernet2` on `dist-01` — the uplink toward `spine-01` — and watch the full alert chain fire across all three tools simultaneously.

```
Why dist-01 Gi2?
  dist-01 Gi2 is the link between the distribution and spine layers.
  Shutting it causes:
    - Interface down event on dist-01 (ifOperStatus → 2)
    - BGP session loss: dist-01 ↔ spine-01 (iBGP drops)
    - BGP session cascades: spine-01 ↔ leaf-01, spine-01 ↔ leaf-02
      MAY be affected depending on next-hop reachability
    - Syslog messages from dist-01 AND spine-01 AND leaf switches
    - Multiple correlated events in Graylog
    - Multiple Prometheus alerts firing
  This gives the most interesting multi-tool observation with
  a single change.
```

### Execute the failure

```bash
# Step 1: Open three browser tabs before triggering the failure
# Tab 1: Grafana — Lab Operations Centre
#         http://172.16.0.50:3000/d/lab-ops-centre
#         Set refresh to 10s (top right)
#         Scroll to Fabric Topology row — watch BGP stat panels

# Tab 2: Zabbix — Monitoring → Problems
#         http://172.16.0.40/zabbix/problems.php
#         Set auto-refresh to 15s

# Tab 3: Graylog — Streams → Interface Events
#         http://172.16.0.60:9000/streams
#         Click Interface Events → Search within this stream
#         Set time range to "Last 5 minutes", enable live update

# Step 2: Trigger the failure from the control node
ssh ansible@172.16.0.21  # dist-01

conf t
interface GigabitEthernet2
  shutdown
end

# You should see immediately on the terminal:
# %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet2, changed state to down
# %BGP-5-ADJCHANGE: neighbor 10.0.2.2 Down Interface flap
# (spine-01 sees the same BGP ADJCHANGE from its side)
```

### What each tool shows — timeline

```
t+0s  — dist-01: interface GigabitEthernet2 shut
t+2s  — dist-01: IOS generates syslog messages:
          %LINEPROTO-5-UPDOWN: ... GigabitEthernet2 ... down
          %BGP-5-ADJCHANGE: neighbor 10.0.2.2 Down Interface flap

t+5s  — spine-01: IOS generates syslog messages:
          %BGP-5-ADJCHANGE: neighbor 10.0.2.1 Down Holdtime expired

── GRAYLOG (t+10–20s) ────────────────────────────────────────────────
  Streams → Interface Events:
    Source: 172.16.0.21 (dist-01)
    Message: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet2, changed state to down
    Fields extracted by pipeline:
      cisco_facility: LINEPROTO
      cisco_mnemonic: UPDOWN
      cisco_severity_code: 5
      event_type: interface_down
      interface_name: GigabitEthernet2
      interface_name_short: Gi2
      device_name: dist-01
      device_role: distribution
      device_platform: ios-xe

  Streams → BGP Events:
    Source: 172.16.0.21 (dist-01)
    Message: %BGP-5-ADJCHANGE: neighbor 10.0.2.2 Down Interface flap
    Fields extracted:
      event_type: bgp_neighbor_down
      bgp_peer_ip: 10.0.2.2
      device_name: dist-01

  Same BGP event appears from 172.16.0.31 (spine-01) seconds later

  Graylog alerts evaluate at t+60s:
    → "Network Interface Down" event definition fires
    → "BGP Neighbor Down" event definition fires for dist-01
    → Slack notifications sent

── ZABBIX (t+60–120s) ────────────────────────────────────────────────
  Monitoring → Problems:
    [HIGH]    Interface GigabitEthernet2 is down on dist-01
    [DISASTER] BGP neighbor 10.0.2.2 Down on dist-01
    [DISASTER] BGP neighbor 10.0.2.1 Down on spine-01

  The trigger fires after the second consecutive poll shows
  the interface down — typically 1-2 poll cycles (60-120 seconds).

  Zabbix shows:
    - Duration: actively counting since trigger fired
    - Acknowledged: No (until engineer acknowledges)
    - Actions: alert sent (if email/Slack configured)

── PROMETHEUS/GRAFANA (t+60–120s) ────────────────────────────────────
  Prometheus → Alerts page (http://172.16.0.50:9090/alerts):
    InterfaceDown{device="dist-01", ifName="GigabitEthernet2"}
      State: PENDING → FIRING (after 'for: 1m' elapses)
    BGPPeerDown{device="dist-01"}
      State: PENDING → FIRING
    FabricBGPSessionCountLow
      State: PENDING → FIRING (count drops below 6)

  Grafana — Lab Operations Centre:
    Row 1: Active Alerts counter increments (1 → 3 → more)
    Row 3: BGP peer state table shows dist-01 peer as non-6
    Row 3: dist-01 ↔ spine-01 stat panel turns RED
    Row 6: Loki logs panel shows the syslog messages real-time
    Fabric row: dist-01↔spine-01 traffic drops to zero
               spine-01↔leaf BGP stat panels may turn RED

── AWX (t+60–90s) ────────────────────────────────────────────────────
  Graylog BGP alert fires webhook → AWX
  Jobs → Workflow Jobs:
    "BGP Failure — Automated Remediation" — RUNNING
    Stage 1 (Backup): capturing dist-01 running config
    Stage 2 (Diagnose): collecting BGP summary, interface states
    Stage 3 (Notify): Slack message with full diagnosis

  Slack #network-alerts receives:
    "BGP Neighbor Down — Automated Diagnosis Report"
    Device: dist-01
    Root cause: BGP session idle — peer unreachable or TCP not established
    BGP Summary: (excerpt of show bgp summary)
    Recent logs: (last 10 syslog lines from device buffer)
    [View in Graylog] [View in Grafana]
```

### Restore and observe recovery

```bash
# Restore the interface on dist-01
conf t
interface GigabitEthernet2
  no shutdown
end

# Watch recovery across all three tools:
# Graylog: new messages appear — "changed state to up", "ADJCHANGE ... Up"
# Zabbix:  problem duration stops, status shows "RESOLVED"
# Grafana: BGP stat panels return to green, alert count drops
# Prometheus: alerts clear (state → RESOLVED in AlertManager)
# AWX:     no new workflow fires (grace period prevents re-trigger)

# Recovery timeline:
# t+0s:  no shutdown entered
# t+5s:  IOS logs "GigabitEthernet2 changed state to up"
# t+15s: BGP session begins reconnecting (TCP SYN)
# t+30s: BGP OPEN messages exchanged
# t+45s: BGP ESTABLISHED — IOS logs "ADJCHANGE: neighbor Up"
# t+60s: Zabbix poll sees ifOperStatus=1 — problem resolves
# t+60s: Prometheus scrape sees bgpPeerState=6 — alert clears
# t+90s: AlertManager sends resolved notification to Slack
```

---

## 41.10 — End-State Verification Playbook

```bash
cat > ~/projects/ansible-network/playbooks/fabric_verify.yml << 'EOF'
---
# Single verification playbook — checks all devices are registered
# and actively sending data to all monitoring tools
# Usage: ansible-playbook fabric_verify.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Fabric Verify | End-state verification"
  hosts: localhost
  gather_facts: false
  tags: [verify]

  vars:
    zabbix_url:    "http://172.16.0.40"
    prometheus_url: "http://172.16.0.50:9090"
    graylog_url:   "http://172.16.0.60:9000"
    netbox_url:    "http://172.16.0.20"
    zabbix_user:   Admin
    zabbix_password: "{{ vault_zabbix_web_password }}"
    graylog_user:  admin
    graylog_password: "{{ vault_graylog_admin_password }}"
    netbox_token:  "{{ vault_netbox_api_token }}"

    # Expected devices — adjust if topology differs
    expected_devices:
      - { name: wan-r1,     ip: "172.16.0.11", platform: ios-xe }
      - { name: wan-r2,     ip: "172.16.0.12", platform: ios-xe }
      - { name: dist-01,    ip: "172.16.0.21", platform: ios-xe }
      - { name: spine-01,   ip: "172.16.0.31", platform: nxos   }
      - { name: leaf-01,    ip: "172.16.0.33", platform: ios-xe }
      - { name: leaf-02,    ip: "172.16.0.34", platform: ios-xe }
      - { name: panos-fw01, ip: "172.16.0.51", platform: panos  }

  tasks:

    # ── Zabbix: all devices registered and available ──────────────────
    - name: "Verify | Zabbix — get all registered hosts"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: host.get
          params:
            output: [hostid, host, status, available]
            selectInterfaces: [ip, available]
          auth: "{{ zbx_auth_token | default('') }}"
          id: 1
      register: zbx_hosts_raw

    - name: "Verify | Zabbix auth (needed for host.get)"
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
          id: 0
      register: zbx_auth
      no_log: true

    - name: "Verify | Zabbix — get all registered hosts (authenticated)"
      ansible.builtin.uri:
        url: "{{ zabbix_url }}/api_jsonrpc.php"
        method: POST
        body_format: json
        body:
          jsonrpc: "2.0"
          method: host.get
          params:
            output: [hostid, host, status]
            selectInterfaces: [ip, available]
          auth: "{{ zbx_auth.json.result }}"
          id: 1
      register: zbx_hosts
      no_log: true

    - name: "Verify | Zabbix — check each expected device"
      ansible.builtin.assert:
        that:
          - zbx_hosts.json.result | selectattr('host', 'eq', item.name) | list | length > 0
        success_msg: "✅ Zabbix: {{ item.name }} is registered"
        fail_msg:    "❌ Zabbix: {{ item.name }} NOT FOUND"
      loop: "{{ expected_devices }}"
      loop_control:
        label: "{{ item.name }}"

    # ── Prometheus: all devices are active scrape targets ─────────────
    - name: "Verify | Prometheus — get active targets"
      ansible.builtin.uri:
        url: "{{ prometheus_url }}/api/v1/targets?state=active"
      register: prom_targets

    - name: "Verify | Prometheus — extract target IPs"
      ansible.builtin.set_fact:
        prom_target_ips: >-
          {{ prom_targets.json.data.activeTargets |
             map(attribute='labels') |
             map(attribute='instance') |
             map('regex_replace', ':.*', '') |
             list }}

    - name: "Verify | Prometheus — check each device has a target"
      ansible.builtin.assert:
        that:
          - item.ip in prom_target_ips
        success_msg: "✅ Prometheus: {{ item.name }} ({{ item.ip }}) has an active target"
        fail_msg:    "❌ Prometheus: {{ item.name }} ({{ item.ip }}) NOT in scrape targets"
      loop: "{{ expected_devices }}"
      loop_control:
        label: "{{ item.name }}"

    # ── Prometheus: devices are UP (SNMP responding) ──────────────────
    - name: "Verify | Prometheus — check SNMP scrapes are up"
      ansible.builtin.uri:
        url: "{{ prometheus_url }}/api/v1/query?query=up%7Bcategory%3D%22network%22%7D"
      register: prom_up

    - name: "Verify | Prometheus — report UP status per device"
      ansible.builtin.debug:
        msg: >
          {{ 'UP' if item.value[1] == '1' else 'DOWN' }} — {{ item.metric.device | default(item.metric.instance) }}
      loop: "{{ prom_up.json.data.result }}"
      loop_control:
        label: "{{ item.metric.device | default('unknown') }}"

    - name: "Verify | Prometheus — assert all devices UP"
      ansible.builtin.assert:
        that:
          - prom_up.json.data.result | selectattr('value', 'contains', '1') | list | length == expected_devices | length
        success_msg: "✅ Prometheus: all {{ expected_devices | length }} devices reporting UP"
        fail_msg: >
          ❌ Prometheus: only {{ prom_up.json.data.result | selectattr('value', 'contains', '1') | list | length }}
          of {{ expected_devices | length }} devices reporting UP

    # ── Graylog: all devices have sent messages recently ──────────────
    - name: "Verify | Graylog — check each device has sent messages"
      ansible.builtin.uri:
        url: >
          {{ graylog_url }}/api/search/universal/relative
          ?query=source:{{ item.ip }}&range=3600&limit=1&fields=source
        method: GET
        user: "{{ graylog_user }}"
        password: "{{ graylog_password }}"
        force_basic_auth: true
        headers:
          X-Requested-By: ansible
      register: graylog_check
      loop: "{{ expected_devices }}"
      loop_control:
        label: "{{ item.name }}"

    - name: "Verify | Graylog — report message status"
      ansible.builtin.assert:
        that:
          - item.json.total_results > 0
        success_msg: "✅ Graylog: {{ expected_devices[loop_index0].name }} — {{ item.json.total_results }} messages in last hour"
        fail_msg:    "❌ Graylog: {{ expected_devices[loop_index0].name }} — NO messages received in last hour (check syslog config)"
      loop: "{{ graylog_check.results }}"
      loop_control:
        label: "{{ expected_devices[loop_index0].name }}"

    # ── BGP fabric health ─────────────────────────────────────────────
    - name: "Verify | Prometheus — BGP session count"
      ansible.builtin.uri:
        url: "{{ prometheus_url }}/api/v1/query?query=count(bgpPeerState%7Bcategory%3D%22network%22%7D%20%3D%3D%206)"
      register: bgp_count

    - name: "Verify | BGP established session count"
      ansible.builtin.assert:
        that:
          - bgp_count.json.data.result | length > 0
          - (bgp_count.json.data.result[0].value[1] | int) >= 6
        success_msg: "✅ BGP: {{ bgp_count.json.data.result[0].value[1] }} sessions established (expected: 6)"
        fail_msg: >
          ❌ BGP: only {{ bgp_count.json.data.result[0].value[1] | default('0') }} sessions established
          (expected: 6). Check dist-01↔spine-01 and leaf uplinks.

    # ── Summary ───────────────────────────────────────────────────────
    - name: "Verify | Print summary"
      ansible.builtin.debug:
        msg:
          - "══════════════════════════════════════════════════════════════"
          - " Containerlab Fabric — Monitoring Verification Complete"
          - "══════════════════════════════════════════════════════════════"
          - " Devices in scope:    {{ expected_devices | length }}"
          - " Zabbix registered:   see results above"
          - " Prometheus scraping: see results above"
          - " Graylog receiving:   see results above"
          - " BGP sessions up:     {{ bgp_count.json.data.result[0].value[1] | default('unknown') }}/6"
          - ""
          - " Dashboards:"
          - "   Lab Ops Centre: http://172.16.0.50:3000/d/lab-ops-centre"
          - "   Zabbix Problems: http://172.16.0.40/zabbix/problems.php"
          - "   Graylog Search:  http://172.16.0.60:9000"
          - "══════════════════════════════════════════════════════════════"
EOF
```

---

## 41.11 — Checklist

```
[ ] Containerlab topology running and reachable:
    [ ] All 7 device management IPs ping from control node
    [ ] SSH accessible to all IOS-XE devices
    [ ] NX-OS spine-01 SSH accessible
    [ ] PAN-OS web UI accessible at 172.16.0.51

[ ] Gap 1: SNMP quirks resolved
    [ ] containerlab_snmp_fixes.yml ran successfully
    [ ] IOS-XE devices: no ACL suffix on snmp-server community
    [ ] N9Kv: SNMP host does NOT specify use-vrf management
    [ ] Manual snmpget from Zabbix container succeeds for all 7 devices:
        docker exec zabbix-server snmpget -v 2c -c lab-monitor-ro
          172.16.0.{11,12,21,31,33,34,51} 1.3.6.1.2.1.1.1.0

[ ] Gap 2: BGP topology-aware Prometheus rules
    [ ] /opt/observability/prometheus/rules/fabric_bgp.yml deployed
    [ ] Prometheus reload triggered
    [ ] Prometheus UI → Rules shows:
        [ ] FabricBGPSessionCountLow
        [ ] SpineTotalConnectivityLoss
        [ ] LeafIsolated
        [ ] fabric_bgp_convergence_seconds (recording rule)
    [ ] Query count(bgpPeerState{category="network"} == 6)
        returns 6 in Prometheus

[ ] Gap 3: Interface naming normalised
    [ ] "Normalise Interface Name" Graylog pipeline rule created
    [ ] Test: message from dist-01 with GigabitEthernet2 shows
        interface_name_short=Gi2 field in Graylog message detail

[ ] Gap 4: Fabric Grafana panels
    [ ] fabric_grafana_panels.yml ran successfully
    [ ] Lab Operations Centre dashboard has Fabric Topology row:
        [ ] Spine uplink traffic panel (dist-01 ↔ spine-01)
        [ ] Leaf uplink traffic panel (spine-01 ↔ leaf-01/02)
        [ ] 6 BGP session stat panels — all green when topology healthy

[ ] Gap 5: Startup noise suppression
    [ ] "Tag Containerlab Startup Noise" Graylog pipeline rule created
    [ ] BGP and interface alert Graylog queries updated to add:
        AND NOT startup_noise:true

[ ] Master onboarding playbook
    [ ] fabric_onboard.yml created
    [ ] Full run completes without errors on a fresh topology start:
        ansible-playbook fabric_onboard.yml \
          -i inventory/hosts.yml --vault-id lab@.vault/lab.txt
    [ ] fabric_verify.yml runs at end of onboard and shows all green

[ ] Simulated failure test
    [ ] Three browser tabs open before triggering:
        [ ] Grafana Lab Ops Centre (10s refresh)
        [ ] Zabbix Problems page (15s refresh)
        [ ] Graylog Interface Events stream (live update)
    [ ] dist-01 Gi2 shut — failure triggered
    [ ] Graylog (t+10-20s):
        [ ] Interface Events stream: LINEPROTO-5-UPDOWN message visible
        [ ] BGP Events stream: ADJCHANGE Down message visible
        [ ] Fields extracted: event_type, interface_name, device_name, bgp_peer_ip
        [ ] device_name=dist-01 enriched from Netbox lookup table
    [ ] Zabbix (t+60-120s):
        [ ] Interface GigabitEthernet2 down trigger fires
        [ ] BGP ADJCHANGE trigger fires for dist-01
        [ ] Optional: spine-01 BGP trigger fires if session seen from spine side
    [ ] Prometheus/Grafana (t+60-120s):
        [ ] InterfaceDown alert FIRING in Prometheus /alerts
        [ ] BGPPeerDown alert FIRING
        [ ] FabricBGPSessionCountLow alert FIRING
        [ ] Grafana: dist-01↔spine-01 BGP stat panel RED
        [ ] Grafana: Active Alerts count > 0
        [ ] Grafana: Loki panel shows syslog messages
    [ ] AWX (t+60-90s):
        [ ] BGP Failure workflow triggered
        [ ] Slack notification received with diagnosis content
    [ ] dist-01 Gi2 restored — no shutdown
    [ ] Recovery confirmed in all three tools within 3 minutes
    [ ] All Prometheus alerts clear
    [ ] Zabbix problems resolve
    [ ] Grafana BGP panels return to green

[ ] End-state verification
    [ ] fabric_verify.yml runs standalone and all asserts pass:
        [ ] All 7 devices registered in Zabbix
        [ ] All 7 devices have active Prometheus scrape targets
        [ ] All 7 devices UP (SNMP responding)
        [ ] All 7 devices have sent messages to Graylog in last hour
        [ ] BGP session count == 6

[ ] All new playbooks committed to Gitea:
    [ ] playbooks/infrastructure/zabbix/containerlab_snmp_fixes.yml
    [ ] playbooks/integration/fabric_grafana_panels.yml
    [ ] playbooks/fabric_onboard.yml
    [ ] playbooks/fabric_verify.yml
    [ ] files/fabric_bgp.yml (Prometheus rules)
```

---

*The two threads of this guide have fully merged. The Containerlab fabric from Parts 1–32 is now a fully observed network — every device monitored by Zabbix via SNMP, scraped by Prometheus via the SNMP Exporter, shipping syslog into Graylog where messages are parsed and enriched with Netbox metadata, with a single Grafana dashboard showing the health of every BGP session and every critical interface pair. A failure on dist-01 produces correlated, structured alerts in three tools simultaneously, triggers an automated diagnostic workflow, and delivers a diagnosis report to Slack before most engineers would have opened a terminal. Part 42 is the capstone — a complete project that starts with nothing and ends with a fully deployed, monitored, and documented network automation platform.*

*Next up: **Part 42 — Capstone Project: Full Stack from Scratch***
