---
draft: false
title: '19 - NXOS'
description: "Part 19 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 19
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 19: Cisco NX-OS Network Automation

> *NX-OS is a different operating system from IOS, not just a different version. It has a feature model, a separate CLI hierarchy for things like VRFs and BGP address families, and data center constructs — VPC, VXLAN, vPC peer-links — that don't exist in the IOS world. The `cisco.nxos` collection mirrors the `cisco.ios` collection in structure but differs in every detail. This part builds the complete NX-OS automation system for the spine-leaf lab topology: one data model per device, one master playbook, IOS vs NX-OS differences noted as each topic appears.*

---

## 19.1 — The NX-OS Feature Model

The first and most important NX-OS difference: **features must be explicitly enabled before any related configuration will work**. On IOS, BGP, OSPF, and LACP are always available. On NX-OS, attempting to configure BGP on a device where `feature bgp` hasn't been enabled returns an error.

```yaml
# This is the first task in every NX-OS playbook
- name: "Features | Enable required NX-OS features"
  cisco.nxos.nxos_feature:
    feature: "{{ item }}"
    state: enabled
  loop:
    - bgp
    - ospf
    - interface-vlan
    - lacp
    - lldp
    - vpc
    - vn-segment-vlan-based    # Required for VXLAN
    - nv overlay               # Required for VXLAN NVE
  loop_control:
    label: "feature {{ item }}"
```

Features are idempotent — enabling an already-enabled feature returns `ok`. Disabling a feature that has active configuration referencing it will fail — NX-OS protects against orphaned config. The `nxos_feature` module handles both states cleanly.

**IOS equivalent:** No equivalent. IOS features are always on. The closest analog is `license` activation on newer IOS-XE platforms, but even that doesn't gate individual protocol configuration the way NX-OS features do.

---

## 19.2 — The NX-OS Data Model

The lab has four NX-OS devices across two roles:

- `spine-01`, `spine-02` — NX-OS spine switches, eBGP route reflectors
- `leaf-01`, `leaf-02` — NX-OS leaf switches, VPC pairs, VXLAN VTEPs

I build the complete data model for all four devices. The spines share most configuration; the leaves share base config and diverge on VPC and VTEP details.

### Group Variables for NX-OS Devices

```bash
cat > ~/projects/ansible-network/inventory/group_vars/cisco_nxos.yml << 'EOF'
---
# Common settings for all NX-OS devices
ansible_network_os: cisco.nxos.nxos
ansible_connection: ansible.netcommon.network_cli
ansible_become: false       # NX-OS uses privilege levels differently — no enable by default

# NX-OS doesn't use become/enable the same way IOS does.
# admin user on NX-OS has privilege 15 equivalent by default.
# If the device requires privilege escalation:
# ansible_become: true
# ansible_become_method: enable

# Global NX-OS features required on all devices
nxos_base_features:
  - lldp
  - lacp
  - interface-vlan

# Underlay BGP AS assignments (spine-leaf model)
# Spines: AS 65000 (shared iBGP within spine tier)
# Leaves: unique AS per leaf (eBGP to spines)

ntp_servers:
  - 216.239.35.0
  - 216.239.35.4

syslog_server: 172.16.0.100
domain_name: lab.local
EOF
```

### Group Variables for Spine Switches

```bash
cat > ~/projects/ansible-network/inventory/group_vars/spine_switches.yml << 'EOF'
---
# Spine-specific settings
nxos_spine_features:
  - bgp
  - ospf

bgp_spine_as: 65000

# Base VLANs present on all spine switches
base_vlans: []    # Spines are pure L3 in this design — no VLANs
EOF
```

### Group Variables for Leaf Switches

```bash
cat > ~/projects/ansible-network/inventory/group_vars/leaf_switches.yml << 'EOF'
---
# Leaf-specific settings
nxos_leaf_features:
  - bgp
  - ospf
  - vpc
  - vn-segment-vlan-based
  - nv overlay
  - interface-vlan

# Base VLANs on all leaf switches
base_vlans:
  - id: 10
    name: SERVERS
    vni: 10010       # VXLAN VNI for this VLAN
  - id: 20
    name: STORAGE
    vni: 10020
  - id: 30
    name: MGMT
    vni: 10030
  - id: 99
    name: NATIVE
    vni: ~           # No VNI for native VLAN
EOF
```

### Host Variables — Spine-01

```bash
cat > ~/projects/ansible-network/inventory/host_vars/spine-01.yml << 'EOF'
---
# =============================================================
# spine-01 — NX-OS Spine Switch 1
# Role: eBGP route reflector, underlay routing
# =============================================================

device_hostname: spine-01
device_role: spine
ansible_host: 172.16.0.21

# ── Loopbacks ─────────────────────────────────────────────────────
loopbacks:
  loopback0:
    description: "Router-ID and BGP peering"
    ip: 10.0.0.1
    prefix: 32

# ── Fabric Interfaces (to leaves) ─────────────────────────────────
interfaces:
  Ethernet1/1:
    description: "Fabric | To leaf-01 Eth1/1"
    ip: 10.1.1.0
    prefix: 31
    mtu: 9216
    shutdown: false
  Ethernet1/2:
    description: "Fabric | To leaf-01 Eth1/2"
    ip: 10.1.1.2
    prefix: 31
    mtu: 9216
    shutdown: false
  Ethernet1/3:
    description: "Fabric | To leaf-02 Eth1/1"
    ip: 10.1.1.4
    prefix: 31
    mtu: 9216
    shutdown: false
  Ethernet1/4:
    description: "Fabric | To leaf-02 Eth1/2"
    ip: 10.1.1.6
    prefix: 31
    mtu: 9216
    shutdown: false

# ── BGP (eBGP to leaves, acting as route reflector for EVPN) ──────
bgp:
  as_number: 65000
  router_id: 10.0.0.1
  bestpath:
    as_path: multipath-relax    # Allow eBGP multipath despite different AS
  neighbors:
    - ip: 10.1.1.1
      remote_as: 65001
      description: "eBGP to leaf-01 (Eth1/1)"
      update_source: ~
      bfd: false
    - ip: 10.1.1.3
      remote_as: 65001
      description: "eBGP to leaf-01 (Eth1/2)"
    - ip: 10.1.1.5
      remote_as: 65002
      description: "eBGP to leaf-02 (Eth1/1)"
    - ip: 10.1.1.7
      remote_as: 65002
      description: "eBGP to leaf-02 (Eth1/2)"
  address_families:
    - afi: ipv4
      safi: unicast
      networks:
        - prefix: 10.0.0.1
          mask: 255.255.255.255
    - afi: l2vpn
      safi: evpn
      # EVPN AF — spines act as route reflectors for overlay
      # Details in Part 20 (VXLAN/EVPN deep dive)

# ── OSPF underlay (optional — BGP-only underlay is also valid) ────
ospf:
  process_id: 1
  router_id: 10.0.0.1
  areas:
    - area_id: 0
      interfaces:
        - name: Ethernet1/1
          type: point-to-point
          passive: false
        - name: Ethernet1/2
          type: point-to-point
          passive: false
        - name: Ethernet1/3
          type: point-to-point
          passive: false
        - name: Ethernet1/4
          type: point-to-point
          passive: false
        - name: loopback0
          passive: true

# ── Local Users ───────────────────────────────────────────────────
local_users:
  - username: ansible
    role: network-admin
    password: "{{ vault_ansible_password }}"
  - username: netadmin
    role: network-admin
    password: "{{ vault_netadmin_password }}"

snmp_location: "HQ-DC-Row1-Rack1-U1"
snmp_contact: "netops@lab.local"
EOF
```

