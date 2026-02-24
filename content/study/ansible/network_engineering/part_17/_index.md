---
draft: true
title: '17 - IOS for CCNA'
weight: 17
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 17: Cisco IOS/IOS-XE Network Automation (CCNA Level)

> *This part covers every configuration domain a CCNA-level engineer works with on IOS/IOS-XE — interfaces, VLANs, STP, routing, ACLs, NAT, SSH hardening, and services. Rather than a collection of disconnected examples, everything is built around a single data model and a master playbook that feeds from it. The result is a complete, data-driven IOS automation system where changing a VLAN, an interface IP, or an ACL entry means editing one YAML file and running one playbook.*

---

## 17.1 — The Complete IOS Data Model

Before writing a single task, I define the complete data model that drives everything in this part. Every configuration domain gets its own key in `host_vars`. This extends the data model started in Part 11.

The lab device for this part is `wan-r1` — a Cisco IOS-XE router. The full `host_vars/wan-r1.yml` after this part:

```bash
cat > ~/projects/ansible-network/inventory/host_vars/wan-r1.yml << 'EOF'
---
# =============================================================
# wan-r1 — Cisco IOS-XE WAN Router 1
# Complete CCNA-level data model — drives all configuration
# =============================================================

# ── Device Identity ───────────────────────────────────────────────
device_hostname: wan-r1
device_role: wan_router
device_platform: ios-xe
ansible_host: 172.16.0.11

# ── Loopback Interfaces ───────────────────────────────────────────
loopbacks:
  Loopback0:
    description: "Loopback0 | Router-ID and MGT reachability"
    ip: 10.255.0.1
    prefix: 32

# ── Physical Interfaces ───────────────────────────────────────────
interfaces:
  GigabitEthernet1:
    description: "WAN | To FW-01 eth1 (outside)"
    ip: 10.10.10.1
    prefix: 30
    shutdown: false
    nat_outside: true            # Used by NAT template section
  GigabitEthernet2:
    description: "LAN | To internal switch"
    ip: 192.168.1.1
    prefix: 24
    shutdown: false
    nat_inside: true             # Used by NAT template section
  GigabitEthernet3:
    description: "WAN | To WAN-R2 inter-router link"
    ip: 10.10.20.1
    prefix: 30
    shutdown: false

# ── VLANs (for Layer 2 switch ports if present) ───────────────────
vlans:
  - id: 10
    name: MANAGEMENT
  - id: 20
    name: SERVERS
  - id: 30
    name: USERS
  - id: 99
    name: NATIVE

# ── Switched Interfaces (trunk/access) ────────────────────────────
# (Applicable when IOS-XE device has switchport capabilities)
switched_interfaces:
  GigabitEthernet4:
    mode: access
    access_vlan: 20
    spanning_tree:
      portfast: true
      bpduguard: true
  GigabitEthernet5:
    mode: trunk
    trunk_vlans: "10,20,30"
    native_vlan: 99

# ── OSPF ──────────────────────────────────────────────────────────
ospf:
  process_id: 1
  router_id: 10.255.0.1
  areas:
    - area_id: 0
      interfaces:
        - name: GigabitEthernet2
          type: broadcast
          passive: false
        - name: GigabitEthernet3
          type: point-to-point
          passive: false
        - name: Loopback0
          passive: true

# ── Static Routes ─────────────────────────────────────────────────
static_routes:
  - destination: 0.0.0.0
    mask: 0.0.0.0
    next_hop: 10.10.10.2
    description: "Default route to FW-01"
    admin_distance: 254         # Floating static — OSPF wins unless it fails
  - destination: 10.20.0.0
    mask: 255.255.0.0
    next_hop: 10.10.20.2
    description: "Summary to spine fabric"

# ── BGP ───────────────────────────────────────────────────────────
bgp:
  as_number: 65100
  router_id: 10.255.0.1
  neighbors:
    - ip: 10.10.10.2
      remote_as: 65200
      description: "eBGP to FW-01"
  address_families:
    - afi: ipv4
      networks:
        - prefix: 10.255.0.1
          mask: 255.255.255.255
        - prefix: 192.168.1.0
          mask: 255.255.255.0

# ── ACLs ──────────────────────────────────────────────────────────
acls:
  - name: MGMT_ACCESS
    type: standard
    remarks:
      - "Permit management network access"
    entries:
      - sequence: 10
        action: permit
        source: 192.168.1.0
        wildcard: 0.0.0.255
      - sequence: 20
        action: deny
        source: any
        log: true

  - name: INTERNET_FILTER
    type: extended
    remarks:
      - "Block inbound RFC1918 from WAN"
    entries:
      - sequence: 10
        action: deny
        protocol: ip
        source: 10.0.0.0
        source_wildcard: 0.255.255.255
        destination: any
      - sequence: 20
        action: deny
        protocol: ip
        source: 172.16.0.0
        source_wildcard: 0.15.255.255
        destination: any
      - sequence: 30
        action: deny
        protocol: ip
        source: 192.168.0.0
        source_wildcard: 0.0.255.255
        destination: any
      - sequence: 40
        action: permit
        protocol: ip
        source: any
        destination: any

# ── NAT ───────────────────────────────────────────────────────────
nat:
  enabled: true
  type: overload                # PAT / NAT overload
  acl_name: NAT_INSIDE_SOURCES  # Standard ACL that defines what to NAT
  inside_sources:
    - network: 192.168.1.0
      wildcard: 0.0.0.255
  # nat_outside interface: GigabitEthernet1 (defined in interfaces.nat_outside)
  # nat_inside interface:  GigabitEthernet2 (defined in interfaces.nat_inside)

# ── SSH and Local Users ───────────────────────────────────────────
ssh:
  version: 2
  timeout: 60
  authentication_retries: 3
  rsa_bits: 4096
  vty_access_class: MGMT_ACCESS

local_users:
  - username: ansible
    privilege: 15
    secret: "{{ vault_ansible_enable_secret }}"  # Vaulted
  - username: netadmin
    privilege: 15
    secret: "{{ vault_netadmin_secret }}"         # Vaulted

# ── NTP (device-specific overrides — global NTP in group_vars/all.yml) ─
ntp_source_interface: Loopback0

# ── Device-specific SNMP overrides ────────────────────────────────
snmp_location: "HQ-DC-Rack-A1-U12"
snmp_contact: "netops@lab.local"
EOF
```

