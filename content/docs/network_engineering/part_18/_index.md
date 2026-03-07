---
draft: false
title: '18 - IOS for CCNP'
description: "Part 18 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 18
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 18: Cisco IOS/IOS-XE Network Automation (CCNP Level)

> *Part 17 covered every CCNA-level configuration domain. Part 18 goes deeper — BGP policy, multi-area OSPF with authentication, QoS, VRFs, HSRP, EtherChannel, GRE tunnels, and more. The same data-model-first approach applies: one comprehensive `host_vars` data model drives a master playbook and a Jinja2 template. The `ios_config` module's advanced parameters (`parents`, `before`, `after`, `match`) appear where they naturally fit — in the configurations that require them.*

---

## 18.1 — The CCNP Data Model Extension

The CCNP data model extends `host_vars/wan-r1.yml` with new configuration domains. I add these keys to the existing file rather than replacing it — the CCNA domains from Part 17 remain intact.

```bash
cat >> ~/projects/ansible-network/inventory/host_vars/wan-r1.yml << 'EOF'

# =============================================================
# CCNP Extensions — added in Part 18
# =============================================================

# ── VRFs ──────────────────────────────────────────────────────────
vrfs:
  - name: MGMT
    rd: "65100:1"
    description: "Out-of-band management VRF"
    address_families:
      - afi: ipv4
        import_targets:
          - "65100:1"
        export_targets:
          - "65100:1"
  - name: CUSTOMER_A
    rd: "65100:100"
    description: "Customer A tenant VRF"
    address_families:
      - afi: ipv4
        import_targets:
          - "65100:100"
        export_targets:
          - "65100:100"

# ── VRF-Lite Interfaces ───────────────────────────────────────────
vrf_interfaces:
  GigabitEthernet4:
    vrf: CUSTOMER_A
    description: "Customer A CE-facing interface"
    ip: 10.100.0.1
    prefix: 30
  GigabitEthernet5:
    vrf: MGMT
    description: "OOB management interface"
    ip: 172.16.99.1
    prefix: 24

# ── OSPF Multi-Area with Authentication ──────────────────────────
ospf_advanced:
  process_id: 1
  router_id: 10.255.0.1
  areas:
    - area_id: 0
      type: backbone
      authentication: message-digest
      interfaces:
        - name: GigabitEthernet2
          type: broadcast
          cost: 10
          md5_key_id: 1
          md5_key: "{{ vault_ospf_md5_key }}"
        - name: Loopback0
          passive: true
    - area_id: 1
      type: stub
      interfaces:
        - name: GigabitEthernet3
          type: point-to-point
          cost: 20

# ── EIGRP Named Mode ──────────────────────────────────────────────
eigrp:
  process_name: ENTERPRISE
  as_number: 100
  address_families:
    - afi: ipv4
      vrf: default
      networks:
        - prefix: 10.255.0.1
          wildcard: 0.0.0.0
        - prefix: 192.168.1.0
          wildcard: 0.0.0.255
      passive_interfaces:
        - Loopback0
      variance: 2
      maximum_paths: 4

# ── BGP Advanced (extends base bgp: key from Part 17) ────────────
bgp_advanced:
  as_number: 65100
  router_id: 10.255.0.1

  # eBGP peers
  neighbors:
    - ip: 10.10.10.2
      remote_as: 65200
      description: "eBGP to FW-01 (upstream)"
      update_source: ~
      route_map_in: FROM_UPSTREAM
      route_map_out: TO_UPSTREAM
      soft_reconfiguration_inbound: true
      send_community: both

    - ip: 10.10.20.2
      remote_as: 65100
      description: "iBGP to WAN-R2"
      update_source: Loopback0
      next_hop_self: true

  # Prefix lists
  prefix_lists:
    - name: ALLOW_DEFAULT
      entries:
        - seq: 5
          action: permit
          prefix: 0.0.0.0/0
    - name: BLOCK_RFC1918
      entries:
        - seq: 5
          action: deny
          prefix: 10.0.0.0/8
          le: 32
        - seq: 10
          action: deny
          prefix: 172.16.0.0/12
          le: 32
        - seq: 15
          action: deny
          prefix: 192.168.0.0/16
          le: 32
        - seq: 20
          action: permit
          prefix: 0.0.0.0/0
          le: 32

  # Route maps
  route_maps:
    - name: FROM_UPSTREAM
      entries:
        - seq: 10
          action: permit
          match:
            ip_prefix_list: ALLOW_DEFAULT
          set:
            local_preference: 200
        - seq: 20
          action: deny
          match:
            ip_prefix_list: BLOCK_RFC1918

    - name: TO_UPSTREAM
      entries:
        - seq: 10
          action: permit
          match: {}
          set:
            as_path_prepend: "65100 65100"

  address_families:
    - afi: ipv4
      networks:
        - prefix: 10.255.0.1
          mask: 255.255.255.255
        - prefix: 192.168.1.0
          mask: 255.255.255.0

# ── HSRP ──────────────────────────────────────────────────────────
hsrp:
  - interface: GigabitEthernet2
    group: 10
    version: 2
    virtual_ip: 192.168.1.254
    priority: 110          # Higher = active (wan-r1 is primary)
    preempt: true
    preempt_delay: 60
    timers:
      hello: 1
      hold: 3
    authentication:
      type: md5
      key: "{{ vault_hsrp_md5_key }}"
    track:
      - object: 1
        decrement: 20      # Lose priority if tracked object fails

# ── Object Tracking (for HSRP) ────────────────────────────────────
tracking:
  - id: 1
    type: interface
    interface: GigabitEthernet1
    track: line-protocol
    # If GigabitEthernet1 (WAN) goes down, HSRP priority drops by 20

# ── EtherChannel / Port-Channel ───────────────────────────────────
port_channels:
  - id: 1
    mode: active             # LACP active
    description: "LAG to Distribution Switch"
    min_links: 1
    members:
      - GigabitEthernet6
      - GigabitEthernet7
    layer: 3                 # Routed port-channel
    ip: 10.30.0.1
    prefix: 30

# ── GRE Tunnels ───────────────────────────────────────────────────
tunnels:
  - id: 0
    description: "GRE to Branch-01"
    source: GigabitEthernet1
    destination: 203.0.113.1
    tunnel_ip: 172.16.10.1
    prefix: 30
    keepalive:
      period: 10
      retries: 3

# ── QoS Policy ────────────────────────────────────────────────────
qos:
  class_maps:
    - name: VOICE
      match:
        dscp: [ef]
    - name: VIDEO
      match:
        dscp: [af41, af42, af43]
    - name: CRITICAL_DATA
      match:
        dscp: [af31, af32, af33]
    - name: BULK
      match:
        dscp: [af11, af12, af13]

  policy_maps:
    - name: WAN_EGRESS
      classes:
        - name: VOICE
          priority_percent: 10
        - name: VIDEO
          bandwidth_percent: 20
          queue_limit: 64
        - name: CRITICAL_DATA
          bandwidth_percent: 30
        - name: BULK
          bandwidth_percent: 10
          fair_queue: true
        - name: class-default
          fair_queue: true

  service_policy:
    - interface: GigabitEthernet1
      direction: output
      policy: WAN_EGRESS

# ── MPLS LDP (brief — niche topic) ────────────────────────────────
mpls:
  enabled: false             # Set true to enable MPLS LDP
  ldp_router_id: Loopback0
  interfaces:
    - GigabitEthernet2
    - GigabitEthernet3

# ── IPsec VPN (brief — niche topic) ──────────────────────────────
ipsec:
  enabled: false             # Set true to enable IPsec
  proposals:
    - name: AES256-SHA256
      encryption: aes 256
      hash: sha256
      dh_group: 14
  policies:
    - peer: 203.0.113.1
      proposal: AES256-SHA256
      psk: "{{ vault_ipsec_psk }}"
      tunnel_interface: Tunnel0
EOF
```