### Host Variables — Leaf-01

```bash
cat > ~/projects/ansible-network/inventory/host_vars/leaf-01.yml << 'EOF'
---
# =============================================================
# leaf-01 — NX-OS Leaf Switch 1 (VPC Primary)
# Role: ToR switch, VTEP, VPC primary peer
# =============================================================

device_hostname: leaf-01
device_role: leaf
ansible_host: 172.16.0.31

# ── Loopbacks ─────────────────────────────────────────────────────
loopbacks:
  loopback0:
    description: "Router-ID and BGP peering"
    ip: 10.0.1.1
    prefix: 32
  loopback1:
    description: "VTEP source (shared with leaf-02 via anycast)"
    ip: 10.0.1.100
    prefix: 32
    # loopback1 on both leaf-01 and leaf-02 has the same IP (anycast VTEP)
    # This requires NX-OS secondary IP or a dedicated anycast loopback

# ── Uplink Interfaces (to spines) ─────────────────────────────────
interfaces:
  Ethernet1/1:
    description: "Uplink | To spine-01 Eth1/1"
    ip: 10.1.1.1
    prefix: 31
    mtu: 9216
    shutdown: false
  Ethernet1/2:
    description: "Uplink | To spine-02 Eth1/1"
    ip: 10.1.2.1
    prefix: 31
    mtu: 9216
    shutdown: false

# ── VPC Configuration ─────────────────────────────────────────────
vpc:
  domain_id: 10
  role_priority: 100          # Lower = primary (leaf-01 is primary)
  system_priority: 100
  peer_keepalive:
    destination: 172.16.99.2  # leaf-02 mgmt IP
    source: 172.16.99.1       # leaf-01 mgmt IP
    vrf: management
  peer_link:
    port_channel_id: 1
    members:
      - Ethernet1/3
      - Ethernet1/4
    description: "vPC Peer-Link to leaf-02"
  member_port_channels:
    - id: 10
      mode: active
      description: "vPC to Server-01"
      members:
        - Ethernet1/5
      vlan_mode: trunk
      trunk_vlans: "10,20,30"
      native_vlan: 99

# ── Local VLANs (device-specific, extends base_vlans from group_vars) ─
local_vlans:
  - id: 100
    name: "SERVER_CLUSTER_A"
    vni: 10100
    svi_ip: 10.100.0.1
    svi_prefix: 24

# ── SVIs (L3 VLAN interfaces) ─────────────────────────────────────
svis:
  - vlan_id: 10
    ip: 10.10.0.1
    prefix: 24
    description: "SVI VLAN10 SERVERS"
    vrf: TENANT_A
  - vlan_id: 20
    ip: 10.20.0.1
    prefix: 24
    description: "SVI VLAN20 STORAGE"
    vrf: TENANT_A

# ── VRFs ──────────────────────────────────────────────────────────
vrfs:
  - name: TENANT_A
    vni: 50001              # L3VNI for inter-VLAN routing in overlay
    rd: auto
    description: "Tenant A production VRF"
    address_families:
      - afi: ipv4
        import_targets:
          - "65001:50001"
        export_targets:
          - "65001:50001"

# ── BGP ───────────────────────────────────────────────────────────
bgp:
  as_number: 65001
  router_id: 10.0.1.1
  bestpath:
    as_path: multipath-relax
  neighbors:
    - ip: 10.1.1.0
      remote_as: 65000
      description: "eBGP to spine-01 (Eth1/1)"
    - ip: 10.1.2.0
      remote_as: 65000
      description: "eBGP to spine-02 (Eth1/1)"
  address_families:
    - afi: ipv4
      safi: unicast
      networks:
        - prefix: 10.0.1.1
          mask: 255.255.255.255
        - prefix: 10.0.1.100
          mask: 255.255.255.255

local_users:
  - username: ansible
    role: network-admin
    password: "{{ vault_ansible_password }}"

snmp_location: "HQ-DC-Row1-Rack2-U1"
snmp_contact: "netops@lab.local"
EOF
```

---

## 19.3 — The NX-OS Master Playbook

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_nxos.yml
```

```yaml
---
# =============================================================
# deploy_nxos.yml — Complete NX-OS deployment
# Data-driven: all values sourced from host_vars and group_vars
# Covers: features, identity, interfaces, VLANs, SVIs, VRFs,
#         VPC, BGP spine-leaf, port-channels, users, backup
#
# Common invocations:
#   Full run:        ansible-playbook deploy_nxos.yml
#   Features only:   ansible-playbook deploy_nxos.yml --tags features
#   BGP only:        ansible-playbook deploy_nxos.yml --tags bgp
#   VPC only:        ansible-playbook deploy_nxos.yml --tags vpc
#   Spines only:     ansible-playbook deploy_nxos.yml -l spine_switches
#   Single device:   ansible-playbook deploy_nxos.yml -l spine-01
# =============================================================