---

## 17.2 — The Master Playbook Structure

One master playbook, organized into blocks by configuration domain:

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_ios_ccna.yml
```

```yaml
---
# =============================================================
# deploy_ios_ccna.yml — Complete CCNA-level IOS-XE deployment
# Data-driven: all values sourced from host_vars and group_vars
# Each section tagged independently for selective execution
#
# Common invocations:
#   Full deploy:     ansible-playbook deploy_ios_ccna.yml
#   Identity only:   ansible-playbook deploy_ios_ccna.yml --tags identity
#   Routing only:    ansible-playbook deploy_ios_ccna.yml --tags routing
#   ACLs only:       ansible-playbook deploy_ios_ccna.yml --tags acl
#   Skip backup:     ansible-playbook deploy_ios_ccna.yml --skip-tags backup
#   Single device:   ansible-playbook deploy_ios_ccna.yml -l wan-r1
# =============================================================

- name: "Deploy | Complete CCNA-level IOS-XE configuration"
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

    - name: "Pre-flight | Assert required data model variables"
      ansible.builtin.assert:
        that:
          - device_hostname is defined
          - interfaces is defined
        fail_msg: "Missing required data model variables for {{ inventory_hostname }}"
      tags: always

    # ── Backup (runs first) ───────────────────────────────────────
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost
      tags: backup

    - name: "Backup | Capture pre-change running config"
      cisco.ios.ios_command:
        commands: [show running-config]
      register: pre_change_config
      tags: backup

    - name: "Backup | Save to control node"
      ansible.builtin.copy:
        content: "{{ pre_change_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost
      tags: backup

    # ── Section 1: Identity ───────────────────────────────────────
    - block:
        - name: "Identity | Set hostname"
          cisco.ios.ios_hostname:
            config:
              hostname: "{{ device_hostname }}"
            state: merged
          notify: Save IOS configuration

        - name: "Identity | Set IP domain name"
          cisco.ios.ios_config:
            lines:
              - "ip domain-name {{ domain_name }}"
          notify: Save IOS configuration

        - name: "Identity | Configure login banner"
          cisco.ios.ios_banner:
            banner: login
            text: "{{ ios_banner_text | default(default_banner) }}"
            state: present
          vars:
            default_banner: |
              **************************************************************************
              *  Authorized access only. Activity is monitored and logged.             *
              **************************************************************************
          notify: Save IOS configuration

      tags: [identity, base]

    # ── Section 2: Interfaces ─────────────────────────────────────
    - block:
        - name: "Interfaces | Configure loopback interfaces"
          cisco.ios.ios_l3_interfaces:
            config:
              - name: "{{ item.key }}"
                ipv4:
                  - address: "{{ item.value.ip }}/{{ item.value.prefix }}"
            state: merged
          loop: "{{ loopbacks | dict2items }}"
          loop_control:
            label: "{{ item.key }} — {{ item.value.ip }}/{{ item.value.prefix }}"
          notify: Save IOS configuration

        - name: "Interfaces | Configure loopback descriptions"
          cisco.ios.ios_interfaces:
            config:
              - name: "{{ item.key }}"
                description: "{{ item.value.description }}"
                enabled: true
            state: merged
          loop: "{{ loopbacks | dict2items }}"
          loop_control:
            label: "{{ item.key }}"
          notify: Save IOS configuration

        - name: "Interfaces | Configure L3 interface IP addresses"
          cisco.ios.ios_l3_interfaces:
            config:
              - name: "{{ item.key }}"
                ipv4:
                  - address: "{{ item.value.ip }}/{{ item.value.prefix }}"
            state: merged
          loop: "{{ interfaces | dict2items | selectattr('value.ip', 'defined') | list }}"
          loop_control:
            label: "{{ item.key }} — {{ item.value.ip }}/{{ item.value.prefix }}"
          notify: Save IOS configuration

        - name: "Interfaces | Configure interface descriptions and state"
          cisco.ios.ios_interfaces:
            config:
              - name: "{{ item.key }}"
                description: "{{ item.value.description | default('') }}"
                enabled: "{{ not item.value.shutdown | default(false) }}"
            state: merged
          loop: "{{ interfaces | dict2items }}"
          loop_control:
            label: "{{ item.key }}"
          notify: Save IOS configuration

      tags: [interfaces, base]

    # ── Section 3: VLANs ──────────────────────────────────────────
    - block:
        - name: "VLANs | Configure VLANs"
          cisco.ios.ios_vlans:
            config:
              - vlan_id: "{{ item.id }}"
                name: "{{ item.name }}"
                state: active
            state: merged
          loop: "{{ vlans | default([]) }}"
          loop_control:
            label: "VLAN {{ item.id }} — {{ item.name }}"
          when: vlans is defined
          notify: Save IOS configuration

        - name: "VLANs | Configure access ports"
          cisco.ios.ios_l2_interfaces:
            config:
              - name: "{{ item.key }}"
                mode: access
                access:
                  vlan: "{{ item.value.access_vlan }}"
            state: merged
          loop: >-
            {{ switched_interfaces | default({}) | dict2items
               | selectattr('value.mode', 'equalto', 'access') | list }}
          loop_control:
            label: "{{ item.key }} access VLAN {{ item.value.access_vlan }}"
          when: switched_interfaces is defined
          notify: Save IOS configuration

        - name: "VLANs | Configure trunk ports"
          cisco.ios.ios_l2_interfaces:
            config:
              - name: "{{ item.key }}"
                mode: trunk
                trunk:
                  allowed_vlans: "{{ item.value.trunk_vlans }}"
                  native_vlan: "{{ item.value.native_vlan | default(1) }}"
            state: merged
          loop: >-
            {{ switched_interfaces | default({}) | dict2items
               | selectattr('value.mode', 'equalto', 'trunk') | list }}
          loop_control:
            label: "{{ item.key }} trunk VLANs {{ item.value.trunk_vlans }}"
          when: switched_interfaces is defined
          notify: Save IOS configuration

      tags: [vlans, switching, base]

    # ── Section 4: Spanning Tree ──────────────────────────────────
    # No ios_stp resource module exists — ios_config required
    - block:
        - name: "STP | Configure PortFast and BPDU Guard on access ports"
          cisco.ios.ios_config:
            lines:
              - spanning-tree portfast
              - spanning-tree bpduguard enable
            parents: "interface {{ item.key }}"
          loop: >-
            {{ switched_interfaces | default({}) | dict2items
               | selectattr('value.spanning_tree.portfast', 'defined')
               | selectattr('value.spanning_tree.portfast', 'equalto', true) | list }}
          loop_control:
            label: "STP: {{ item.key }} portfast+bpduguard"
          when: switched_interfaces is defined
          notify: Save IOS configuration

        - name: "STP | Enable BPDU Guard globally as default"
          cisco.ios.ios_config:
            lines:
              - spanning-tree portfast bpduguard default
          notify: Save IOS configuration

      tags: [stp, switching]

    # ── Section 5: Static Routes ──────────────────────────────────
    - block:
        - name: "Routing | Configure static routes"
          cisco.ios.ios_static_routes:
            config:
              - address_families:
                  - afi: ipv4
                    routes:
                      - dest: "{{ item.destination }}/{{ item.mask | ansible.utils.ipaddr('prefix') }}"
                        next_hops:
                          - forward_router_address: "{{ item.next_hop }}"
                            distance_metric: "{{ item.admin_distance | default(1) }}"
            state: merged
          loop: "{{ static_routes | default([]) }}"
          loop_control:
            label: "Route {{ item.destination }}/{{ item.mask }} via {{ item.next_hop }}"
          when: static_routes is defined
          notify: Save IOS configuration

      tags: [routing, static_routes]

    # ── Section 6: OSPF ───────────────────────────────────────────
    - block:
        - name: "OSPF | Configure OSPF process and router-id"
          cisco.ios.ios_ospfv2:
            config:
              processes:
                - process_id: "{{ ospf.process_id }}"
                  router_id: "{{ ospf.router_id }}"
                  passive_interfaces:
                    default: false
                    interface:
                      - name: "{{ item.name }}"
                  auto_cost:
                    reference_bandwidth: 10000
            state: merged
          loop: "{{ ospf.areas | map(attribute='interfaces') | flatten
                    | selectattr('passive', 'defined')
                    | selectattr('passive', 'equalto', true) | list }}"
          loop_control:
            label: "Passive: {{ item.name }}"
          when: ospf is defined
          notify: Save IOS configuration

        - name: "OSPF | Assign interfaces to OSPF areas"
          cisco.ios.ios_config:
            lines:
              - "ip ospf {{ ospf.process_id }} area {{ item.1.area_id }}"
              - "{% if item.0.type is defined %}ip ospf network {{ item.0.type }}{% endif %}"
            parents: "interface {{ item.0.name }}"
          # Flatten: loop over (interface, area) pairs
          loop: >-
            {{ ospf.areas
               | subelements('interfaces')
               | map('reverse') | list }}
          loop_control:
            label: "{{ item.1.name }} → OSPF area {{ item.0.area_id }}"
          when: ospf is defined
          notify: Save IOS configuration

      tags: [routing, ospf]

    # ── Section 7: ACLs ───────────────────────────────────────────
    - block:
        - name: "ACL | Configure standard ACLs"
          cisco.ios.ios_acls:
            config:
              - afi: ipv4
                acls:
                  - name: "{{ item.name }}"
                    acl_type: standard
                    aces:
                      - sequence: "{{ ace.sequence }}"
                        grant: "{{ ace.action }}"
                        source:
                          address: "{{ ace.source if ace.source != 'any' else omit }}"
                          any: "{{ true if ace.source == 'any' else omit }}"
                          wildcard_bits: "{{ ace.wildcard | default(omit) }}"
                        log: "{{ ace.log | default(omit) }}"
                      for ace in item.entries
            state: merged
          loop: "{{ acls | default([]) | selectattr('type', 'equalto', 'standard') | list }}"
          loop_control:
            label: "ACL {{ item.name }} (standard)"
          when: acls is defined
          notify: Save IOS configuration

        - name: "ACL | Configure extended ACLs"
          cisco.ios.ios_acls:
            config:
              - afi: ipv4
                acls:
                  - name: "{{ item.name }}"
                    acl_type: extended
                    aces:
                      - sequence: "{{ ace.sequence }}"
                        grant: "{{ ace.action }}"
                        protocol: "{{ ace.protocol }}"
                        source:
                          address: "{{ ace.source if ace.source != 'any' else omit }}"
                          any: "{{ true if ace.source == 'any' else omit }}"
                          wildcard_bits: "{{ ace.source_wildcard | default(omit) }}"
                        destination:
                          address: "{{ ace.destination if ace.destination != 'any' else omit }}"
                          any: "{{ true if ace.destination == 'any' else omit }}"
                      for ace in item.entries
            state: merged
          loop: "{{ acls | default([]) | selectattr('type', 'equalto', 'extended') | list }}"
          loop_control:
            label: "ACL {{ item.name }} (extended)"
          when: acls is defined
          notify: Save IOS configuration

        - name: "ACL | Apply ACL to VTY lines for SSH access control"
          cisco.ios.ios_config:
            lines:
              - "access-class {{ ssh.vty_access_class }} in"
            parents: "line vty 0 15"
          when: ssh is defined and ssh.vty_access_class is defined
          notify: Save IOS configuration

      tags: [acl, security]

    # ── Section 8: NAT (PAT / Overload) ──────────────────────────
    # No ios_nat resource module — ios_config required
    - block:
        - name: "NAT | Create NAT inside source ACL"
          cisco.ios.ios_config:
            lines:
              - "{{ item }}"
          loop:
            - "ip access-list standard {{ nat.acl_name }}"
            - " permit {{ nat.inside_sources[0].network }} {{ nat.inside_sources[0].wildcard }}"
          when: nat is defined and nat.enabled | bool
          notify: Save IOS configuration

        - name: "NAT | Configure NAT overload (PAT) statement"
          cisco.ios.ios_config:
            lines:
              - >-
                ip nat inside source list {{ nat.acl_name }}
                interface {{ interfaces | dict2items
                             | selectattr('value.nat_outside', 'defined')
                             | selectattr('value.nat_outside', 'equalto', true)
                             | map(attribute='key') | first }}
                overload
          when: nat is defined and nat.enabled | bool
          notify: Save IOS configuration

        - name: "NAT | Mark inside interfaces"
          cisco.ios.ios_config:
            lines:
              - ip nat inside
            parents: "interface {{ item.key }}"
          loop: >-
            {{ interfaces | dict2items
               | selectattr('value.nat_inside', 'defined')
               | selectattr('value.nat_inside', 'equalto', true) | list }}
          loop_control:
            label: "NAT inside: {{ item.key }}"
          when: nat is defined and nat.enabled | bool
          notify: Save IOS configuration

        - name: "NAT | Mark outside interfaces"
          cisco.ios.ios_config:
            lines:
              - ip nat outside
            parents: "interface {{ item.key }}"
          loop: >-
            {{ interfaces | dict2items
               | selectattr('value.nat_outside', 'defined')
               | selectattr('value.nat_outside', 'equalto', true) | list }}
          loop_control:
            label: "NAT outside: {{ item.key }}"
          when: nat is defined and nat.enabled | bool
          notify: Save IOS configuration

      tags: [nat, routing]

    # ── Section 9: SSH Hardening ──────────────────────────────────
    # Mix of resource modules and ios_config (no full SSH resource module)
    - block:
        - name: "SSH | Generate RSA key pair"
          cisco.ios.ios_config:
            lines:
              - "crypto key generate rsa modulus {{ ssh.rsa_bits | default(2048) }}"
          when: ssh is defined
          # Note: This is not idempotent — ios_config sends this even if key exists
          # Use changed_when to suppress false positives
          changed_when: false
          # The key exists check would require ios_command + parsing;
          # safe to re-run this on IOS (re-generates, doesn't duplicate)

        - name: "SSH | Set SSH version and parameters"
          cisco.ios.ios_config:
            lines:
              - "ip ssh version {{ ssh.version | default(2) }}"
              - "ip ssh time-out {{ ssh.timeout | default(60) }}"
              - "ip ssh authentication-retries {{ ssh.authentication_retries | default(3) }}"
          when: ssh is defined
          notify: Save IOS configuration

        - name: "SSH | Configure local user accounts"
          cisco.ios.ios_users:
            config:
              - name: "{{ item.username }}"
                privilege: "{{ item.privilege }}"
                configured_password: "{{ item.secret }}"
                password_type: secret
                state: present
            state: merged
          loop: "{{ local_users | default([]) }}"
          loop_control:
            label: "User: {{ item.username }} (priv {{ item.privilege }})"
          no_log: true    # Secrets in loop — suppress task output
          notify: Save IOS configuration

        - name: "SSH | Configure VTY lines for SSH-only access"
          cisco.ios.ios_config:
            lines:
              - transport input ssh
              - transport output none
              - login local
              - exec-timeout 10 0
            parents: line vty 0 15
          notify: Save IOS configuration

        - name: "SSH | Disable Telnet on console"
          cisco.ios.ios_config:
            lines:
              - transport input none
              - transport output none
              - login local
            parents: line con 0
          notify: Save IOS configuration

      tags: [ssh, security, hardening]

    # ── Section 10: NTP ───────────────────────────────────────────
    - block:
        - name: "NTP | Configure NTP servers"
          cisco.ios.ios_ntp_global:
            config:
              servers:
                - server: "{{ item }}"
                  prefer: "{{ true if loop.index == 1 else false }}"
            state: merged
          loop: "{{ ntp_servers }}"
          loop_control:
            label: "NTP: {{ item }}"
          notify: Save IOS configuration

        - name: "NTP | Configure NTP source interface"
          cisco.ios.ios_config:
            lines:
              - "ntp source {{ ntp_source_interface }}"
          when: ntp_source_interface is defined
          notify: Save IOS configuration

      tags: [ntp, services, base]

    # ── Section 11: Syslog ────────────────────────────────────────
    - block:
        - name: "Logging | Configure syslog"
          cisco.ios.ios_logging_global:
            config:
              hosts:
                - hostname: "{{ syslog_server }}"
                  severity: "{{ ios_logging_severity | default('informational') }}"
              buffered:
                size: "{{ ios_logging_buffer_size | default(16384) }}"
                severity: "{{ ios_logging_severity | default('informational') }}"
              console:
                severity: critical
            state: merged
          notify: Save IOS configuration

      tags: [logging, services, base]

    # ── Section 12: SNMP ──────────────────────────────────────────
    - block:
        - name: "SNMP | Configure community and location"
          cisco.ios.ios_config:
            lines:
              - "snmp-server community {{ snmp_community_ro }} RO"
              - "snmp-server location {{ snmp_location | default('Unknown') }}"
              - "snmp-server contact {{ snmp_contact | default('netops@lab.local') }}"
          no_log: true
          notify: Save IOS configuration

      tags: [snmp, services, base]

    # ── Verification (complex topics only) ───────────────────────
    - block:
        - name: "Verify | Check OSPF neighbor state"
          cisco.ios.ios_command:
            commands:
              - show ip ospf neighbor
          register: ospf_neighbors
          changed_when: false
          when: ospf is defined

        - name: "Verify | Assert OSPF neighbors are FULL"
          ansible.builtin.assert:
            that:
              - "'FULL' in ospf_neighbors.stdout[0]"
            fail_msg: "OSPF neighbors not in FULL state on {{ inventory_hostname }}"
            success_msg: "PASS: OSPF neighbors are FULL on {{ inventory_hostname }}"
          when: ospf is defined and ospf_neighbors is not skipped

        - name: "Verify | Display OSPF neighbors"
          ansible.builtin.debug:
            msg: "{{ ospf_neighbors.stdout_lines[0] }}"
          when: ospf is defined and ospf_neighbors is not skipped

        - name: "Verify | Check ACL existence"
          cisco.ios.ios_command:
            commands:
              - "show ip access-lists"
          register: acl_output
          changed_when: false
          when: acls is defined

        - name: "Verify | Assert all ACLs are present"
          ansible.builtin.assert:
            that:
              - "item.name in acl_output.stdout[0]"
            fail_msg: "ACL {{ item.name }} not found on {{ inventory_hostname }}"
            success_msg: "PASS: ACL {{ item.name }} present"
          loop: "{{ acls | default([]) }}"
          loop_control:
            label: "ACL: {{ item.name }}"
          when: acls is defined and acl_output is not skipped

        - name: "Verify | Check NAT translations"
          cisco.ios.ios_command:
            commands:
              - show ip nat translations
              - show ip nat statistics
          register: nat_output
          changed_when: false
          when: nat is defined and nat.enabled | bool

        - name: "Verify | Display NAT statistics"
          ansible.builtin.debug:
            msg: "{{ nat_output.stdout_lines[1] }}"
          when: nat is defined and nat.enabled | bool and nat_output is not skipped

      tags: [validate, verify]

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands:
          - write memory
      listen: Save IOS configuration