---

## 18.2 — `ios_config` Advanced Parameters

Before building the playbook, I cover the `ios_config` parameters that appear throughout this part. At CCNP level, most configuration doesn't have resource modules — `ios_config` is the primary tool, and its advanced parameters matter.

### `parents` — Context Hierarchy

`parents` establishes the configuration context before sending `lines`. This is the most important `ios_config` parameter:

```yaml
# Single-level parent (interface config)
cisco.ios.ios_config:
  parents: "interface GigabitEthernet1"
  lines:
    - "ip ospf 1 area 0"
    - "ip ospf network point-to-point"

# Multi-level parents (nested contexts)
cisco.ios.ios_config:
  parents:
    - "router bgp 65100"
    - "address-family ipv4"
  lines:
    - "neighbor 10.10.10.2 activate"
    - "network 10.255.0.1 mask 255.255.255.255"

# Policy-map with class context
cisco.ios.ios_config:
  parents:
    - "policy-map WAN_EGRESS"
    - "class VOICE"
  lines:
    - "priority percent 10"
```

### `match` — How Lines Are Compared to Existing Config

`match` controls how Ansible determines whether a line already exists and needs to be pushed:

```yaml
# match: line (default) — checks if each line exists anywhere under the parent
# Efficient but can false-positive if the same text appears in different contexts
cisco.ios.ios_config:
  parents: "router bgp 65100"
  lines:
    - "neighbor 10.10.10.2 remote-as 65200"
  match: line      # ← Checks: does "neighbor 10.10.10.2 remote-as 65200" exist?

# match: strict — checks lines exist in EXACT ORDER under the parent
# More precise but fails if order differs even when config is correct
cisco.ios.ios_config:
  parents: "ip access-list extended INTERNET_FILTER"
  lines:
    - "10 deny ip 10.0.0.0 0.255.255.255 any"
    - "20 deny ip 172.16.0.0 0.15.255.255 any"
    - "30 permit ip any any"
  match: strict

# match: exact — ALL lines under parent must match — removes extras
# Use for complete replacement of a config block
cisco.ios.ios_config:
  parents: "policy-map WAN_EGRESS"
  lines:
    - "class VOICE"
    - " priority percent 10"
  match: exact     # ← Any other class entries under this policy-map will be removed

# match: none — always push the lines, no comparison
# Use when idempotency check is too expensive or unreliable
cisco.ios.ios_config:
  lines:
    - "crypto key generate rsa modulus 4096"
  match: none      # ← Always sends this command
```

### `before` and `after` — Bracketing Commands

`before` sends commands before the `lines` block. `after` sends commands after. Both run only when the task reports `changed` (when at least one line was actually pushed):

```yaml
# before: useful for entering a mode that requires a prerequisite command
cisco.ios.ios_config:
  before:
    - "no router eigrp 100"    # ← Remove old numbered EIGRP before named mode
  parents: "router eigrp ENTERPRISE"
  lines:
    - "no shutdown"

# after: useful for committing or activating configuration
cisco.ios.ios_config:
  parents:
    - "router bgp 65100"
    - "address-family ipv4"
  lines:
    - "neighbor 10.10.10.2 activate"
  after:
    - "end"       # ← Exit config mode explicitly (usually not needed but explicit)

# Real use case: clear BGP session after changing policy
cisco.ios.ios_config:
  parents: "router bgp 65100"
  lines:
    - "neighbor 10.10.10.2 route-map FROM_UPSTREAM in"
  after:
    - "clear ip bgp 10.10.10.2 soft in"   # ← Reset BGP session to apply new policy
  # after: only runs when the route-map line was actually changed
```

### `replace` — Block vs Line Replacement

```yaml
# replace: line (default) — only push missing lines (additive)
cisco.ios.ios_config:
  parents: "router bgp 65100"
  lines:
    - "neighbor 10.10.10.2 remote-as 65200"
  replace: line    # ← Adds the line if missing, ignores if present

# replace: block — replace the entire parent block with exactly these lines
# Removes lines not in the list — use with caution
cisco.ios.ios_config:
  parents: "ip prefix-list ALLOW_DEFAULT"
  lines:
    - "seq 5 permit 0.0.0.0/0"
  replace: block   # ← Removes ALL other entries from this prefix-list
```

---

## 18.3 — The CCNP Master Playbook

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_ios_ccnp.yml
```

```yaml
---
# =============================================================
# deploy_ios_ccnp.yml — CCNP-level IOS-XE deployment
# Extends deploy_ios_ccna.yml with advanced configuration domains
# All values sourced from host_vars and group_vars data model
#
# Common invocations:
#   Full run:         ansible-playbook deploy_ios_ccnp.yml
#   BGP policy:       ansible-playbook deploy_ios_ccnp.yml --tags bgp
#   QoS only:         ansible-playbook deploy_ios_ccnp.yml --tags qos
#   VRF + routing:    ansible-playbook deploy_ios_ccnp.yml --tags vrf,routing
#   Tunnels:          ansible-playbook deploy_ios_ccnp.yml --tags tunnels
#   Single device:    ansible-playbook deploy_ios_ccnp.yml -l wan-r1
# =============================================================