- name: "Deploy | NX-OS complete configuration"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  force_handlers: true

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:

    # ── Pre-flight ────────────────────────────────────────────────
    - name: "Pre-flight | Gather NX-OS facts"
      cisco.nxos.nxos_facts:
        gather_subset:
          - default
          - hardware
      tags: always

    - name: "Pre-flight | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device:   {{ inventory_hostname }}"
          - "Hostname: {{ ansible_net_hostname }}"
          - "Version:  {{ ansible_net_version }}"
          - "Model:    {{ ansible_net_model | default('unknown') }}"
        verbosity: 1
      tags: always

    # ── Backup ────────────────────────────────────────────────────
    - block:
        - name: "Backup | Create backup directory"
          ansible.builtin.file:
            path: "backups/cisco_nxos/{{ inventory_hostname }}"
            state: directory
            mode: '0755'
          delegate_to: localhost

        - name: "Backup | Capture running config"
          cisco.nxos.nxos_command:
            commands:
              - show running-config
          register: nxos_running_config

        - name: "Backup | Save to control node"
          ansible.builtin.copy:
            content: "{{ nxos_running_config.stdout[0] }}"
            dest: "backups/cisco_nxos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
            mode: '0644'
          delegate_to: localhost
      tags: backup

    # ── Section 1: Feature Enablement ─────────────────────────────
    # NX-OS difference: features must be enabled before configuration
    # IOS equivalent: not applicable — IOS protocols are always available
    - block:
        - name: "Features | Enable base features (all NX-OS devices)"
          cisco.nxos.nxos_feature:
            feature: "{{ item }}"
            state: enabled
          loop: "{{ nxos_base_features | default([]) }}"
          loop_control:
            label: "feature {{ item }}"
          notify: Save NX-OS configuration

        - name: "Features | Enable spine-specific features"
          cisco.nxos.nxos_feature:
            feature: "{{ item }}"
            state: enabled
          loop: "{{ nxos_spine_features | default([]) }}"
          loop_control:
            label: "feature {{ item }}"
          when: device_role == 'spine'
          notify: Save NX-OS configuration

        - name: "Features | Enable leaf-specific features"
          cisco.nxos.nxos_feature:
            feature: "{{ item }}"
            state: enabled
          loop: "{{ nxos_leaf_features | default([]) }}"
          loop_control:
            label: "feature {{ item }}"
          when: device_role == 'leaf'
          notify: Save NX-OS configuration

        - name: "Features | Verify critical features are enabled"
          cisco.nxos.nxos_command:
            commands:
              - show feature | include enabled
          register: feature_state
          changed_when: false

        - name: "Features | Assert BGP feature is enabled"
          ansible.builtin.assert:
            that:
              - "'bgp' in feature_state.stdout[0]"
            fail_msg: "BGP feature not enabled on {{ inventory_hostname }}"
            success_msg: "PASS: BGP feature enabled"
          when: "'bgp' in (nxos_spine_features | default([]) + nxos_leaf_features | default([]))"

      tags: [features, always]
      # Features tagged always — they must run before anything else
      # without features, all subsequent config tasks will fail

    # ── Section 2: Device Identity ────────────────────────────────
    # NX-OS difference: hostname module exists but banner uses nxos_config
    - block:
        - name: "Identity | Set hostname"
          cisco.nxos.nxos_hostname:
            config:
              hostname: "{{ device_hostname }}"
            state: merged
          notify: Save NX-OS configuration

        - name: "Identity | Set IP domain name"
          cisco.nxos.nxos_config:
            lines:
              - "ip domain-name {{ domain_name }}"
          notify: Save NX-OS configuration

        - name: "Identity | Configure login banner"
          cisco.nxos.nxos_banner:
            banner: login
            text: |
              **************************************************************************
              *  Authorized access only. All activity is monitored and logged.         *
              **************************************************************************
            state: present
          notify: Save NX-OS configuration

      tags: [identity, base]

    # ── Section 3: Interfaces ─────────────────────────────────────
    # NX-OS difference: interfaces default to L3 (no switchport) on Nexus 9000
    # On NX-OS, 'no switchport' is not needed for routed interfaces — they're L3 by default
    # 'switchport' must be explicitly added for L2 ports
    - block:
        - name: "Interfaces | Configure loopback interfaces"
          cisco.nxos.nxos_interfaces:
            config:
              - name: "{{ item.key }}"
                description: "{{ item.value.description }}"
                enabled: true
            state: merged
          loop: "{{ loopbacks | dict2items }}"
          loop_control:
            label: "{{ item.key }}"
          notify: Save NX-OS configuration

        - name: "Interfaces | Configure loopback IP addresses"
          cisco.nxos.nxos_l3_interfaces:
            config:
              - name: "{{ item.key }}"
                ipv4:
                  - address: "{{ item.value.ip }}/{{ item.value.prefix }}"
            state: merged
          loop: "{{ loopbacks | dict2items }}"
          loop_control:
            label: "{{ item.key }} — {{ item.value.ip }}/{{ item.value.prefix }}"
          notify: Save NX-OS configuration

        - name: "Interfaces | Configure fabric/uplink interfaces"
          cisco.nxos.nxos_interfaces:
            config:
              - name: "{{ item.key }}"
                description: "{{ item.value.description | default('') }}"
                mtu: "{{ item.value.mtu | default(1500) | string }}"
                enabled: "{{ not item.value.shutdown | default(false) }}"
            state: merged
          loop: "{{ interfaces | dict2items }}"
          loop_control:
            label: "{{ item.key }}"
          notify: Save NX-OS configuration

        - name: "Interfaces | Configure L3 IP addresses on fabric interfaces"
          cisco.nxos.nxos_l3_interfaces:
            config:
              - name: "{{ item.key }}"
                ipv4:
                  - address: "{{ item.value.ip }}/{{ item.value.prefix }}"
            state: merged
          loop: "{{ interfaces | dict2items | selectattr('value.ip', 'defined') | list }}"
          loop_control:
            label: "{{ item.key }} — {{ item.value.ip }}/{{ item.value.prefix }}"
          notify: Save NX-OS configuration

        - name: "Interfaces | Set jumbo MTU globally (NX-OS system-level)"
          cisco.nxos.nxos_config:
            lines:
              - "system jumbomtu 9216"
          when: interfaces | dict2items | selectattr('value.mtu', 'defined') |
                selectattr('value.mtu', 'equalto', 9216) | list | length > 0
          notify: Save NX-OS configuration
          # NX-OS difference: system jumbomtu must be set globally AND per-interface
          # IOS: only per-interface MTU setting is needed

      tags: [interfaces, base]

    # ── Section 4: VLANs ──────────────────────────────────────────
    # NX-OS difference: VTP doesn't exist — VLAN config is always local
    # NX-OS difference: VLANs require vn-segment assignment for VXLAN
    - block:
        - name: "VLANs | Configure base VLANs (from group_vars)"
          cisco.nxos.nxos_vlans:
            config:
              - vlan_id: "{{ item.id }}"
                name: "{{ item.name }}"
                state: active
            state: merged
          loop: "{{ base_vlans | default([]) }}"
          loop_control:
            label: "VLAN {{ item.id }} — {{ item.name }}"
          when: base_vlans is defined
          notify: Save NX-OS configuration

        - name: "VLANs | Configure local VLANs (from host_vars)"
          cisco.nxos.nxos_vlans:
            config:
              - vlan_id: "{{ item.id }}"
                name: "{{ item.name }}"
                state: active
            state: merged
          loop: "{{ local_vlans | default([]) }}"
          loop_control:
            label: "VLAN {{ item.id }} — {{ item.name }}"
          when: local_vlans is defined
          notify: Save NX-OS configuration

        - name: "VLANs | Assign VNI segments to VLANs (VXLAN mapping)"
          cisco.nxos.nxos_config:
            lines:
              - "vn-segment {{ item.vni }}"
            parents: "vlan {{ item.id }}"
          loop: "{{ (base_vlans | default([]) + local_vlans | default([]))
                    | selectattr('vni', 'defined')
                    | selectattr('vni', 'ne', none) | list }}"
          loop_control:
            label: "VLAN {{ item.id }} → VNI {{ item.vni }}"
          when: device_role == 'leaf'
          notify: Save NX-OS configuration
          # NX-OS difference: vn-segment is NX-OS-only — no IOS equivalent
          # This maps a VLAN to a VXLAN VNI — prerequisite for EVPN overlay

      tags: [vlans, switching, base]

    # ── Section 5: VRF Definition ─────────────────────────────────
    # NX-OS difference: VRF syntax differs — 'vrf context' not 'vrf definition'
    - block:
        - name: "VRF | Create VRF contexts"
          cisco.nxos.nxos_vrf:
            name: "{{ item.name }}"
            rd: "{{ item.rd }}"
            description: "{{ item.description | default('') }}"
            state: present
          loop: "{{ vrfs | default([]) }}"
          loop_control:
            label: "VRF {{ item.name }} RD {{ item.rd }}"
          when: vrfs is defined
          notify: Save NX-OS configuration

        - name: "VRF | Configure VRF address family and route targets"
          cisco.nxos.nxos_config:
            parents:
              - "vrf context {{ item.name }}"
            lines:
              - "address-family ipv4 unicast"
              - " route-target import {{ item.address_families[0].import_targets[0] }}"
              - " route-target export {{ item.address_families[0].export_targets[0] }}"
          loop: "{{ vrfs | default([]) }}"
          loop_control:
            label: "VRF {{ item.name }} route targets"
          when: vrfs is defined
          notify: Save NX-OS configuration
          # NX-OS difference: 'vrf context' vs IOS 'vrf definition'
          # NX-OS syntax:  vrf context TENANT_A
          # IOS syntax:    vrf definition TENANT_A

        - name: "VRF | Assign L3VNI to VRF (VXLAN overlay routing)"
          cisco.nxos.nxos_config:
            parents: "vrf context {{ item.name }}"
            lines:
              - "vni {{ item.vni }}"
          loop: "{{ vrfs | default([]) | selectattr('vni', 'defined') | list }}"
          loop_control:
            label: "VRF {{ item.name }} L3VNI {{ item.vni }}"
          when: vrfs is defined and device_role == 'leaf'
          notify: Save NX-OS configuration

      tags: [vrf, routing]

    # ── Section 6: SVI Configuration ──────────────────────────────
    # NX-OS difference: 'interface Vlan10' — capital V, no space before VLAN ID
    # IOS equivalent:   'interface Vlan10' — same syntax, different module needed
    - block:
        - name: "SVI | Configure SVI interfaces"
          cisco.nxos.nxos_interfaces:
            config:
              - name: "Vlan{{ item.vlan_id }}"
                description: "{{ item.description | default('SVI VLAN' + item.vlan_id | string) }}"
                enabled: true
            state: merged
          loop: "{{ svis | default([]) }}"
          loop_control:
            label: "Vlan{{ item.vlan_id }}"
          when: svis is defined
          notify: Save NX-OS configuration

        - name: "SVI | Configure SVI IP addresses"
          cisco.nxos.nxos_l3_interfaces:
            config:
              - name: "Vlan{{ item.vlan_id }}"
                ipv4:
                  - address: "{{ item.ip }}/{{ item.prefix }}"
            state: merged
          loop: "{{ svis | default([]) }}"
          loop_control:
            label: "Vlan{{ item.vlan_id }} — {{ item.ip }}/{{ item.prefix }}"
          when: svis is defined
          notify: Save NX-OS configuration

        - name: "SVI | Assign SVIs to VRFs"
          cisco.nxos.nxos_config:
            parents: "interface Vlan{{ item.vlan_id }}"
            lines:
              - "vrf member {{ item.vrf }}"
          loop: "{{ svis | default([]) | selectattr('vrf', 'defined') | list }}"
          loop_control:
            label: "Vlan{{ item.vlan_id }} → VRF {{ item.vrf }}"
          when: svis is defined
          notify: Save NX-OS configuration
          # NX-OS difference: 'vrf member' vs IOS 'vrf forwarding'
          # NX-OS syntax:  vrf member TENANT_A
          # IOS syntax:    vrf forwarding TENANT_A
          # Both clear the IP address when applied — re-apply IP after VRF assignment

      tags: [svi, vlans, routing]

    # ── Section 7: Port-Channels with LACP ────────────────────────
    # NX-OS difference: port-channel syntax is largely the same but
    # NX-OS uses 'channel-group N mode active' (identical to IOS LACP)
    # NX-OS port-channels default to L2 (switchport); IOS defaults to L3
    - block:
        - name: "Port-Channel | Create port-channel interface"
          cisco.nxos.nxos_interfaces:
            config:
              - name: "port-channel{{ item.id }}"
                description: "{{ item.description | default('') }}"
                enabled: true
            state: merged
          loop: "{{ vpc.member_port_channels | default([]) }}"
          loop_control:
            label: "port-channel{{ item.id }}"
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "Port-Channel | Configure trunk settings on port-channel"
          cisco.nxos.nxos_l2_interfaces:
            config:
              - name: "port-channel{{ item.id }}"
                mode: trunk
                trunk:
                  allowed_vlans: "{{ item.trunk_vlans }}"
                  native_vlan: "{{ item.native_vlan | default(1) }}"
            state: merged
          loop: "{{ vpc.member_port_channels | default([])
                    | selectattr('vlan_mode', 'equalto', 'trunk') | list }}"
          loop_control:
            label: "port-channel{{ item.id }} trunk"
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "Port-Channel | Add member interfaces"
          cisco.nxos.nxos_config:
            parents: "interface {{ item.1 }}"
            lines:
              - "channel-group {{ item.0.id }} mode {{ item.0.mode | default('active') }}"
              - "no shutdown"
          loop: "{{ vpc.member_port_channels | default([]) | subelements('members') }}"
          loop_control:
            label: "{{ item.1 }} → port-channel{{ item.0.id }}"
          when: vpc is defined
          notify: Save NX-OS configuration

      tags: [portchannel, lacp, switching]

    # ── Section 8: VPC Configuration ──────────────────────────────
    # NX-OS difference: VPC is NX-OS-only — no IOS equivalent
    # VPC allows two NX-OS switches to present a single logical switch to downstream devices
    #
    # SEQUENCING NOTE: VPC configuration has a specific required order:
    # 1. Enable 'feature vpc'         (done in features section)
    # 2. Create port-channel for peer-link (done above)
    # 3. Configure VPC domain
    # 4. Configure peer-keepalive
    # 5. Assign peer-link port-channel
    # 6. Configure VPC member port-channels
    #
    # The chicken-and-egg problem: the peer-link port-channel must exist
    # before 'vpc peer-link' can be issued, but 'vpc peer-link' must be
    # issued before VPC member ports work correctly.
    # Solution: create the port-channel first (done in Section 7),
    # then assign it as peer-link in this section.
    - block:
        - name: "VPC | Create peer-link port-channel"
          cisco.nxos.nxos_interfaces:
            config:
              - name: "port-channel{{ vpc.peer_link.port_channel_id }}"
                description: "{{ vpc.peer_link.description }}"
                enabled: true
            state: merged
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "VPC | Add peer-link member interfaces to port-channel"
          cisco.nxos.nxos_config:
            parents: "interface {{ item }}"
            lines:
              - "channel-group {{ vpc.peer_link.port_channel_id }} mode active"
              - "no shutdown"
          loop: "{{ vpc.peer_link.members | default([]) }}"
          loop_control:
            label: "{{ item }} → port-channel{{ vpc.peer_link.port_channel_id }} (peer-link)"
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "VPC | Configure VPC domain"
          cisco.nxos.nxos_vpc:
            domain: "{{ vpc.domain_id }}"
            role_priority: "{{ vpc.role_priority | default(32667) }}"
            system_priority: "{{ vpc.system_priority | default(32667) }}"
            pkl_dest: "{{ vpc.peer_keepalive.destination }}"
            pkl_src: "{{ vpc.peer_keepalive.source }}"
            pkl_vrf: "{{ vpc.peer_keepalive.vrf | default('management') }}"
            state: present
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "VPC | Designate port-channel as VPC peer-link"
          cisco.nxos.nxos_vpc_interface:
            portchannel: "{{ vpc.peer_link.port_channel_id }}"
            peer_link: true
            state: present
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "VPC | Assign VPC IDs to member port-channels"
          cisco.nxos.nxos_vpc_interface:
            portchannel: "{{ item.id }}"
            vpc: "{{ item.id }}"
            state: present
          loop: "{{ vpc.member_port_channels | default([]) }}"
          loop_control:
            label: "port-channel{{ item.id }} → VPC {{ item.id }}"
          when: vpc is defined
          notify: Save NX-OS configuration

        - name: "VPC | Verify VPC peer status"
          cisco.nxos.nxos_command:
            commands:
              - show vpc brief
          register: vpc_status
          changed_when: false
          when: vpc is defined

        - name: "VPC | Display VPC status"
          ansible.builtin.debug:
            msg: "{{ vpc_status.stdout_lines[0] }}"
          when: vpc is defined and vpc_status is not skipped

      tags: [vpc, switching]

    # ── Section 9: BGP Spine-Leaf ──────────────────────────────────
    # NX-OS difference: BGP on NX-OS requires 'feature bgp' (done in features)
    # NX-OS difference: 'address-family ipv4 unicast' syntax matches IOS
    # NX-OS difference: 'bestpath as-path multipath-relax' — same concept, same syntax
    # NX-OS difference: BGP neighbor templates ('template peer') exist on NX-OS, not IOS
    - block:
        - name: "BGP | Configure BGP process and router-id"
          cisco.nxos.nxos_bgp_global:
            config:
              as_number: "{{ bgp.as_number }}"
              router_id: "{{ bgp.router_id }}"
              log_neighbor_changes: true
              bestpath:
                as_path:
                  multipath_relax: "{{ true if bgp.bestpath.as_path == 'multipath-relax' else false }}"
            state: merged
          when: bgp is defined
          notify: Save NX-OS configuration

        - name: "BGP | Configure BGP neighbors"
          cisco.nxos.nxos_bgp_neighbor_address_family:
            config:
              as_number: "{{ bgp.as_number }}"
              neighbors:
                - neighbor_address: "{{ item.ip }}"
                  remote_as: "{{ item.remote_as }}"
                  description: "{{ item.description }}"
                  address_family:
                    - afi: ipv4
                      safi: unicast
                      send_community:
                        extended: true
                        standard: true
            state: merged
          loop: "{{ bgp.neighbors | default([]) }}"
          loop_control:
            label: "BGP neighbor {{ item.ip }} AS {{ item.remote_as }}"
          when: bgp is defined
          notify: Save NX-OS configuration

        - name: "BGP | Configure BGP address family networks"
          cisco.nxos.nxos_bgp_address_family:
            config:
              as_number: "{{ bgp.as_number }}"
              address_family:
                - afi: ipv4
                  safi: unicast
                  networks:
                    - prefix: "{{ item.prefix }}/{{ item.mask | ansible.utils.ipaddr('prefix') }}"
            state: merged
          loop: "{{ bgp.address_families[0].networks | default([]) }}"
          loop_control:
            label: "BGP network {{ item.prefix }}"
          when: bgp is defined and bgp.address_families is defined
          notify: Save NX-OS configuration

        - name: "BGP | Verify BGP neighbor state"
          cisco.nxos.nxos_command:
            commands:
              - show bgp ipv4 unicast summary
            wait_for:
              - result[0] contains Established
            retries: 12
            interval: 5
          register: bgp_summary
          changed_when: false
          when: bgp is defined

        - name: "BGP | Display BGP summary"
          ansible.builtin.debug:
            msg: "{{ bgp_summary.stdout_lines[0] }}"
          when: bgp is defined and bgp_summary is not skipped

      tags: [bgp, routing]

    # ── Section 10: OSPF Underlay ──────────────────────────────────
    # NX-OS difference: OSPF module syntax is nearly identical to IOS
    # NX-OS difference: 'feature ospf' must be enabled (done in features)
    - block:
        - name: "OSPF | Configure OSPF process"
          cisco.nxos.nxos_ospfv2:
            config:
              processes:
                - process_id: "{{ ospf.process_id | string }}"
                  router_id: "{{ ospf.router_id }}"
            state: merged
          when: ospf is defined
          notify: Save NX-OS configuration

        - name: "OSPF | Assign interfaces to OSPF areas"
          cisco.nxos.nxos_config:
            parents: "interface {{ item.0.name }}"
            lines:
              - "ip router ospf {{ ospf.process_id }} area {{ item.1.area_id }}"
              - "{% if item.0.type is defined %}ip ospf network {{ item.0.type }}{% endif %}"
          loop: "{{ ospf.areas | subelements('interfaces') | map('reverse') | list }}"
          loop_control:
            label: "{{ item.1.name }} → OSPF area {{ item.0.area_id }}"
          when: ospf is defined
          notify: Save NX-OS configuration
          # NX-OS difference: 'ip router ospf 1 area 0' vs IOS 'ip ospf 1 area 0'
          # NX-OS adds 'router' in the keyword — easy to miss

      tags: [ospf, routing]

    # ── Section 11: NTP and Logging ────────────────────────────────
    # NX-OS difference: nxos_ntp_global module vs ios_ntp_global — same concept
    - block:
        - name: "NTP | Configure NTP servers"
          cisco.nxos.nxos_config:
            lines:
              - "ntp server {{ item }} use-vrf management"
          loop: "{{ ntp_servers | default([]) }}"
          loop_control:
            label: "NTP: {{ item }}"
          notify: Save NX-OS configuration
          # NX-OS difference: 'use-vrf management' routes NTP via management VRF
          # IOS equivalent: 'ntp server X source Mgmt0' for management VRF

        - name: "Logging | Configure syslog"
          cisco.nxos.nxos_logging_global:
            config:
              hosts:
                - hostname: "{{ syslog_server }}"
                  severity: informational
                  use_vrf: management
            state: merged
          notify: Save NX-OS configuration

      tags: [ntp, logging, services, base]

    # ── Section 12: SNMP ───────────────────────────────────────────
    - block:
        - name: "SNMP | Configure SNMP community and location"
          cisco.nxos.nxos_config:
            lines:
              - "snmp-server community {{ snmp_community_ro }} group network-operator"
              - "snmp-server location {{ snmp_location | default('Unknown') }}"
              - "snmp-server contact {{ snmp_contact | default('netops@lab.local') }}"
          no_log: true
          notify: Save NX-OS configuration
          # NX-OS difference: 'group network-operator' vs IOS 'RO'
          # NX-OS uses RBAC groups for SNMP access control
          # network-operator = read-only equivalent

      tags: [snmp, services, base]

    # ── Section 13: Local Users and AAA ────────────────────────────
    # NX-OS difference: user roles ('network-admin', 'network-operator')
    # instead of IOS privilege levels (1-15)
    - block:
        - name: "Users | Configure local user accounts"
          cisco.nxos.nxos_user:
            name: "{{ item.username }}"
            role: "{{ item.role }}"
            configured_password: "{{ item.password }}"
            state: present
          loop: "{{ local_users | default([]) }}"
          loop_control:
            label: "User: {{ item.username }} ({{ item.role }})"
          no_log: true
          notify: Save NX-OS configuration
          # NX-OS difference: roles instead of privilege levels
          # NX-OS role 'network-admin' ≈ IOS privilege 15
          # NX-OS role 'network-operator' ≈ IOS privilege 1

        - name: "Users | Configure SSH key-based auth"
          cisco.nxos.nxos_config:
            lines:
              - "username {{ item.username }} sshkey {{ lookup('file', '~/.ssh/ansible_ed25519.pub') }}"
          loop: "{{ local_users | default([]) | selectattr('role', 'equalto', 'network-admin') | list }}"
          loop_control:
            label: "SSH key for {{ item.username }}"
          when: local_users is defined
          notify: Save NX-OS configuration

      tags: [users, security, aaa]

    # ── Section 14: VXLAN/EVPN Overview ───────────────────────────
    # Full VXLAN/EVPN automation is covered in Part 20.
    # This section enables the prerequisite features and confirms
    # the fabric is ready for overlay configuration.
    - block:
        - name: "VXLAN | Verify NVE feature is enabled"
          cisco.nxos.nxos_command:
            commands:
              - show feature | include nv
          register: nve_feature_check
          changed_when: false
          when: device_role == 'leaf'

        - name: "VXLAN | Assert NVE overlay feature is enabled"
          ansible.builtin.assert:
            that:
              - "'nv overlay' in nve_feature_check.stdout[0]"
            fail_msg: "NV overlay feature not enabled on {{ inventory_hostname }} — required for VXLAN"
            success_msg: "PASS: NV overlay feature enabled on {{ inventory_hostname }}"
          when: device_role == 'leaf' and nve_feature_check is not skipped

        - name: "VXLAN | Verify VNI segment feature is enabled"
          cisco.nxos.nxos_command:
            commands:
              - show feature | include vn-segment
          register: vni_feature_check
          changed_when: false
          when: device_role == 'leaf'

        - name: "VXLAN | Display fabric readiness summary"
          ansible.builtin.debug:
            msg:
              - "VXLAN fabric readiness for {{ inventory_hostname }}:"
              - "NV overlay: {{ 'ENABLED' if 'nv overlay' in nve_feature_check.stdout[0] | default('') else 'NOT ENABLED' }}"
              - "VN-segment: {{ 'ENABLED' if 'vn-segment' in vni_feature_check.stdout[0] | default('') else 'NOT ENABLED' }}"
              - "VNI-VLAN mappings: {{ (base_vlans | default([]) + local_vlans | default([]))
                                        | selectattr('vni', 'defined')
                                        | selectattr('vni', 'ne', none)
                                        | list | length }} VLANs mapped"
              - "See Part 20 for full VXLAN/EVPN NVE and BGP EVPN configuration."
          when: device_role == 'leaf'

      tags: [vxlan, evpn]

  handlers:
    - name: Save NX-OS configuration
      cisco.nxos.nxos_command:
        commands:
          - copy running-config startup-config
      listen: Save NX-OS configuration
      # NX-OS difference: 'copy running-config startup-config' vs IOS 'write memory'
      # Both work on NX-OS but 'copy run start' is the NX-OS convention