```

---

## 17.3 — The Companion Jinja2 Template

The master playbook above uses resource modules and `ios_config` to push configuration. The companion template provides the same configuration as a rendered text file — useful for review before pushing, for generating config snippets for manual application, and for the `ios_config src:` approach when pushing a complete config to a fresh device.

```bash
cat > ~/projects/ansible-network/templates/ios/wan_router.j2 << 'TEMPLATE'
{# =============================================================
   templates/ios/wan_router.j2
   Complete IOS-XE WAN router configuration template.
   Covers all 12 CCNA-level configuration domains.
   Generated by Ansible — do not edit on device.
   ============================================================= #}

{# ── Section 1: Identity ────────────────────────────────────── #}
hostname {{ device_hostname }}
!
ip domain-name {{ domain_name }}
!
banner login ^
{{ ios_banner_text | default('Authorized access only.') }}
^
!

{# ── Section 2: Loopback Interfaces ────────────────────────── #}
{% for iface_name, iface in loopbacks.items() %}
interface {{ iface_name }}
 description {{ iface.description }}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
 no shutdown
!
{% endfor %}

{# ── Section 3: Physical Interfaces ────────────────────────── #}
{% for iface_name, iface in interfaces.items() %}
interface {{ iface_name }}
 description {{ iface.description | default('') }}
{% if iface.ip is defined %}
 ip address {{ iface.ip }} {{ iface.prefix | ansible.utils.ipaddr('netmask') }}
{% endif %}
{% if iface.nat_inside | default(false) %}
 ip nat inside
{% endif %}
{% if iface.nat_outside | default(false) %}
 ip nat outside
{% endif %}
{% if iface.shutdown | default(false) %}
 shutdown
{% else %}
 no shutdown
{% endif %}
!
{% endfor %}

{# ── Section 4: VLANs ────────────────────────────────────────── #}
{% if vlans is defined %}
{% for vlan in vlans %}
vlan {{ vlan.id }}
 name {{ vlan.name }}
!
{% endfor %}
{% endif %}

{# ── Section 5: Switched Interfaces ────────────────────────── #}
{% if switched_interfaces is defined %}
{% for iface_name, iface in switched_interfaces.items() %}
interface {{ iface_name }}
{% if iface.mode == 'access' %}
 switchport mode access
 switchport access vlan {{ iface.access_vlan }}
{% if iface.spanning_tree.portfast | default(false) %}
 spanning-tree portfast
{% endif %}
{% if iface.spanning_tree.bpduguard | default(false) %}
 spanning-tree bpduguard enable
{% endif %}
{% elif iface.mode == 'trunk' %}
 switchport mode trunk
 switchport trunk allowed vlan {{ iface.trunk_vlans }}
 switchport trunk native vlan {{ iface.native_vlan | default(1) }}
{% endif %}
!
{% endfor %}
{% endif %}

{# ── Section 6: Static Routes ────────────────────────────────── #}
{% if static_routes is defined %}
{% for route in static_routes %}
ip route {{ route.destination }} {{ route.mask }} {{ route.next_hop }}{% if route.admin_distance is defined %} {{ route.admin_distance }}{% endif %}
{% endfor %}
!
{% endif %}

{# ── Section 7: OSPF ─────────────────────────────────────────── #}
{% if ospf is defined %}
router ospf {{ ospf.process_id }}
 router-id {{ ospf.router_id }}
 auto-cost reference-bandwidth 10000
{# Collect passive interfaces across all areas #}
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
{% if iface.passive | default(false) %}
 passive-interface {{ iface.name }}
{% endif %}
{% endfor %}
{% endfor %}
!
{# Assign interfaces to OSPF areas #}
{% for area in ospf.areas %}
{% for iface in area.interfaces %}
interface {{ iface.name }}
 ip ospf {{ ospf.process_id }} area {{ area.area_id }}
{% if iface.type is defined %}
 ip ospf network {{ iface.type }}
{% endif %}
!
{% endfor %}
{% endfor %}
{% endif %}

{# ── Section 8: ACLs ─────────────────────────────────────────── #}
{% if acls is defined %}
{% for acl in acls %}
{% if acl.type == 'standard' %}
ip access-list standard {{ acl.name }}
{% for remark in acl.remarks | default([]) %}
 remark {{ remark }}
{% endfor %}
{% for entry in acl.entries %}
 {{ entry.sequence }} {{ entry.action }} {{ entry.source }}{% if entry.source != 'any' %} {{ entry.wildcard | default('') }}{% endif %}{% if entry.log | default(false) %} log{% endif %}

{% endfor %}
!
{% elif acl.type == 'extended' %}
ip access-list extended {{ acl.name }}
{% for remark in acl.remarks | default([]) %}
 remark {{ remark }}
{% endfor %}
{% for entry in acl.entries %}
 {{ entry.sequence }} {{ entry.action }} {{ entry.protocol }} {{ entry.source }}{% if entry.source != 'any' %} {{ entry.source_wildcard | default('0.0.0.0') }}{% endif %} {{ entry.destination }}{% if entry.destination != 'any' %} {{ entry.dest_wildcard | default('0.0.0.0') }}{% endif %}

{% endfor %}
!
{% endif %}
{% endfor %}
{% endif %}

{# ── Section 9: NAT ──────────────────────────────────────────── #}
{% if nat is defined and nat.enabled %}
ip access-list standard {{ nat.acl_name }}
{% for source in nat.inside_sources %}
 permit {{ source.network }} {{ source.wildcard }}
{% endfor %}
!
{# Find the nat_outside interface name #}
{% set nat_outside_iface = interfaces | dict2items
   | selectattr('value.nat_outside', 'defined')
   | selectattr('value.nat_outside', 'equalto', true)
   | map(attribute='key') | first %}
ip nat inside source list {{ nat.acl_name }} interface {{ nat_outside_iface }} overload
!
{% endif %}

{# ── Section 10: SSH ─────────────────────────────────────────── #}
{% if ssh is defined %}
ip ssh version {{ ssh.version | default(2) }}
ip ssh time-out {{ ssh.timeout | default(60) }}
ip ssh authentication-retries {{ ssh.authentication_retries | default(3) }}
!
{% for user in local_users | default([]) %}
username {{ user.username }} privilege {{ user.privilege }} secret <vaulted>
{% endfor %}
!
line vty 0 15
 transport input ssh
 transport output none
 login local
{% if ssh.vty_access_class is defined %}
 access-class {{ ssh.vty_access_class }} in
{% endif %}
 exec-timeout 10 0
!
line con 0
 transport input none
 login local
!
{% endif %}

{# ── Section 11: NTP ─────────────────────────────────────────── #}
{% for ntp in ntp_servers %}
ntp server {{ ntp }}{% if loop.first %} prefer{% endif %}

{% endfor %}
{% if ntp_source_interface is defined %}
ntp source {{ ntp_source_interface }}
{% endif %}
!

{# ── Section 12: Services ────────────────────────────────────── #}
service password-encryption
no ip source-route
no ip proxy-arp
logging host {{ syslog_server }}
logging buffered {{ ios_logging_buffer_size | default(16384) }} {{ ios_logging_severity | default('informational') }}
snmp-server community <vaulted> RO
snmp-server location {{ snmp_location | default('Unknown') }}
snmp-server contact {{ snmp_contact | default('netops@lab.local') }}
!
{# End of template #}
TEMPLATE
```

### Rendering the Template for Review

```bash
# Render to a file for review — no changes pushed
ansible wan-r1 -m ansible.builtin.template \
    -a "src=templates/ios/wan_router.j2 dest=/tmp/wan-r1-rendered.txt" \
    --connection local

# Or via playbook with render-only tag
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
    --tags render --limit wan-r1

# Review the output
cat /tmp/wan-r1-rendered.txt
```

---

## 17.4 — Module Reference: The `cisco.ios` Collection

Quick reference for every module used in this part:

### `ios_facts` — Gather Device Information

```yaml
cisco.ios.ios_facts:
  gather_subset:
    - default      # hostname, version, model, serial, image
    - hardware     # memory, flash storage
    - interfaces   # interface names, IPs, MAC addresses, state
    - config       # full running config (expensive — avoid unless needed)
    - all          # everything (very expensive — never in production loops)
```

Key variables populated: `ansible_net_hostname`, `ansible_net_version`, `ansible_net_model`, `ansible_net_serialnum`, `ansible_net_interfaces`, `ansible_net_all_ipv4_addresses`.

### `ios_command` — Run Operational Commands

```yaml
cisco.ios.ios_command:
  commands:
    - show version
    - show ip interface brief
  # Optional: wait for expected output before returning
  wait_for:
    - result[0] contains Established
  retries: 10      # How many times to retry
  interval: 5      # Seconds between retries
```

Always use `changed_when: false` on `ios_command` — show commands never make changes.

### `ios_config` — Push Raw Configuration Lines

```yaml
cisco.ios.ios_config:
  lines:
    - "ntp server 8.8.8.8"
    - "ntp server 8.8.4.4"
  parents: null        # Global config (no parent)
  # OR
  parents: "interface GigabitEthernet1"  # Interface config
  # OR
  parents:
    - "router ospf 1"    # Nested parents
    - "area 0"
  replace: line        # Default: merge changes (line = add missing lines)
  save_when: changed   # Write memory only when something changed
```

### `ios_interfaces` — Interface State and Description

```yaml
cisco.ios.ios_interfaces:
  config:
    - name: GigabitEthernet1
      description: "WAN | To FW-01"
      enabled: true        # true = no shutdown, false = shutdown
      mtu: 1500
      speed: auto
      duplex: auto
  state: merged            # merged | replaced | overridden | deleted
```

### `ios_l2_interfaces` — Layer 2 Switchport Configuration

```yaml
cisco.ios.ios_l2_interfaces:
  config:
    - name: GigabitEthernet4
      mode: access
      access:
        vlan: 20
    - name: GigabitEthernet5
      mode: trunk
      trunk:
        allowed_vlans: "10,20,30"
        native_vlan: 99
        encapsulation: dot1q
  state: merged
```

### `ios_l3_interfaces` — Layer 3 IP Addressing

```yaml
cisco.ios.ios_l3_interfaces:
  config:
    - name: GigabitEthernet1
      ipv4:
        - address: 10.10.10.1/30
          secondary: false    # Primary address
        - address: 10.10.10.5/30
          secondary: true     # Secondary address
      ipv6:
        - address: 2001:db8::1/64
  state: merged
```

### `ios_vlans` — VLAN Database

```yaml
cisco.ios.ios_vlans:
  config:
    - vlan_id: 10
      name: MGMT
      state: active
    - vlan_id: 20
      name: SERVERS
      state: active
  state: merged     # merged = add VLANs | replaced = replace all | deleted = remove
```

### `ios_acls` — Access Control Lists

```yaml
cisco.ios.ios_acls:
  config:
    - afi: ipv4
      acls:
        - name: MGMT_ACCESS
          acl_type: standard
          aces:
            - sequence: 10
              grant: permit
              source:
                address: 192.168.1.0
                wildcard_bits: 0.0.0.255
  state: merged
```

### `ios_static_routes` — Static Routing

```yaml
cisco.ios.ios_static_routes:
  config:
    - address_families:
        - afi: ipv4
          routes:
            - dest: 0.0.0.0/0
              next_hops:
                - forward_router_address: 10.10.10.2
                  distance_metric: 254
                  description: "Default route — floating"
  state: merged
```

### `ios_ospfv2` — OSPFv2 Process Configuration

```yaml
cisco.ios.ios_ospfv2:
  config:
    processes:
      - process_id: 1
        router_id: 10.255.0.1
        auto_cost:
          reference_bandwidth: 10000
        passive_interfaces:
          default: false
          interface:
            - name: Loopback0
        areas:
          - area_id: 0
            authentication:
              message_digest: false
  state: merged
```

### `ios_bgp_global` — BGP Global Configuration

```yaml
cisco.ios.ios_bgp_global:
  config:
    as_number: "65100"
    bgp:
      router_id:
        address: 10.255.0.1
      log_neighbor_changes: true
    neighbor:
      - neighbor_address: 10.10.10.2
        remote_as: 65200
        description: "eBGP to FW-01"
  state: merged
```

### `ios_ntp_global` — NTP Configuration

```yaml
cisco.ios.ios_ntp_global:
  config:
    servers:
      - server: 216.239.35.0
        prefer: true
      - server: 216.239.35.4
  state: merged
```

### `ios_logging_global` — Logging Configuration

```yaml
cisco.ios.ios_logging_global:
  config:
    hosts:
      - hostname: 172.16.0.100
        severity: informational
    buffered:
      size: 16384
      severity: informational
    console:
      severity: critical
  state: merged
```

### `ios_users` — Local User Accounts

```yaml
cisco.ios.ios_users:
  config:
    - name: ansible
      privilege: 15
      configured_password: "{{ vault_ansible_enable_secret }}"
      password_type: secret    # Stores as type 9 secret (scrypt)
      state: present
  state: merged
```

### `ios_banner` — Login/MOTD Banners

```yaml
cisco.ios.ios_banner:
  banner: login    # login | motd | exec | incoming
  text: |
    ******************************************
    * Authorized access only                 *
    ******************************************
  state: present
```

---

## 17.5 — Understanding `state:` in Resource Modules

The `state:` parameter controls how the module reconciles the desired state with the current device configuration. Getting this wrong is the most common mistake with resource modules.

```
merged     → Add the specified config. Leave existing config untouched.
            Example: I specify VLAN 10. VLANs 20, 30 already exist.
            Result: VLAN 10 added. VLANs 20, 30 remain.

replaced   → Replace the config for the specified resources only.
            Example: I specify interface Gi1 with one IP.
            Result: Gi1 gets exactly that IP. All other Gi1 config replaced.
            Gi2, Gi3 are untouched.

overridden → Replace the ENTIRE resource config globally.
            Example: I specify only VLAN 10 with overridden.
            Result: VLAN 10 configured. ALL OTHER VLANs deleted.
            Use with extreme caution — designed for enforcement, not incremental.

deleted    → Remove the specified config.
            Example: I specify VLAN 10 with deleted.
            Result: VLAN 10 removed. All other VLANs untouched.

gathered   → Read-only: gather current config into ansible_network_resources.
            No changes made.

rendered   → Convert provided config to CLI commands without connecting.
            Useful for dry-run preview.

parsed     → Parse provided text (e.g., backup file) into structured data.
            No connection needed.
```

### The State That Bites People: `overridden`

```yaml
# ❌ Dangerous misuse — this deletes ALL VLANs except VLAN 10
cisco.ios.ios_vlans:
  config:
    - vlan_id: 10
      name: MGMT
  state: overridden    # ← Removes every VLAN not in this list

# ✅ Safe — adds VLAN 10, doesn't touch existing VLANs
cisco.ios.ios_vlans:
  config:
    - vlan_id: 10
      name: MGMT
  state: merged
```

`overridden` is the right tool for compliance enforcement — when I want to ensure a device has exactly and only the configuration defined in the data model. It's never the right default. I use `merged` for incremental adds and `replaced` when I need to replace a specific resource's config while leaving others untouched.

---

## 17.6 — Common Gotchas for IOS Automation

### ### 🪲 Gotcha — `ios_vlans` requires the device to be in VTP transparent or VTP off mode

```yaml
# If VTP mode is server or client, ios_vlans may fail or changes may be
# overridden by the VTP server. Always check VTP mode first:
- name: "Pre-flight | Check VTP mode"
  cisco.ios.ios_command:
    commands: [show vtp status]
  register: vtp_status
  changed_when: false

- name: "Pre-flight | Set VTP to transparent mode"
  cisco.ios.ios_config:
    lines:
      - vtp mode transparent
  when: "'VTP Operating Mode.*Server' in vtp_status.stdout[0]"
```

### ### 🪲 Gotcha — `ios_l2_interfaces` requires interface to be in switchport mode first

```yaml
# ❌ Fails if interface is in routed (no switchport) mode
cisco.ios.ios_l2_interfaces:
  config:
    - name: GigabitEthernet4
      mode: access

# ✅ Set switchport mode first with ios_config
- name: "Switching | Enable switchport on interface"
  cisco.ios.ios_config:
    lines:
      - switchport
    parents: "interface GigabitEthernet4"
  notify: Save IOS configuration

- name: "Switching | Configure access VLAN"
  cisco.ios.ios_l2_interfaces:
    config:
      - name: GigabitEthernet4
        mode: access
        access:
          vlan: 20
    state: merged
```

### ### 🪲 Gotcha — `ios_ospfv2` `passive_interfaces` syntax is counterintuitive

The `passive_interfaces` key under `ios_ospfv2` expects a specific structure. Getting it wrong causes the task to succeed but not configure passive interfaces:

```yaml
# ✅ Correct structure for passive interfaces
processes:
  - process_id: 1
    passive_interfaces:
      default: false          # Don't make ALL interfaces passive
      interface:
        - name: Loopback0     # Make only these interfaces passive
        - name: GigabitEthernet2
```

Alternatively, use `ios_config` for passive interface configuration — it's more predictable:

```yaml
- name: "OSPF | Set passive interfaces"
  cisco.ios.ios_config:
    lines:
      - "passive-interface {{ item }}"
    parents: "router ospf {{ ospf.process_id }}"
  loop: >-
    {{ ospf.areas | map(attribute='interfaces') | flatten
       | selectattr('passive', 'defined')
       | selectattr('passive', 'equalto', true)
       | map(attribute='name') | list }}
```

### ### 🪲 Gotcha — `ios_acls` and `any` source/destination handling

The `ios_acls` module requires explicit `any: true` for wildcard matches rather than the string `"any"`. Getting this wrong generates a task that reports success but doesn't push the correct ACE:

```yaml
# ✅ Correct — use any: true key
aces:
  - sequence: 10
    grant: permit
    source:
      any: true              # ← Must be the key, not a string value

# ❌ Wrong — string 'any' in address field doesn't work
aces:
  - sequence: 10
    grant: permit
    source:
      address: any           # ← This doesn't work
```

---


The complete CCNA-level IOS automation system is in place — one data model, one playbook, one template, covering every configuration domain a network engineer works with daily. Part 18 applies the same treatment to Cisco NX-OS — covering the data center specific features: VPC, VXLAN, fabric routing, and the NX-OS modules that differ from their IOS equivalents.