- name: "Deploy | CCNP-level IOS-XE advanced configuration"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:

    # ── Pre-flight ────────────────────────────────────────────────
    - name: "Pre-flight | Gather IOS facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - hardware
      tags: always

    - name: "Pre-flight | Backup before CCNP changes"
      block:
        - name: "Backup | Capture running config"
          cisco.ios.ios_command:
            commands: [show running-config]
          register: pre_change_config

        - name: "Backup | Save to control node"
          ansible.builtin.copy:
            content: "{{ pre_change_config.stdout[0] }}"
            dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_ccnp_{{ timestamp }}.cfg"
            mode: '0644'
          delegate_to: localhost
      tags: backup

    # ── Section 1: VRF Definition ─────────────────────────────────
    - block:
        - name: "VRF | Define VRF instances"
          cisco.ios.ios_vrf:
            vrfs:
              - name: "{{ item.name }}"
                rd: "{{ item.rd }}"
                description: "{{ item.description | default('') }}"
                route_import:
                  - "{{ item.address_families[0].import_targets[0] }}"
                route_export:
                  - "{{ item.address_families[0].export_targets[0] }}"
            state: present
          loop: "{{ vrfs | default([]) }}"
          loop_control:
            label: "VRF {{ item.name }} RD {{ item.rd }}"
          when: vrfs is defined
          notify: Save IOS configuration

        - name: "VRF | Assign interfaces to VRFs"
          cisco.ios.ios_config:
            parents: "interface {{ item.key }}"
            lines:
              - "vrf forwarding {{ item.value.vrf }}"
              - "description {{ item.value.description | default('') }}"
              - "ip address {{ item.value.ip }} {{ item.value.prefix | ansible.utils.ipaddr('netmask') }}"
              - "no shutdown"
          loop: "{{ vrf_interfaces | default({}) | dict2items }}"
          loop_control:
            label: "{{ item.key }} → VRF {{ item.value.vrf }}"
          when: vrf_interfaces is defined
          notify: Save IOS configuration
          # Note: vrf forwarding clears the IP address — must re-apply IP in same task

      tags: [vrf, routing]

    # ── Section 2: OSPF Multi-Area with Authentication ────────────
    - block:
        - name: "OSPF | Configure OSPF process"
          cisco.ios.ios_ospfv2:
            config:
              processes:
                - process_id: "{{ ospf_advanced.process_id }}"
                  router_id: "{{ ospf_advanced.router_id }}"
                  auto_cost:
                    reference_bandwidth: 10000
            state: merged
          when: ospf_advanced is defined
          notify: Save IOS configuration

        - name: "OSPF | Configure area types (stub areas)"
          cisco.ios.ios_config:
            parents: "router ospf {{ ospf_advanced.process_id }}"
            lines:
              - "area {{ item.area_id }} stub"
          loop: "{{ ospf_advanced.areas | default([]) | selectattr('type', 'equalto', 'stub') | list }}"
          loop_control:
            label: "Area {{ item.area_id }} stub"
          when: ospf_advanced is defined
          notify: Save IOS configuration

        - name: "OSPF | Configure area authentication"
          cisco.ios.ios_config:
            parents: "router ospf {{ ospf_advanced.process_id }}"
            lines:
              - "area {{ item.area_id }} authentication message-digest"
          loop: "{{ ospf_advanced.areas | default([]) | selectattr('authentication', 'defined') | list }}"
          loop_control:
            label: "Area {{ item.area_id }} authentication"
          when: ospf_advanced is defined
          notify: Save IOS configuration

        - name: "OSPF | Assign interfaces to areas with cost and MD5"
          cisco.ios.ios_config:
            parents: "interface {{ item.0.name }}"
            lines:
              - "ip ospf {{ ospf_advanced.process_id }} area {{ item.1.area_id }}"
              - "{% if item.0.type is defined %}ip ospf network {{ item.0.type }}{% endif %}"
              - "{% if item.0.cost is defined %}ip ospf cost {{ item.0.cost }}{% endif %}"
              - "{% if item.0.md5_key_id is defined %}ip ospf message-digest-key {{ item.0.md5_key_id }} md5 {{ item.0.md5_key }}{% endif %}"
          loop: "{{ ospf_advanced.areas | subelements('interfaces') | map('reverse') | list }}"
          loop_control:
            label: "{{ item.1.name }} → OSPF area {{ item.0.area_id }}"
          when: ospf_advanced is defined
          no_log: true    # md5_key may be in loop — suppress output
          notify: Save IOS configuration

        - name: "OSPF | Verify adjacencies after authentication change"
          cisco.ios.ios_command:
            commands:
              - show ip ospf neighbor
            wait_for:
              - result[0] contains FULL
            retries: 12
            interval: 5
          register: ospf_verify
          changed_when: false
          when: ospf_advanced is defined

        - name: "OSPF | Display neighbor state"
          ansible.builtin.debug:
            msg: "{{ ospf_verify.stdout_lines[0] }}"
          when: ospf_advanced is defined and ospf_verify is not skipped

      tags: [ospf, routing]

    # ── Section 3: EIGRP Named Mode ───────────────────────────────
    - block:
        - name: "EIGRP | Configure EIGRP named mode process"
          cisco.ios.ios_config:
            parents: "router eigrp {{ eigrp.process_name }}"
            lines:
              - "no shutdown"
            # ios_config match: line — only sends if the line is missing
          when: eigrp is defined
          notify: Save IOS configuration

        - name: "EIGRP | Configure address family"
          cisco.ios.ios_config:
            parents:
              - "router eigrp {{ eigrp.process_name }}"
              - "address-family ipv4 unicast autonomous-system {{ eigrp.as_number }}"
            lines:
              - "variance {{ eigrp.variance | default(1) }}"
              - "maximum-paths {{ eigrp.maximum_paths | default(4) }}"
          when: eigrp is defined
          notify: Save IOS configuration

        - name: "EIGRP | Configure networks"
          cisco.ios.ios_config:
            parents:
              - "router eigrp {{ eigrp.process_name }}"
              - "address-family ipv4 unicast autonomous-system {{ eigrp.as_number }}"
              - "topology base"
            lines:
              - "network {{ item.prefix }} {{ item.wildcard }}"
          loop: "{{ eigrp.address_families[0].networks | default([]) }}"
          loop_control:
            label: "EIGRP network: {{ item.prefix }}"
          when: eigrp is defined
          notify: Save IOS configuration

        - name: "EIGRP | Configure passive interfaces"
          cisco.ios.ios_config:
            parents:
              - "router eigrp {{ eigrp.process_name }}"
              - "address-family ipv4 unicast autonomous-system {{ eigrp.as_number }}"
            lines:
              - "af-interface {{ item }}"
              - " passive-interface"
              - " exit-af-interface"
          loop: "{{ eigrp.address_families[0].passive_interfaces | default([]) }}"
          loop_control:
            label: "EIGRP passive: {{ item }}"
          when: eigrp is defined
          notify: Save IOS configuration

      tags: [eigrp, routing]

    # ── Section 4: BGP Advanced — Prefix Lists, Route Maps, Policy ─
    - block:
        # Prefix lists first — route maps reference them
        - name: "BGP | Configure prefix lists"
          cisco.ios.ios_prefix_lists:
            config:
              - name: "{{ item.name }}"
                address_family: ipv4
                prefixes:
                  - sequence: "{{ entry.seq }}"
                    action: "{{ entry.action }}"
                    prefix: "{{ entry.prefix }}"
                    le: "{{ entry.le | default(omit) }}"
                    ge: "{{ entry.ge | default(omit) }}"
              for entry in item.entries
            state: merged
          loop: "{{ bgp_advanced.prefix_lists | default([]) }}"
          loop_control:
            label: "Prefix list: {{ item.name }}"
          when: bgp_advanced is defined
          notify: Save IOS configuration

        # Route maps — reference prefix lists
        - name: "BGP | Configure route map entries"
          cisco.ios.ios_config:
            parents: "route-map {{ item.0.name }} {{ item.1.action }} {{ item.1.seq }}"
            lines: >-
              {{ (['match ip address prefix-list ' + item.1.match.ip_prefix_list]
                  if item.1.match.ip_prefix_list is defined else [])
               + (['set local-preference ' + item.1.set.local_preference | string]
                  if item.1.set is defined and item.1.set.local_preference is defined else [])
               + (['set as-path prepend ' + item.1.set.as_path_prepend]
                  if item.1.set is defined and item.1.set.as_path_prepend is defined else []) }}
          loop: "{{ bgp_advanced.route_maps | default([]) | subelements('entries') }}"
          loop_control:
            label: "route-map {{ item.0.name }} {{ item.1.action }} {{ item.1.seq }}"
          when: bgp_advanced is defined
          notify: Save IOS configuration

        # BGP global config — resource module
        - name: "BGP | Configure BGP process and router-id"
          cisco.ios.ios_bgp_global:
            config:
              as_number: "{{ bgp_advanced.as_number }}"
              bgp:
                router_id:
                  address: "{{ bgp_advanced.router_id }}"
                log_neighbor_changes: true
              neighbor:
                - neighbor_address: "{{ item.ip }}"
                  remote_as: "{{ item.remote_as }}"
                  description: "{{ item.description }}"
                  update_source: "{{ item.update_source | default(omit) }}"
              for item in bgp_advanced.neighbors
            state: merged
          when: bgp_advanced is defined
          notify: Save IOS configuration

        # BGP neighbor policy — ios_config for route-map and soft-reconfiguration
        - name: "BGP | Apply route maps to BGP neighbors"
          cisco.ios.ios_config:
            parents: "router bgp {{ bgp_advanced.as_number }}"
            lines: >-
              {{ (['neighbor ' + item.ip + ' route-map ' + item.route_map_in + ' in']
                  if item.route_map_in is defined else [])
               + (['neighbor ' + item.ip + ' route-map ' + item.route_map_out + ' out']
                  if item.route_map_out is defined else [])
               + (['neighbor ' + item.ip + ' soft-reconfiguration inbound']
                  if item.soft_reconfiguration_inbound | default(false) else [])
               + (['neighbor ' + item.ip + ' send-community ' + item.send_community]
                  if item.send_community is defined else [])
               + (['neighbor ' + item.ip + ' next-hop-self']
                  if item.next_hop_self | default(false) else []) }}
          loop: "{{ bgp_advanced.neighbors | default([]) }}"
          loop_control:
            label: "BGP policy: {{ item.ip }}"
          when: bgp_advanced is defined
          after:
            - "clear ip bgp * soft"
            # Uses ios_config after: — only resets BGP when a policy line actually changed
          notify: Save IOS configuration

        # BGP address family and networks
        - name: "BGP | Configure BGP address family networks"
          cisco.ios.ios_bgp_address_family:
            config:
              as_number: "{{ bgp_advanced.as_number }}"
              address_family:
                - afi: ipv4
                  safi: unicast
                  networks:
                    - address: "{{ item.prefix }}"
                      mask: "{{ item.mask }}"
                  for item in bgp_advanced.address_families[0].networks
            state: merged
          when: bgp_advanced is defined
          notify: Save IOS configuration

        # Verification
        - name: "BGP | Verify BGP summary"
          cisco.ios.ios_command:
            commands:
              - show ip bgp summary
          register: bgp_summary
          changed_when: false
          when: bgp_advanced is defined

        - name: "BGP | Display BGP summary"
          ansible.builtin.debug:
            msg: "{{ bgp_summary.stdout_lines[0] }}"
          when: bgp_advanced is defined and bgp_summary is not skipped

        - name: "BGP | Verify BGP prefix lists are installed"
          cisco.ios.ios_command:
            commands:
              - "show ip prefix-list"
          register: prefix_list_output
          changed_when: false
          when: bgp_advanced is defined

        - name: "BGP | Assert prefix lists exist"
          ansible.builtin.assert:
            that:
              - "item.name in prefix_list_output.stdout[0]"
            fail_msg: "Prefix list {{ item.name }} not found on {{ inventory_hostname }}"
            success_msg: "PASS: Prefix list {{ item.name }} present"
          loop: "{{ bgp_advanced.prefix_lists | default([]) }}"
          loop_control:
            label: "{{ item.name }}"
          when: bgp_advanced is defined and prefix_list_output is not skipped

      tags: [bgp, routing, policy]

    # ── Section 5: HSRP ───────────────────────────────────────────
    - block:
        - name: "HSRP | Configure object tracking"
          cisco.ios.ios_config:
            lines:
              - "track {{ item.id }} interface {{ item.interface }} {{ item.track }}"
          loop: "{{ tracking | default([]) }}"
          loop_control:
            label: "Track {{ item.id }}: {{ item.interface }} {{ item.track }}"
          when: tracking is defined
          notify: Save IOS configuration

        - name: "HSRP | Configure HSRP groups"
          cisco.ios.ios_config:
            parents: "interface {{ item.interface }}"
            lines:
              - "standby version {{ item.version | default(2) }}"
              - "standby {{ item.group }} ip {{ item.virtual_ip }}"
              - "standby {{ item.group }} priority {{ item.priority | default(100) }}"
              - "{% if item.preempt | default(false) %}standby {{ item.group }} preempt delay minimum {{ item.preempt_delay | default(0) }}{% endif %}"
              - "standby {{ item.group }} timers {{ item.timers.hello }} {{ item.timers.hold }}"
              - "{% if item.authentication is defined %}standby {{ item.group }} authentication md5 key-string {{ item.authentication.key }}{% endif %}"
              - "{% for track in item.track | default([]) %}standby {{ item.group }} track {{ track.object }} decrement {{ track.decrement }}{% endfor %}"
          loop: "{{ hsrp | default([]) }}"
          loop_control:
            label: "HSRP group {{ item.group }} on {{ item.interface }}"
          when: hsrp is defined
          no_log: true    # authentication.key is sensitive
          notify: Save IOS configuration

        - name: "HSRP | Verify HSRP state"
          cisco.ios.ios_command:
            commands:
              - show standby brief
          register: hsrp_state
          changed_when: false
          when: hsrp is defined

        - name: "HSRP | Display HSRP state"
          ansible.builtin.debug:
            msg: "{{ hsrp_state.stdout_lines[0] }}"
          when: hsrp is defined and hsrp_state is not skipped

      tags: [hsrp, redundancy]

    # ── Section 6: EtherChannel / Port-Channel ────────────────────
    - block:
        - name: "EtherChannel | Create port-channel interfaces"
          cisco.ios.ios_interfaces:
            config:
              - name: "Port-channel{{ item.id }}"
                description: "{{ item.description | default('') }}"
                enabled: true
            state: merged
          loop: "{{ port_channels | default([]) }}"
          loop_control:
            label: "Port-channel{{ item.id }}"
          when: port_channels is defined
          notify: Save IOS configuration

        - name: "EtherChannel | Configure port-channel as routed (L3)"
          cisco.ios.ios_l3_interfaces:
            config:
              - name: "Port-channel{{ item.id }}"
                ipv4:
                  - address: "{{ item.ip }}/{{ item.prefix }}"
            state: merged
          loop: "{{ port_channels | default([]) | selectattr('layer', 'equalto', 3) | list }}"
          loop_control:
            label: "Port-channel{{ item.id }} L3: {{ item.ip }}/{{ item.prefix }}"
          when: port_channels is defined
          notify: Save IOS configuration

        - name: "EtherChannel | Configure min-links on port-channel"
          cisco.ios.ios_config:
            parents: "interface Port-channel{{ item.id }}"
            lines:
              - "port-channel min-links {{ item.min_links | default(1) }}"
          loop: "{{ port_channels | default([]) }}"
          loop_control:
            label: "Port-channel{{ item.id }} min-links {{ item.min_links | default(1) }}"
          when: port_channels is defined
          notify: Save IOS configuration

        - name: "EtherChannel | Add member interfaces to port-channel (LACP)"
          cisco.ios.ios_config:
            parents: "interface {{ item.1 }}"
            lines:
              - "channel-group {{ item.0.id }} mode {{ item.0.mode | default('active') }}"
              - "no shutdown"
          loop: "{{ port_channels | default([]) | subelements('members') }}"
          loop_control:
            label: "{{ item.1 }} → Port-channel{{ item.0.id }} ({{ item.0.mode }})"
          when: port_channels is defined
          notify: Save IOS configuration

        - name: "EtherChannel | Verify LACP negotiation"
          cisco.ios.ios_command:
            commands:
              - show etherchannel summary
          register: etherchannel_state
          changed_when: false
          when: port_channels is defined

        - name: "EtherChannel | Display EtherChannel summary"
          ansible.builtin.debug:
            msg: "{{ etherchannel_state.stdout_lines[0] }}"
          when: port_channels is defined and etherchannel_state is not skipped

      tags: [etherchannel, lag, interfaces]

    # ── Section 7: GRE Tunnels ────────────────────────────────────
    - block:
        - name: "Tunnels | Configure GRE tunnel interfaces"
          cisco.ios.ios_config:
            parents: "interface Tunnel{{ item.id }}"
            lines:
              - "description {{ item.description | default('GRE Tunnel') }}"
              - "ip address {{ item.tunnel_ip }} {{ item.prefix | ansible.utils.ipaddr('netmask') }}"
              - "tunnel source {{ item.source }}"
              - "tunnel destination {{ item.destination }}"
              - "{% if item.keepalive is defined %}keepalive {{ item.keepalive.period }} {{ item.keepalive.retries }}{% endif %}"
              - "no shutdown"
            match: line
          loop: "{{ tunnels | default([]) }}"
          loop_control:
            label: "Tunnel{{ item.id }} → {{ item.destination }}"
          when: tunnels is defined
          notify: Save IOS configuration

      tags: [tunnels, vpn]

    # ── Section 8: QoS Policy ─────────────────────────────────────
    # No ios_qos resource module — ios_config with parents throughout
    - block:
        - name: "QoS | Configure class-maps"
          cisco.ios.ios_config:
            parents: "class-map match-any {{ item.name }}"
            lines: >-
              {{ item.match.dscp | default([])
                 | map('prepend', 'match ip dscp ')
                 | list }}
            # ios_config parents: + lines: with match: line
            # Only pushes missing match statements, idempotent
            match: line
          loop: "{{ qos.class_maps | default([]) }}"
          loop_control:
            label: "class-map {{ item.name }}"
          when: qos is defined
          notify: Save IOS configuration

        - name: "QoS | Configure policy-map class entries (priority)"
          cisco.ios.ios_config:
            parents:
              - "policy-map {{ item.0.name }}"
              - "class {{ item.1.name }}"
            lines:
              - "priority percent {{ item.1.priority_percent }}"
            match: line
          loop: >-
            {{ qos.policy_maps | default([])
               | subelements('classes')
               | selectattr('1.priority_percent', 'defined') | list }}
          loop_control:
            label: "policy-map {{ item.0.name }} class {{ item.1.name }}: priority {{ item.1.priority_percent }}%"
          when: qos is defined
          notify: Save IOS configuration

        - name: "QoS | Configure policy-map class entries (bandwidth)"
          cisco.ios.ios_config:
            parents:
              - "policy-map {{ item.0.name }}"
              - "class {{ item.1.name }}"
            lines:
              - "bandwidth percent {{ item.1.bandwidth_percent }}"
              - "{% if item.1.queue_limit is defined %}queue-limit {{ item.1.queue_limit }} packets{% endif %}"
              - "{% if item.1.fair_queue | default(false) %}fair-queue{% endif %}"
            match: line
          loop: >-
            {{ qos.policy_maps | default([])
               | subelements('classes')
               | selectattr('1.bandwidth_percent', 'defined') | list }}
          loop_control:
            label: "policy-map {{ item.0.name }} class {{ item.1.name }}: bandwidth {{ item.1.bandwidth_percent }}%"
          when: qos is defined
          notify: Save IOS configuration

        - name: "QoS | Apply service-policy to interfaces"
          cisco.ios.ios_config:
            parents: "interface {{ item.interface }}"
            lines:
              - "service-policy {{ item.direction }} {{ item.policy }}"
            match: line
          loop: "{{ qos.service_policy | default([]) }}"
          loop_control:
            label: "service-policy {{ item.direction }} {{ item.policy }} on {{ item.interface }}"
          when: qos is defined
          notify: Save IOS configuration

        - name: "QoS | Verify policy-map applied"
          cisco.ios.ios_command:
            commands:
              - show policy-map interface
          register: qos_verify
          changed_when: false
          when: qos is defined

        - name: "QoS | Display policy-map interface stats"
          ansible.builtin.debug:
            msg: "{{ qos_verify.stdout_lines[0][:30] }}"    # First 30 lines
          when: qos is defined and qos_verify is not skipped

      tags: [qos, services]

    # ── Section 9: MPLS LDP (brief — enable only when mpls.enabled) ─
    - block:
        - name: "MPLS | Enable MPLS IP globally"
          cisco.ios.ios_config:
            lines:
              - mpls ip
              - "mpls ldp router-id {{ mpls.ldp_router_id }} force"
          when: mpls is defined and mpls.enabled | bool
          notify: Save IOS configuration

        - name: "MPLS | Enable MPLS on interfaces"
          cisco.ios.ios_config:
            parents: "interface {{ item }}"
            lines:
              - mpls ip
          loop: "{{ mpls.interfaces | default([]) }}"
          loop_control:
            label: "MPLS on {{ item }}"
          when: mpls is defined and mpls.enabled | bool
          notify: Save IOS configuration

      tags: [mpls, routing]

    # ── Section 10: IPsec VPN (brief — enable only when ipsec.enabled) ─
    - block:
        - name: "IPsec | Configure IKEv2 proposal"
          cisco.ios.ios_config:
            parents: "crypto ikev2 proposal {{ item.name }}"
            lines:
              - "encryption {{ item.encryption }}"
              - "integrity {{ item.hash }}"
              - "group {{ item.dh_group }}"
          loop: "{{ ipsec.proposals | default([]) }}"
          loop_control:
            label: "IKEv2 proposal {{ item.name }}"
          when: ipsec is defined and ipsec.enabled | bool
          notify: Save IOS configuration

        - name: "IPsec | Configure IKEv2 keyring (PSK)"
          cisco.ios.ios_config:
            parents:
              - "crypto ikev2 keyring ANSIBLE_KEYRING"
              - "peer {{ item.peer }}"
            lines:
              - "address {{ item.peer }}"
              - "pre-shared-key {{ item.psk }}"
          loop: "{{ ipsec.policies | default([]) }}"
          loop_control:
            label: "IPsec PSK for {{ item.peer }}"
          when: ipsec is defined and ipsec.enabled | bool
          no_log: true    # PSK is sensitive
          notify: Save IOS configuration

      tags: [ipsec, vpn, security]

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands:
          - write memory
      listen: Save IOS configuration