```

---

## 19.4 — NX-OS Module Reference

Quick reference for the key differences from the IOS equivalents.

### `nxos_facts` — Device Information

```yaml
cisco.nxos.nxos_facts:
  gather_subset:
    - default      # hostname, version, model, serial
    - hardware     # memory, flash
    - interfaces   # interface state and IPs
    - legacy       # older-format facts (ansible_net_* variables)
```

Key populated variables: `ansible_net_hostname`, `ansible_net_version`, `ansible_net_model`, `ansible_net_interfaces`. The `legacy` subset populates the same `ansible_net_*` variable names as `ios_facts` — useful when writing cross-platform playbooks.

### `nxos_feature` — Feature Enablement

```yaml
cisco.nxos.nxos_feature:
  feature: bgp       # bgp | ospf | lacp | vpc | lldp | interface-vlan
                     # vn-segment-vlan-based | nv overlay | bfd | etc.
  state: enabled     # enabled | disabled
# No IOS equivalent — IOS protocols are always available
```

### `nxos_vrf` — VRF Context

```yaml
cisco.nxos.nxos_vrf:
  name: TENANT_A
  rd: "65001:100"
  description: "Tenant A"
  state: present
# NX-OS: 'vrf context TENANT_A'
# IOS:   'vrf definition TENANT_A'
```

### `nxos_vlans` — VLAN Configuration

```yaml
cisco.nxos.nxos_vlans:
  config:
    - vlan_id: 10
      name: SERVERS
      state: active
  state: merged
# Nearly identical to ios_vlans
# NX-OS: no VTP concern — all VLAN config is local
```

### `nxos_bgp_global` — BGP Global Config

```yaml
cisco.nxos.nxos_bgp_global:
  config:
    as_number: "65001"
    router_id: 10.0.1.1
    log_neighbor_changes: true
    bestpath:
      as_path:
        multipath_relax: true    # NX-OS parameter name differs from IOS resource module
  state: merged
```

### `nxos_bgp_address_family` — BGP AF

```yaml
cisco.nxos.nxos_bgp_address_family:
  config:
    as_number: "65001"
    address_family:
      - afi: ipv4
        safi: unicast
        networks:
          - prefix: 10.0.1.1/32
      - afi: l2vpn
        safi: evpn          # EVPN address family — NX-OS only
  state: merged
```

### `nxos_vpc` — VPC Domain

```yaml
cisco.nxos.nxos_vpc:
  domain: 10
  role_priority: 100
  system_priority: 100
  pkl_dest: 172.16.99.2     # Peer-keepalive destination
  pkl_src: 172.16.99.1      # Peer-keepalive source
  pkl_vrf: management
  state: present
# No IOS equivalent
```

### `nxos_vpc_interface` — VPC Interface Assignment

```yaml
# Designate as peer-link
cisco.nxos.nxos_vpc_interface:
  portchannel: 1
  peer_link: true
  state: present