```

---

## 18.4 — The CCNP Jinja2 Template Extension

This template file extends `templates/ios/wan_router.j2` from Part 17 with CCNP-specific sections. I create a separate file that can be included or stand alone:

```bash
cat > ~/projects/ansible-network/templates/ios/wan_router_ccnp.j2 << 'TEMPLATE'
{# =============================================================
   templates/ios/wan_router_ccnp.j2
   CCNP-level IOS-XE configuration extensions.
   Intended to follow wan_router.j2 sections 1-12.
   ============================================================= #}

{# ── VRF Definitions ─────────────────────────────────────────── #}
{% if vrfs is defined %}
{% for vrf in vrfs %}
vrf definition {{ vrf.name }}
 rd {{ vrf.rd }}
 description {{ vrf.description | default('') }}
 !
 address-family ipv4
{% for target in vrf.address_families[0].import_targets | default([]) %}
  route-target import {{ target }}
{% endfor %}
{% for target in vrf.address_families[0].export_targets | default([]) %}
  route-target export {{ target }}
{% endfor %}
 exit-address-family
!
{% endfor %}
{% endif %}

{# ── VRF Interface Assignments ───────────────────────────────── #}
{% if vrf_interfaces is defined %}
{% for iface_name, iface in vrf_interfaces.items() %}
interface {{ iface_name }}
 vrf forwarding {{ iface.vrf }}
 description {{ iface.description | default('') }}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
 no shutdown
!
{% endfor %}
{% endif %}

{# ── OSPF Multi-Area with Authentication ─────────────────────── #}
{% if ospf_advanced is defined %}
router ospf {{ ospf_advanced.process_id }}
 router-id {{ ospf_advanced.router_id }}
 auto-cost reference-bandwidth 10000
{# Passive interfaces #}
{% for area in ospf_advanced.areas %}
{% for iface in area.interfaces %}
{% if iface.passive | default(false) %}
 passive-interface {{ iface.name }}
{% endif %}
{% endfor %}
{% endfor %}
{# Stub areas #}
{% for area in ospf_advanced.areas | selectattr('type', 'equalto', 'stub') | list %}
 area {{ area.area_id }} stub
{% endfor %}
{# Area authentication #}
{% for area in ospf_advanced.areas | selectattr('authentication', 'defined') | list %}
 area {{ area.area_id }} authentication message-digest
{% endfor %}
!
{# Interface-level OSPF config #}
{% for area in ospf_advanced.areas %}
{% for iface in area.interfaces %}
interface {{ iface.name }}
 ip ospf {{ ospf_advanced.process_id }} area {{ area.area_id }}
{% if iface.type is defined %}
 ip ospf network {{ iface.type }}
{% endif %}
{% if iface.cost is defined %}
 ip ospf cost {{ iface.cost }}
{% endif %}
{% if iface.md5_key_id is defined %}
 ip ospf message-digest-key {{ iface.md5_key_id }} md5 <vaulted>
{% endif %}
!
{% endfor %}
{% endfor %}
{% endif %}

{# ── EIGRP Named Mode ────────────────────────────────────────── #}
{% if eigrp is defined %}
router eigrp {{ eigrp.process_name }}
 !
 address-family ipv4 unicast autonomous-system {{ eigrp.as_number }}
  !
  topology base
   variance {{ eigrp.variance | default(1) }}
   maximum-paths {{ eigrp.maximum_paths | default(4) }}
  exit-af-topology
  !
{% for net in eigrp.address_families[0].networks | default([]) %}
  network {{ net.prefix }} {{ net.wildcard }}
{% endfor %}
{% for iface in eigrp.address_families[0].passive_interfaces | default([]) %}
  af-interface {{ iface }}
   passive-interface
  exit-af-interface
{% endfor %}
 exit-address-family
!
{% endif %}

{# ── BGP Prefix Lists ────────────────────────────────────────── #}
{% if bgp_advanced is defined %}
{% for pl in bgp_advanced.prefix_lists | default([]) %}
{% for entry in pl.entries %}
ip prefix-list {{ pl.name }} seq {{ entry.seq }} {{ entry.action }} {{ entry.prefix }}{% if entry.le is defined %} le {{ entry.le }}{% endif %}{% if entry.ge is defined %} ge {{ entry.ge }}{% endif %}

{% endfor %}
!
{% endfor %}

{# ── BGP Route Maps ──────────────────────────────────────────── #}
{% for rm in bgp_advanced.route_maps | default([]) %}
{% for entry in rm.entries %}
route-map {{ rm.name }} {{ entry.action }} {{ entry.seq }}
{% if entry.match.ip_prefix_list is defined %}
 match ip address prefix-list {{ entry.match.ip_prefix_list }}
{% endif %}
{% if entry.set is defined %}
{% if entry.set.local_preference is defined %}
 set local-preference {{ entry.set.local_preference }}
{% endif %}
{% if entry.set.as_path_prepend is defined %}
 set as-path prepend {{ entry.set.as_path_prepend }}
{% endif %}
{% endif %}
!
{% endfor %}
{% endfor %}

{# ── BGP Advanced Neighbor Policy ────────────────────────────── #}
router bgp {{ bgp_advanced.as_number }}
{% for neighbor in bgp_advanced.neighbors | default([]) %}
{% if neighbor.route_map_in is defined %}
 neighbor {{ neighbor.ip }} route-map {{ neighbor.route_map_in }} in
{% endif %}
{% if neighbor.route_map_out is defined %}
 neighbor {{ neighbor.ip }} route-map {{ neighbor.route_map_out }} out
{% endif %}
{% if neighbor.soft_reconfiguration_inbound | default(false) %}
 neighbor {{ neighbor.ip }} soft-reconfiguration inbound
{% endif %}
{% if neighbor.send_community is defined %}
 neighbor {{ neighbor.ip }} send-community {{ neighbor.send_community }}
{% endif %}
{% if neighbor.next_hop_self | default(false) %}
 neighbor {{ neighbor.ip }} next-hop-self
{% endif %}
{% endfor %}
!
{% endif %}

{# ── Object Tracking ─────────────────────────────────────────── #}
{% if tracking is defined %}
{% for track in tracking %}
track {{ track.id }} interface {{ track.interface }} {{ track.track }}
!
{% endfor %}
{% endif %}

{# ── HSRP ────────────────────────────────────────────────────── #}
{% if hsrp is defined %}
{% for group in hsrp %}
interface {{ group.interface }}
 standby version {{ group.version | default(2) }}
 standby {{ group.group }} ip {{ group.virtual_ip }}
 standby {{ group.group }} priority {{ group.priority | default(100) }}
{% if group.preempt | default(false) %}
 standby {{ group.group }} preempt delay minimum {{ group.preempt_delay | default(0) }}
{% endif %}
 standby {{ group.group }} timers {{ group.timers.hello }} {{ group.timers.hold }}
{% if group.authentication is defined %}
 standby {{ group.group }} authentication md5 key-string <vaulted>
{% endif %}
{% for track in group.track | default([]) %}
 standby {{ group.group }} track {{ track.object }} decrement {{ track.decrement }}
{% endfor %}
!
{% endfor %}
{% endif %}

{# ── EtherChannel ────────────────────────────────────────────── #}
{% if port_channels is defined %}
{% for pc in port_channels %}
interface Port-channel{{ pc.id }}
 description {{ pc.description | default('') }}
{% if pc.layer == 3 %}
 ip address {{ pc.ip }} {{ pc.prefix | ansible.utils.ipaddr('netmask') }}
{% endif %}
 port-channel min-links {{ pc.min_links | default(1) }}
 no shutdown
!
{% for member in pc.members %}
interface {{ member }}
 channel-group {{ pc.id }} mode {{ pc.mode | default('active') }}
 no shutdown
!
{% endfor %}
{% endfor %}
{% endif %}

{# ── GRE Tunnels ─────────────────────────────────────────────── #}
{% if tunnels is defined %}
{% for tunnel in tunnels %}
interface Tunnel{{ tunnel.id }}
 description {{ tunnel.description | default('GRE Tunnel') }}
 ip address {{ tunnel.tunnel_ip }} {{ tunnel.prefix | ansible.utils.ipaddr('netmask') }}
 tunnel source {{ tunnel.source }}
 tunnel destination {{ tunnel.destination }}
{% if tunnel.keepalive is defined %}
 keepalive {{ tunnel.keepalive.period }} {{ tunnel.keepalive.retries }}
{% endif %}
 no shutdown
!
{% endfor %}
{% endif %}

{# ── QoS Class Maps ──────────────────────────────────────────── #}
{% if qos is defined %}
{% for cm in qos.class_maps | default([]) %}
class-map match-any {{ cm.name }}
{% for dscp_val in cm.match.dscp | default([]) %}
 match ip dscp {{ dscp_val }}
{% endfor %}
!
{% endfor %}

{# ── QoS Policy Maps ─────────────────────────────────────────── #}
{% for pm in qos.policy_maps | default([]) %}
policy-map {{ pm.name }}
{% for cls in pm.classes %}
 class {{ cls.name }}
{% if cls.priority_percent is defined %}
  priority percent {{ cls.priority_percent }}
{% elif cls.bandwidth_percent is defined %}
  bandwidth percent {{ cls.bandwidth_percent }}
{% if cls.queue_limit is defined %}
  queue-limit {{ cls.queue_limit }} packets
{% endif %}
{% if cls.fair_queue | default(false) %}
  fair-queue
{% endif %}
{% else %}
  fair-queue
{% endif %}
 !
{% endfor %}
!
{% endfor %}

{# ── QoS Service Policy Application ─────────────────────────── #}
{% for sp in qos.service_policy | default([]) %}
interface {{ sp.interface }}
 service-policy {{ sp.direction }} {{ sp.policy }}
!
{% endfor %}
{% endif %}

{# End of CCNP extension template #}
TEMPLATE
```

---

## 18.5 — Key Module Notes for CCNP Topics

### `ios_prefix_lists` — Resource Module (New in cisco.ios 4.x)

```yaml
cisco.ios.ios_prefix_lists:
  config:
    - name: ALLOW_DEFAULT
      address_family: ipv4
      prefixes:
        - sequence: 5
          action: permit
          prefix: 0.0.0.0/0
    - name: BLOCK_RFC1918
      address_family: ipv4
      prefixes:
        - sequence: 5
          action: deny
          prefix: 10.0.0.0/8
          le: 32
  state: merged    # or replaced to replace all entries in the list
```

If `ios_prefix_lists` isn't available (older collection version), use `ios_config`:

```yaml
cisco.ios.ios_config:
  lines:
    - "ip prefix-list ALLOW_DEFAULT seq 5 permit 0.0.0.0/0"
    - "ip prefix-list BLOCK_RFC1918 seq 5 deny 10.0.0.0/8 le 32"
```

### `ios_vrf` — VRF Resource Module

```yaml
cisco.ios.ios_vrf:
  vrfs:
    - name: CUSTOMER_A
      rd: "65100:100"
      description: "Customer A"
      route_import:
        - "65100:100"
      route_export:
        - "65100:100"
  state: present
```

### `ios_bgp_address_family` — BGP AF Resource Module

```yaml
cisco.ios.ios_bgp_address_family:
  config:
    as_number: "65100"
    address_family:
      - afi: ipv4
        safi: unicast
        neighbors:
          - neighbor_address: 10.10.10.2
            activate: true
            soft_reconfiguration_inbound: always
        networks:
          - address: 10.255.0.1
            mask: 255.255.255.255
  state: merged
```

### QoS — `ios_config` Only

QoS has no resource module as of the current `cisco.ios` collection. The `parents` + `lines` pattern with `match: line` is the correct approach:

```yaml
# The nested parent context (policy-map → class) is what makes
# ios_config.parents powerful for QoS — it handles 2-level nesting correctly
cisco.ios.ios_config:
  parents:
    - "policy-map WAN_EGRESS"
    - "class VOICE"
  lines:
    - "priority percent 10"
  match: line
```

### HSRP — `ios_config` Only

The `ios_hsrp` module exists in some collection versions but has incomplete parameter support for HSRP version 2, MD5 authentication, and object tracking. `ios_config` with explicit `parents: "interface ..."` is more reliable:

```yaml
cisco.ios.ios_config:
  parents: "interface GigabitEthernet2"
  lines:
    - "standby version 2"
    - "standby 10 ip 192.168.1.254"
    - "standby 10 priority 110"
    - "standby 10 preempt delay minimum 60"
    - "standby 10 timers 1 3"
    - "standby 10 authentication md5 key-string {{ vault_hsrp_key }}"
    - "standby 10 track 1 decrement 20"
  match: line    # ← line match is safe here — each standby line is unique
  no_log: true
```

---

## 18.6 — Common CCNP Automation Gotchas

### ### 🪲 Gotcha — `vrf forwarding` on an interface clears its IP address

This is the most disruptive IOS behavior for VRF automation. When I assign a VRF to an interface, IOS removes the IP address. The IP must be re-applied in the same `ios_config` task:

```yaml
# ❌ Wrong — two separate tasks lose the IP address
- name: Assign VRF
  cisco.ios.ios_config:
    parents: "interface GigabitEthernet4"
    lines:
      - "vrf forwarding CUSTOMER_A"

- name: Set IP (never runs correctly — interface lost IP when VRF was applied)
  cisco.ios.ios_l3_interfaces:
    config:
      - name: GigabitEthernet4
        ipv4:
          - address: 10.100.0.1/30
    state: merged

# ✅ Correct — VRF and IP in the same task
- name: Assign VRF and restore IP
  cisco.ios.ios_config:
    parents: "interface GigabitEthernet4"
    lines:
      - "vrf forwarding CUSTOMER_A"
      - "ip address 10.100.0.1 255.255.255.252"
      - "no shutdown"
```

### ### 🪲 Gotcha — BGP `clear ip bgp * soft` in `after:` runs on every play, not just on change

The `after:` parameter in `ios_config` only runs when the task reports `changed`. But `ios_config` with `match: line` may report `ok` even when BGP policy is already correct — meaning BGP never gets soft-cleared after the first apply.

```yaml
# More reliable pattern: register the result and conditionally run clear
- name: "BGP | Apply route map to neighbor"
  cisco.ios.ios_config:
    parents: "router bgp {{ bgp_advanced.as_number }}"
    lines:
      - "neighbor 10.10.10.2 route-map FROM_UPSTREAM in"
  register: bgp_policy_result
  notify: Save IOS configuration

- name: "BGP | Soft-reset BGP inbound if policy changed"
  cisco.ios.ios_command:
    commands:
      - "clear ip bgp 10.10.10.2 soft in"
  when: bgp_policy_result.changed
  changed_when: false
```

### ### 🪲 Gotcha — EIGRP named mode and classic mode can't coexist

If the device has `router eigrp 100` (classic mode) and I push `router eigrp ENTERPRISE` (named mode), they coexist without error — but they don't merge. Named mode doesn't inherit classic mode neighbors or networks. The `before:` parameter handles the transition:

```yaml
- name: "EIGRP | Migrate from classic to named mode"
  cisco.ios.ios_config:
    before:
      - "no router eigrp 100"    # Remove classic mode first
    parents: "router eigrp ENTERPRISE"
    lines:
      - "no shutdown"
  # before: only sends "no router eigrp 100" when the parent block is missing
  # Once named mode exists, before: still runs — use with care in production
```

### ### 🪲 Gotcha — QoS `match: line` on policy-map class entries can create duplicates

`ios_config` with `match: line` checks if the line exists anywhere under the parent. Inside a policy-map class, if I push `bandwidth percent 20` and later change it to `bandwidth percent 30`, the old line remains because `ios_config` only adds lines — it doesn't remove the old bandwidth statement:

```yaml
# ❌ This leaves both bandwidth statements in the config
cisco.ios.ios_config:
  parents:
    - "policy-map WAN_EGRESS"
    - "class VIDEO"
  lines:
    - "bandwidth percent 30"    # Old: bandwidth percent 20 still exists!
  match: line

# ✅ Use replace: block to replace the entire class config
cisco.ios.ios_config:
  parents:
    - "policy-map WAN_EGRESS"
    - "class VIDEO"
  lines:
    - "bandwidth percent 30"
    - "queue-limit 64 packets"
  replace: block    # ← Removes all existing lines under this class, applies new ones
```

### ### 🪲 Gotcha — LACP `channel-group` conflicts with existing port-channel config

If a physical interface already has a `channel-group` statement from a previous Ansible run or manual config, and the new config specifies a different port-channel ID, the task fails with a conflict error. Always verify and clean up existing channel-group assignments:

```yaml
- name: "EtherChannel | Pre-flight: check existing channel-group assignments"
  cisco.ios.ios_command:
    commands:
      - "show etherchannel summary"
  register: ec_check
  changed_when: false

- name: "EtherChannel | Display current EtherChannel state (pre-change)"
  ansible.builtin.debug:
    msg: "{{ ec_check.stdout_lines[0] }}"
    verbosity: 1
```

---


CCNP-level IOS automation is in place — VRFs, multi-area OSPF with authentication, EIGRP named mode, full BGP policy with prefix lists and route maps, HSRP with object tracking, EtherChannel, GRE tunnels, and QoS. Part 19 applies the same depth to Cisco NX-OS — the data center platform with its own modules, features, and automation patterns that differ significantly from IOS.