# Assign VPC ID to member port-channel
cisco.nxos.nxos_vpc_interface:
  portchannel: 10
  vpc: 10
  state: present
# No IOS equivalent
```

### `nxos_user` — Local User Accounts

```yaml
cisco.nxos.nxos_user:
  name: ansible
  role: network-admin       # NX-OS role vs IOS privilege level
  configured_password: "{{ vault_password }}"
  state: present
# NX-OS: role-based (network-admin, network-operator, custom roles)
# IOS:   privilege-based (privilege 15, privilege 1)
```

---

## 19.5 — IOS vs NX-OS: Consolidated Differences

Throughout this part, differences were noted inline. Here they are consolidated for reference:

| Topic | NX-OS | IOS |
|---|---|---|
| Protocol availability | `feature bgp`, `feature ospf` required | Always available |
| VRF syntax | `vrf context NAME` | `vrf definition NAME` |
| VRF interface assignment | `vrf member NAME` | `vrf forwarding NAME` |
| Interface default mode | L3 (routed) on N9K | L2 (switchport) on most platforms |
| Jumbo MTU | `system jumbomtu 9216` + per-interface | Per-interface only |
| OSPF interface assignment | `ip router ospf 1 area 0` | `ip ospf 1 area 0` |
| SNMP access control | `group network-operator` | `RO` / `RW` |
| User access control | Roles (`network-admin`) | Privilege levels (15) |
| Save configuration | `copy running-config startup-config` | `write memory` |
| VLAN scope | Always local | Depends on VTP mode |
| VNI/VXLAN support | Native (`vn-segment`) | Requires IOS-XE VXLAN config |
| Peer redundancy | VPC (Virtual Port Channel) | VSS / StackWise (different platforms) |
| NTP VRF routing | `ntp server X use-vrf management` | `ntp server X source Mgmt0` |
| Syslog VRF routing | `use-vrf management` option | `source-interface` option |
| BGP EVPN AF | `address-family l2vpn evpn` (native) | Requires EVPN feature on IOS-XE |

---

## 19.6 — NX-OS Configuration Backup and Restore

NX-OS backup and restore follows the same pattern as IOS but uses different commands and has additional checkpoint capabilities.

### Backup Playbook

```yaml
# playbooks/backup/backup_nxos.yml
- name: "Backup | NX-OS configuration backup"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "backups/cisco_nxos/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Capture running configuration"
      cisco.nxos.nxos_command:
        commands:
          - show running-config
      register: nxos_config

    - name: "Backup | Save running config to control node"
      ansible.builtin.copy:
        content: "{{ nxos_config.stdout[0] }}"
        dest: "backups/cisco_nxos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Create NX-OS checkpoint on device"
      cisco.nxos.nxos_command:
        commands:
          - "checkpoint ansible_backup_{{ timestamp }}"
      # NX-OS difference: 'checkpoint' creates a named rollback point ON the device
      # IOS: no equivalent — rollback requires restoring from external backup file
      # NX-OS checkpoints: 'show checkpoint summary', 'rollback running-config checkpoint NAME'

    - name: "Backup | Show checkpoint summary"
      cisco.nxos.nxos_command:
        commands:
          - show checkpoint summary
      register: checkpoint_summary
      changed_when: false

    - name: "Backup | Display checkpoint list"
      ansible.builtin.debug:
        msg: "{{ checkpoint_summary.stdout_lines[0] }}"
        verbosity: 1
```

### Restore from Backup

```yaml
# playbooks/rollback/rollback_nxos.yml
- name: "Rollback | NX-OS configuration restore"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Rollback | Find most recent backup"
      ansible.builtin.find:
        paths: "backups/cisco_nxos/{{ inventory_hostname }}"
        patterns: "*.cfg"
        age_stamp: mtime
      register: backup_files
      delegate_to: localhost

    - name: "Rollback | Identify most recent backup file"
      ansible.builtin.set_fact:
        latest_backup: "{{ backup_files.files | sort(attribute='mtime') | last }}"

    - name: "Rollback | Display backup to be restored"
      ansible.builtin.debug:
        msg:
          - "Restoring {{ inventory_hostname }} from: {{ latest_backup.path }}"
          - "Backup date: {{ '%Y-%m-%d %H:%M:%S' | strftime(latest_backup.mtime) }}"

    - name: "Rollback | Pause for confirmation"
      ansible.builtin.pause:
        prompt: "Press ENTER to restore {{ inventory_hostname }} or Ctrl+C then A to abort"

    - name: "Rollback | Push backup configuration"
      cisco.nxos.nxos_config:
        src: "{{ latest_backup.path }}"
        replace: config    # Replace entire running config
      register: rollback_result

    - name: "Rollback | Save restored configuration"
      cisco.nxos.nxos_command:
        commands:
          - copy running-config startup-config
      when: rollback_result.changed
```

---

## 19.7 — Common NX-OS Automation Gotchas

### ### 🪲 Gotcha — `vrf member` clears the interface IP (same as IOS)

Identical to the IOS `vrf forwarding` behavior from Part 18 — applying `vrf member` to an interface clears its IP address. Always set VRF and IP in the same `nxos_config` task:

```yaml
# ✅ Correct for NX-OS
cisco.nxos.nxos_config:
  parents: "interface Ethernet1/10"
  lines:
    - "vrf member TENANT_A"
    - "ip address 10.100.0.1/30"
    - "no shutdown"
# vrf member and ip address in the same lines: list
```

### ### 🪲 Gotcha — `nxos_bgp_global` `as_number` must be a string, not an integer

```yaml
# ❌ Fails — as_number as integer
cisco.nxos.nxos_bgp_global:
  config:
    as_number: 65001    # ← Integer causes type error

# ✅ Correct — as_number as string
cisco.nxos.nxos_bgp_global:
  config:
    as_number: "65001"  # ← Must be quoted string
```

### ### 🪲 Gotcha — `nxos_feature` disable order matters

Disabling a feature while dependent configuration exists fails silently or with a cryptic error. Always remove dependent config before disabling:

```yaml
# ❌ Fails if BGP neighbors/networks are configured
cisco.nxos.nxos_feature:
  feature: bgp
  state: disabled

# ✅ Remove BGP config first, then disable
- name: "Remove BGP configuration"
  cisco.nxos.nxos_bgp_global:
    config:
      as_number: "65001"
    state: deleted

- name: "Disable BGP feature"
  cisco.nxos.nxos_feature:
    feature: bgp
    state: disabled
```

### ### 🪲 Gotcha — NX-OS `copy running-config startup-config` interactive prompt

On some NX-OS versions, `copy running-config startup-config` prompts for a filename confirmation. The `nxos_command` module handles this automatically by detecting and answering the prompt, but only when using `network_cli`. If using `raw` connection, the prompt causes a timeout.

### ### 🪲 Gotcha — OSPF keyword difference: `ip router ospf` vs `ip ospf`

The single most common mistake when converting IOS OSPF playbooks to NX-OS:

```yaml
# NX-OS interface OSPF assignment
cisco.nxos.nxos_config:
  parents: "interface Ethernet1/1"
  lines:
    - "ip router ospf 1 area 0"    # NX-OS: 'router' is required
    # NOT: "ip ospf 1 area 0"      # IOS syntax — fails on NX-OS

# Both produce the same result: the interface joins OSPF area 0
# but the CLI keyword differs between platforms
```

---


The complete NX-OS automation system is in place — features, interfaces, VLANs with VNI mappings, VRFs, SVIs, VPC, BGP spine-leaf, port-channels, users, and backup. The VXLAN/EVPN fabric is validated as ready. Part 20 builds the full overlay: NVE interface, BGP EVPN address family, VTEP configuration, and end-to-end VXLAN verification.


