---
draft: true
title: '21 - PanOS'
weight: 21
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 21: Palo Alto PAN-OS Network Automation

> *Every platform covered so far — IOS, NX-OS, Junos — is fundamentally a routing/switching OS that you configure via CLI or NETCONF. PAN-OS is different in kind, not just degree. It's a security operating system built around an XML API, where configuration is expressed as a tree of named objects that reference each other. You don't write ACL lines — you create address objects, group them, define zones, write application-based security rules that reference those objects, and commit. Nothing takes effect until commit, and commit is all-or-nothing. Understanding the object hierarchy is what makes PAN-OS automation make sense.*

---

## 21.1 — The PAN-OS Model: Objects, References, and Commit

### API-First Architecture

PAN-OS has no meaningful CLI-driven configuration model. The CLI exists, but the underlying system is XML — every configuration element is an XML node in a hierarchy rooted at `/config`. The REST API and the XML API both read and write this tree directly. The `paloaltonetworks.panos` collection talks to the XML API, not SSH CLI.

```
Traditional network OS:        PAN-OS:
  SSH → CLI → parser →           HTTPS → XML API → config tree
  running config                   │
                                   ├── /config/devices/entry/vsys/entry/
                                   │     ├── address/             ← address objects
                                   │     ├── address-group/       ← address groups
                                   │     ├── service/             ← service objects
                                   │     ├── security/rules/      ← security policy
                                   │     └── nat/rules/           ← NAT policy
                                   └── /config/devices/entry/network/
                                         ├── interface/           ← interfaces
                                         └── zone/                ← security zones
```

### The Object Hierarchy

PAN-OS configuration is object-based — every element that can be named and reused is an object, and rules reference objects by name rather than by value. This is the hierarchy:

```
Level 1 — Zones (trust, untrust, dmz...)
  └── Level 2 — Interfaces (assigned to zones)
        └── Level 3 — Address Objects (named IP addresses/ranges/FQDNs)
              └── Level 4 — Address Groups (named collections of address objects)
                    └── Level 5 — Service Objects (named port/protocol combos)
                          └── Level 6 — Security Rules (reference all of the above)
                                └── Level 7 — NAT Rules (reference zones and addresses)
```

A security rule doesn't contain `source: 192.168.1.0/24`. It contains `source: INTERNAL_SERVERS` — where `INTERNAL_SERVERS` is an address group that contains address objects `SERVER-01` (192.168.1.10) and `SERVER-02` (192.168.1.20). When a server IP changes, I update one address object and the change propagates to every rule that references the group. This is the PAN-OS way.

### The Commit Model

Like Junos, nothing takes effect until commit. Unlike Junos, there's no NETCONF candidate — PAN-OS has a running config (active) and a candidate config (uncommitted changes). Every API call that modifies configuration writes to the candidate. `commit` pushes candidate to running.

```
panos_address_object task → Writes to candidate config
panos_security_rule task  → Writes to candidate config
panos_nat_rule task       → Writes to candidate config
                                    │
                            panos_commit_firewall
                                    │
                         All changes become active simultaneously
```

The critical implication for Ansible: **object creation tasks must complete before rule creation tasks**, because rules reference objects by name. If the address group doesn't exist when the security rule is created, the rule creation fails. Dependency ordering is mandatory.

---

## 21.2 — Collection Setup

```bash
# Install the Palo Alto collection
ansible-galaxy collection install paloaltonetworks.panos

# Install the Python PAN-OS library
pip install pan-python pan-os-python --break-system-packages

# Verify
python3 -c "import panos; print('pan-os-python OK')"
```

### Connection Model

The `panos` collection uses HTTPS to the management interface — not SSH, not NETCONF:

```yaml
# group_vars/panos_devices.yml
ansible_network_os: paloaltonetworks.panos.panos
ansible_connection: local    # ← panos modules run locally and make HTTPS calls
                              # Not network_cli, not netconf

# Connection credentials — set in host_vars or vault
# ansible_host: the firewall management IP
# ansible_user: admin (or API-only account)
# ansible_password: vaulted
```

The `ansible_connection: local` deserves explanation. PAN-OS modules don't use Ansible's standard connection plugins at all. Each module makes its own HTTPS API calls to the firewall using the `pan-os-python` library. Ansible runs the module on the control node (`local`), and the module itself handles the HTTPS connection to the firewall. This means PAN-OS tasks in the play recap show as running on `localhost`, not on the firewall.

### API Key Authentication (Preferred)

```bash
# Generate an API key from the firewall (one-time)
curl -k -X GET \
  "https://172.16.0.51/api/?type=keygen&user=ansible&password=YourPassword"
# Returns: <response status="success"><result><key>LUFRPT14...</key></result></response>

# Store the key in vault
ansible-vault encrypt_string --vault-id lab@.vault/lab.txt \
  'LUFRPT14...' --name 'vault_panos_api_key'
```

```yaml
# host_vars/panos-fw01.yml
api_key: "{{ vault_panos_api_key }}"
# Use api_key instead of username/password for all panos module calls
```

---

## 21.3 — The PAN-OS Data Model

Built around the object hierarchy: zones and interfaces first, then objects, then rules.

```bash
cat > ~/projects/ansible-network/inventory/host_vars/panos-fw01.yml << 'EOF'
---
# =============================================================
# panos-fw01 — Palo Alto PAN-OS Firewall (standalone)
# Lab image: pan-os/panos-vm in Containerlab
# =============================================================

device_hostname: panos-fw01
ansible_host: 172.16.0.51
api_key: "{{ vault_panos_api_key }}"

# ── Zones ─────────────────────────────────────────────────────────
# Zones are the foundation — every interface belongs to a zone
# and every security rule references source/destination zones
zones:
  - name: untrust
    mode: layer3
    description: "Internet-facing zone"
    enable_userid: false
  - name: trust
    mode: layer3
    description: "Internal network zone"
    enable_userid: true
  - name: dmz
    mode: layer3
    description: "DMZ — public-facing servers"
    enable_userid: false
  - name: mgmt-zone
    mode: layer3
    description: "Out-of-band management zone"
    enable_userid: false

# ── Interfaces ────────────────────────────────────────────────────
interfaces:
  - name: ethernet1/1
    mode: layer3
    zone: untrust
    description: "WAN | To upstream router"
    ip: 10.10.10.2
    prefix: 30
    management_profile: ~   # No management access from untrust
  - name: ethernet1/2
    mode: layer3
    zone: trust
    description: "LAN | Internal network"
    ip: 192.168.1.1
    prefix: 24
    management_profile: ALLOW_MGMT
  - name: ethernet1/3
    mode: layer3
    zone: dmz
    description: "DMZ | Public-facing servers"
    ip: 172.16.10.1
    prefix: 24
    management_profile: ~

# ── Address Objects ───────────────────────────────────────────────
# Granular named objects — represent individual IPs, subnets, or FQDNs
address_objects:
  - name: INTERNAL-NET
    type: ip-netmask
    value: 192.168.1.0/24
    description: "Internal LAN network"
    tags: [internal, production]

  - name: SERVER-WEB-01
    type: ip-netmask
    value: 172.16.10.10/32
    description: "Primary web server"
    tags: [dmz, web, production]

  - name: SERVER-WEB-02
    type: ip-netmask
    value: 172.16.10.11/32
    description: "Secondary web server"
    tags: [dmz, web, production]

  - name: SERVER-APP-01
    type: ip-netmask
    value: 172.16.10.20/32
    description: "Application server"
    tags: [dmz, app, production]

  - name: SERVER-DB-01
    type: ip-netmask
    value: 172.16.10.30/32
    description: "Database server"
    tags: [dmz, db, production]

  - name: DNS-GOOGLE-PRIMARY
    type: ip-netmask
    value: 8.8.8.8/32
    description: "Google Public DNS primary"
    tags: [dns, internet]

  - name: DNS-GOOGLE-SECONDARY
    type: ip-netmask
    value: 8.8.4.4/32
    description: "Google Public DNS secondary"
    tags: [dns, internet]

  - name: MGMT-WORKSTATION
    type: ip-netmask
    value: 192.168.1.100/32
    description: "Network admin workstation"
    tags: [internal, management]

  - name: SYSLOG-SERVER
    type: ip-netmask
    value: 172.16.0.100/32
    description: "Centralized syslog server"
    tags: [internal, management]

# ── Address Groups ────────────────────────────────────────────────
# Groups aggregate address objects — rules reference groups, not individual objects
# When a new web server is added, add it to WEB-SERVERS group — all rules update automatically
address_groups:
  - name: WEB-SERVERS
    description: "All public-facing web servers"
    members:
      - SERVER-WEB-01
      - SERVER-WEB-02
    tags: [dmz, web]

  - name: APP-SERVERS
    description: "Application tier servers"
    members:
      - SERVER-APP-01
    tags: [dmz, app]

  - name: DB-SERVERS
    description: "Database tier servers"
    members:
      - SERVER-DB-01
    tags: [dmz, db]

  - name: ALL-DMZ-SERVERS
    description: "All servers in DMZ"
    members:
      - SERVER-WEB-01
      - SERVER-WEB-02
      - SERVER-APP-01
      - SERVER-DB-01
    tags: [dmz]

  - name: DNS-SERVERS
    description: "Permitted DNS resolvers"
    members:
      - DNS-GOOGLE-PRIMARY
      - DNS-GOOGLE-SECONDARY
    tags: [dns]

  - name: MGMT-HOSTS
    description: "Management workstations and servers"
    members:
      - MGMT-WORKSTATION
      - SYSLOG-SERVER
    tags: [management]

# ── Service Objects ───────────────────────────────────────────────
# Named port/protocol objects — reused across rules
service_objects:
  - name: SVC-HTTP
    protocol: tcp
    destination_port: 80
    description: "HTTP"
  - name: SVC-HTTPS
    protocol: tcp
    destination_port: 443
    description: "HTTPS"
  - name: SVC-SSH
    protocol: tcp
    destination_port: 22
    description: "SSH management"
  - name: SVC-NETCONF
    protocol: tcp
    destination_port: 830
    description: "NETCONF"
  - name: SVC-DNS
    protocol: udp
    destination_port: 53
    description: "DNS"
  - name: SVC-MYSQL
    protocol: tcp
    destination_port: 3306
    description: "MySQL database"
  - name: SVC-SYSLOG
    protocol: udp
    destination_port: 514
    description: "Syslog"

# ── Service Groups ────────────────────────────────────────────────
service_groups:
  - name: SVC-WEB
    description: "HTTP and HTTPS"
    members:
      - SVC-HTTP
      - SVC-HTTPS

# ── Security Rules ────────────────────────────────────────────────
# Rules are evaluated top-to-bottom — order matters
# Each rule references zones, address objects/groups, and applications
security_rules:
  - name: ALLOW-INTERNET-TO-WEB
    description: "Permit inbound HTTP/HTTPS to DMZ web servers"
    source_zone: [untrust]
    destination_zone: [dmz]
    source_address: [any]
    destination_address: [WEB-SERVERS]
    application: [web-browsing, ssl]
    service: [application-default]
    action: allow
    log_start: false
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [internet-inbound, production]

  - name: ALLOW-WEB-TO-APP
    description: "Permit web tier to app tier communication"
    source_zone: [dmz]
    destination_zone: [dmz]
    source_address: [WEB-SERVERS]
    destination_address: [APP-SERVERS]
    application: [any]
    service: [SVC-HTTPS]
    action: allow
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [dmz-internal, production]

  - name: ALLOW-APP-TO-DB
    description: "Permit app tier to database tier"
    source_zone: [dmz]
    destination_zone: [dmz]
    source_address: [APP-SERVERS]
    destination_address: [DB-SERVERS]
    application: [mysql]
    service: [application-default]
    action: allow
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [dmz-internal, production]

  - name: ALLOW-INTERNAL-OUTBOUND
    description: "Permit internal users to internet (web)"
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [any]
    application: [web-browsing, ssl, dns]
    service: [application-default]
    action: allow
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [internal-outbound, production]

  - name: ALLOW-INTERNAL-DNS
    description: "Permit internal DNS resolution"
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [DNS-SERVERS]
    application: [dns]
    service: [application-default]
    action: allow
    log_end: false
    tags: [internal-outbound, dns]

  - name: ALLOW-MGMT-SSH
    description: "Permit admin SSH to all zones from management hosts"
    source_zone: [trust]
    destination_zone: [dmz, untrust, mgmt-zone]
    source_address: [MGMT-HOSTS]
    destination_address: [any]
    application: [ssh]
    service: [application-default]
    action: allow
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [management, production]

  - name: ALLOW-SYSLOG-OUTBOUND
    description: "Permit syslog forwarding from all zones"
    source_zone: [trust, dmz]
    destination_zone: [trust]
    source_address: [ALL-DMZ-SERVERS]
    destination_address: [SYSLOG-SERVER]
    application: [syslog]
    service: [application-default]
    action: allow
    log_end: false
    tags: [management, logging]

  - name: DENY-ALL
    description: "Default deny — log all blocked traffic"
    source_zone: [any]
    destination_zone: [any]
    source_address: [any]
    destination_address: [any]
    application: [any]
    service: [any]
    action: deny
    log_start: false
    log_end: true
    log_forwarding: SYSLOG-FORWARDING
    tags: [default-deny]

# ── NAT Rules ─────────────────────────────────────────────────────
nat_rules:
  - name: INTERNET-OUTBOUND-PAT
    description: "Source NAT — internal users to internet (PAT)"
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [any]
    nat_type: ipv4
    source_translation:
      type: dynamic-ip-and-port
      interface_address:
        interface: ethernet1/1
    tags: [nat, outbound]

  - name: DNAT-HTTP-TO-WEB
    description: "Destination NAT — inbound HTTP to web server"
    source_zone: [untrust]
    destination_zone: [untrust]    # Pre-NAT zone — always untrust for inbound DNAT
    source_address: [any]
    destination_address: [10.10.10.2]   # Firewall WAN IP (pre-NAT)
    service: SVC-HTTP
    destination_translation:
      translated_address: SERVER-WEB-01
      translated_port: 80
    tags: [nat, inbound, web]

  - name: DNAT-HTTPS-TO-WEB
    description: "Destination NAT — inbound HTTPS to web server"
    source_zone: [untrust]
    destination_zone: [untrust]
    source_address: [any]
    destination_address: [10.10.10.2]
    service: SVC-HTTPS
    destination_translation:
      translated_address: SERVER-WEB-01
      translated_port: 443
    tags: [nat, inbound, web]

# ── Log Forwarding Profile ────────────────────────────────────────
log_forwarding_profiles:
  - name: SYSLOG-FORWARDING
    description: "Forward traffic logs to syslog server"
    match_list:
      - name: ALL-TRAFFIC
        log_type: traffic
        filter: All Logs
        syslog_destinations:
          - name: SYSLOG-SERVER-PROFILE
            syslog_server: 172.16.0.100
            facility: LOG_USER
            format: BSD
EOF
```

---

## 21.4 — The PAN-OS Master Playbook

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_panos.yml
```

```yaml
---
# =============================================================
# deploy_panos.yml — PAN-OS complete configuration
# Object hierarchy: zones → interfaces → objects → groups → rules
# Two commit approaches:
#   Simple:     handler fires after all changes
#   Production: explicit two-phase commit with verification
#
# Common invocations:
#   Full run:       ansible-playbook deploy_panos.yml
#   Objects only:   ansible-playbook deploy_panos.yml --tags objects
#   Rules only:     ansible-playbook deploy_panos.yml --tags rules
#   Commit only:    ansible-playbook deploy_panos.yml --tags commit
#   Diff (no push): ansible-playbook deploy_panos.yml --check
# =============================================================

- name: "PAN-OS | Deploy firewall configuration"
  hosts: panos_devices
  gather_facts: false
  connection: local    # panos modules make their own HTTPS API calls

  vars:
    panos_provider:
      ip_address: "{{ ansible_host }}"
      api_key: "{{ api_key }}"
      # Alternative: username/password
      # username: "{{ ansible_user }}"
      # password: "{{ ansible_password }}"

  tasks:

    # ── Pre-flight ────────────────────────────────────────────────
    - name: "Pre-flight | Gather PAN-OS facts"
      paloaltonetworks.panos.panos_facts:
        provider: "{{ panos_provider }}"
      register: panos_device_facts
      tags: always

    - name: "Pre-flight | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device:   {{ inventory_hostname }}"
          - "Hostname: {{ panos_device_facts.ansible_facts.ansible_net_hostname }}"
          - "Version:  {{ panos_device_facts.ansible_facts.ansible_net_version }}"
          - "Model:    {{ panos_device_facts.ansible_facts.ansible_net_model }}"
        verbosity: 1
      tags: always

    # ── Section 1: Zones ──────────────────────────────────────────
    # Zones must exist before interfaces can be assigned to them
    # PAN-OS difference: zones are first-class config objects, not interface attributes
    - block:
        - name: "Zones | Create security zones"
          paloaltonetworks.panos.panos_zone:
            provider: "{{ panos_provider }}"
            zone: "{{ item.name }}"
            mode: "{{ item.mode }}"
            enable_userid: "{{ item.enable_userid | default(false) }}"
            state: present
          loop: "{{ zones | default([]) }}"
          loop_control:
            label: "Zone: {{ item.name }} ({{ item.mode }})"
          notify: Commit PAN-OS configuration

      tags: [zones, base]

    # ── Section 2: Interfaces ─────────────────────────────────────
    # Interfaces assigned to zones after zones exist
    - block:
        - name: "Interfaces | Configure L3 interfaces"
          paloaltonetworks.panos.panos_interface:
            provider: "{{ panos_provider }}"
            if_name: "{{ item.name }}"
            mode: "{{ item.mode }}"
            ip: ["{{ item.ip }}/{{ item.prefix }}"]
            zone_name: "{{ item.zone }}"
            comment: "{{ item.description | default('') }}"
            state: present
          loop: "{{ interfaces | default([]) }}"
          loop_control:
            label: "{{ item.name }} → zone {{ item.zone }}: {{ item.ip }}/{{ item.prefix }}"
          notify: Commit PAN-OS configuration

      tags: [interfaces, base]

    # ── Section 3: Address Objects ────────────────────────────────
    # Must exist before address groups or rules reference them
    - block:
        - name: "Objects | Create address objects"
          paloaltonetworks.panos.panos_address_object:
            provider: "{{ panos_provider }}"
            name: "{{ item.name }}"
            value: "{{ item.value }}"
            address_type: "{{ item.type }}"
            description: "{{ item.description | default('') }}"
            tag: "{{ item.tags | default([]) }}"
            state: present
          loop: "{{ address_objects | default([]) }}"
          loop_control:
            label: "Address object: {{ item.name }} ({{ item.value }})"
          notify: Commit PAN-OS configuration

      tags: [objects, address-objects]

    # ── Section 4: Address Groups ─────────────────────────────────
    # Must exist before security rules reference them
    - block:
        - name: "Objects | Create address groups"
          paloaltonetworks.panos.panos_address_group:
            provider: "{{ panos_provider }}"
            name: "{{ item.name }}"
            static_value: "{{ item.members }}"
            description: "{{ item.description | default('') }}"
            tag: "{{ item.tags | default([]) }}"
            state: present
          loop: "{{ address_groups | default([]) }}"
          loop_control:
            label: "Address group: {{ item.name }} ({{ item.members | length }} members)"
          notify: Commit PAN-OS configuration

      tags: [objects, address-groups]

    # ── Section 5: Service Objects ────────────────────────────────
    - block:
        - name: "Objects | Create service objects"
          paloaltonetworks.panos.panos_service_object:
            provider: "{{ panos_provider }}"
            name: "{{ item.name }}"
            protocol: "{{ item.protocol }}"
            destination_port: "{{ item.destination_port | string }}"
            description: "{{ item.description | default('') }}"
            state: present
          loop: "{{ service_objects | default([]) }}"
          loop_control:
            label: "Service: {{ item.name }} ({{ item.protocol }}/{{ item.destination_port }})"
          notify: Commit PAN-OS configuration

        - name: "Objects | Create service groups"
          paloaltonetworks.panos.panos_service_group:
            provider: "{{ panos_provider }}"
            name: "{{ item.name }}"
            value: "{{ item.members }}"
            state: present
          loop: "{{ service_groups | default([]) }}"
          loop_control:
            label: "Service group: {{ item.name }}"
          notify: Commit PAN-OS configuration

      tags: [objects, service-objects]

    # ── Section 6: Log Forwarding Profile ─────────────────────────
    # Must exist before security rules reference it
    - block:
        - name: "Logging | Create log forwarding profiles"
          paloaltonetworks.panos.panos_log_forwarding_profile:
            provider: "{{ panos_provider }}"
            name: "{{ item.name }}"
            description: "{{ item.description | default('') }}"
            state: present
          loop: "{{ log_forwarding_profiles | default([]) }}"
          loop_control:
            label: "Log profile: {{ item.name }}"
          notify: Commit PAN-OS configuration

        - name: "Logging | Configure log forwarding match lists"
          paloaltonetworks.panos.panos_log_forwarding_profile_match_list:
            provider: "{{ panos_provider }}"
            log_forwarding_profile: "{{ item.0.name }}"
            name: "{{ item.1.name }}"
            log_type: "{{ item.1.log_type }}"
            filter: "{{ item.1.filter }}"
            syslog: "{{ item.1.syslog_destinations | map(attribute='name') | list }}"
            state: present
          loop: "{{ log_forwarding_profiles | default([]) | subelements('match_list') }}"
          loop_control:
            label: "{{ item.0.name }}/{{ item.1.name }}"
          notify: Commit PAN-OS configuration

      tags: [logging, objects]

    # ── Section 7: Security Rules ─────────────────────────────────
    # All referenced objects must exist before creating rules
    # Rule ORDER matters — rules are evaluated top-to-bottom
    # The loop preserves data model order
    - block:
        - name: "Security | Create security policy rules"
          paloaltonetworks.panos.panos_security_rule:
            provider: "{{ panos_provider }}"
            rule_name: "{{ item.name }}"
            description: "{{ item.description | default('') }}"
            source_zone: "{{ item.source_zone }}"
            destination_zone: "{{ item.destination_zone }}"
            source_ip: "{{ item.source_address }}"
            destination_ip: "{{ item.destination_address }}"
            application: "{{ item.application }}"
            service: "{{ item.service }}"
            action: "{{ item.action }}"
            log_start: "{{ item.log_start | default(false) }}"
            log_end: "{{ item.log_end | default(true) }}"
            log_setting: "{{ item.log_forwarding | default(omit) }}"
            tag_name: "{{ item.tags | default([]) }}"
            state: present
          loop: "{{ security_rules | default([]) }}"
          loop_control:
            label: "Rule: {{ item.name }} ({{ item.action }})"
          notify: Commit PAN-OS configuration

        - name: "Security | Enforce rule ordering"
          paloaltonetworks.panos.panos_security_rule:
            provider: "{{ panos_provider }}"
            rule_name: "{{ item.name }}"
            location: bottom
            state: present
          loop: "{{ security_rules | default([]) | reverse | list }}"
          loop_control:
            label: "Ordering: {{ item.name }}"
          # Move each rule to bottom in reverse order = original order at top
          # This ensures the data model order is enforced even if rules already exist
          notify: Commit PAN-OS configuration

      tags: [security-rules, rules]

    # ── Section 8: NAT Rules ──────────────────────────────────────
    - block:
        - name: "NAT | Create NAT rules"
          paloaltonetworks.panos.panos_nat_rule:
            provider: "{{ panos_provider }}"
            rule_name: "{{ item.name }}"
            description: "{{ item.description | default('') }}"
            source_zone: "{{ item.source_zone }}"
            destination_zone: "{{ item.destination_zone }}"
            source_ip: "{{ item.source_address }}"
            destination_ip: "{{ item.destination_address }}"
            service: "{{ item.service | default('any') }}"
            nat_type: "{{ item.nat_type | default('ipv4') }}"
            # Source NAT
            snat_type: >-
              {{ item.source_translation.type
                 if item.source_translation is defined else omit }}
            snat_interface: >-
              {{ item.source_translation.interface_address.interface
                 if item.source_translation is defined
                 and item.source_translation.type == 'dynamic-ip-and-port'
                 else omit }}
            # Destination NAT
            dnat_address: >-
              {{ item.destination_translation.translated_address
                 if item.destination_translation is defined else omit }}
            dnat_port: >-
              {{ item.destination_translation.translated_port | string
                 if item.destination_translation is defined else omit }}
            tag_name: "{{ item.tags | default([]) }}"
            state: present
          loop: "{{ nat_rules | default([]) }}"
          loop_control:
            label: "NAT rule: {{ item.name }}"
          notify: Commit PAN-OS configuration

      tags: [nat-rules, rules]

  # ── Handler: Commit (simple approach) ───────────────────────────
  handlers:
    - name: Commit PAN-OS configuration
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"
        description: "Committed by Ansible — {{ lookup('pipe', 'date') }}"
      listen: Commit PAN-OS configuration
      # Handler fires ONCE at end of play regardless of how many tasks notified it
      # This is the simple approach — appropriate when no post-commit verification needed
      # For production with verification, use the two-phase play below
```

### Phase 2: Production Commit with Verification

For production environments where post-commit verification is required:

```yaml
---
# =============================================================
# deploy_panos_commit.yml — Two-phase production commit
# Run after deploy_panos.yml when handler-based commit isn't enough
# Verifies policy is behaving correctly after commit
# =============================================================

- name: "PAN-OS | Production commit with verification"
  hosts: panos_devices
  gather_facts: false
  connection: local
  tags: commit

  vars:
    panos_provider:
      ip_address: "{{ ansible_host }}"
      api_key: "{{ api_key }}"

  tasks:
    - name: "Commit | Check for pending changes before commit"
      paloaltonetworks.panos.panos_op:
        provider: "{{ panos_provider }}"
        cmd: "show config list changes"
      register: pending_changes
      changed_when: false

    - name: "Commit | Display pending changes"
      ansible.builtin.debug:
        msg: "{{ pending_changes.stdout }}"
        verbosity: 1

    - name: "Commit | Execute commit to firewall"
      paloaltonetworks.panos.panos_commit_firewall:
        provider: "{{ panos_provider }}"
        description: "Production deploy — {{ lookup('pipe', 'date') }}"
        admins: ["ansible"]    # Commit only changes made by ansible admin
      register: commit_result

    - name: "Commit | Wait for commit to complete"
      paloaltonetworks.panos.panos_op:
        provider: "{{ panos_provider }}"
        cmd: "show jobs id {{ commit_result.jobid }}"
      register: commit_job
      until: "'FIN' in commit_job.stdout"
      retries: 30
      interval: 10
      changed_when: false
      # PAN-OS commits are asynchronous — they return a job ID
      # Poll until the job shows FIN (finished) status

    - name: "Verify | Check security policy is active"
      paloaltonetworks.panos.panos_op:
        provider: "{{ panos_provider }}"
        cmd: "show security policy"
      register: active_policy
      changed_when: false

    - name: "Verify | Assert critical rules are present"
      ansible.builtin.assert:
        that:
          - "item.name in active_policy.stdout"
        fail_msg: "Critical rule '{{ item.name }}' not found in active policy"
        success_msg: "PASS: Rule '{{ item.name }}' is active"
      loop: "{{ security_rules | default([]) | selectattr('tags', 'contains', 'production') | list }}"
      loop_control:
        label: "{{ item.name }}"

    - name: "Verify | Check interface status post-commit"
      paloaltonetworks.panos.panos_op:
        provider: "{{ panos_provider }}"
        cmd: "show interface all"
      register: interface_status
      changed_when: false

    - name: "Verify | Display interface summary"
      ansible.builtin.debug:
        msg: "{{ interface_status.stdout }}"
        verbosity: 1

    - name: "Commit | Report commit outcome"
      ansible.builtin.debug:
        msg:
          - "═══════════════════════════════════════════════"
          - "PAN-OS COMMIT COMPLETE: {{ inventory_hostname }}"
          - "Job ID: {{ commit_result.jobid }}"
          - "Status: {{ commit_job.stdout | regex_search('Status.*') | default('Unknown') }}"
          - "Production rules verified: PASS"
          - "═══════════════════════════════════════════════"
```

---

## 21.5 — Application-Based Security Rules

PAN-OS App-ID identifies applications by behavior, not port. The `application:` field in security rules takes App-ID names — not port numbers. This is what makes PAN-OS fundamentally different from ACL-based firewall automation.

```yaml
# App-ID reference examples for common applications
# These are Palo Alto's built-in application signatures

security_rules:
  - name: ALLOW-WEB-BROWSING
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [any]
    destination_address: [any]
    application:
      - web-browsing    # HTTP (port 80) — but only actual web browsing, not raw TCP/80
      - ssl             # HTTPS (port 443) — TLS-identified web traffic
    service: [application-default]    # Use the app's default port
    action: allow

  - name: ALLOW-OFFICE365
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [any]
    application:
      - ms-office365        # Microsoft Office 365
      - ms-teams            # Microsoft Teams
      - ms-teams-audio      # Teams audio specifically
      - ms-teams-video      # Teams video specifically
      - ms-onedrive         # OneDrive sync
    service: [application-default]
    action: allow

  - name: BLOCK-SOCIAL-MEDIA
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [any]
    application:
      - facebook           # Facebook (all traffic)
      - instagram          # Instagram
      - tiktok             # TikTok
      - twitter            # Twitter/X
    service: [any]
    action: deny
    log_end: true

  - name: ALLOW-ENCRYPTED-UNKNOWN
    # When SSL inspection isn't deployed, allow-unknown-ssl is sometimes needed
    source_zone: [trust]
    destination_zone: [untrust]
    source_address: [INTERNAL-NET]
    destination_address: [any]
    application:
      - ssl                # Known SSL applications
      - unknown-tcp        # Unknown TCP — use carefully, audit regularly
    service: [application-default]
    action: allow
```

### App-ID vs Port-Based Rules: The Key Difference

```yaml
# Traditional port-based rule (works but misses App-ID benefits):
- name: ALLOW-HTTPS-PORT-443
  application: [any]           # Any application on port 443
  service: [SVC-HTTPS]         # TCP/443 specifically
  action: allow
  # Allows: legitimate HTTPS, but also any app that tunnel over 443

# App-ID based rule (recommended):
- name: ALLOW-SSL-APPLICATIONS
  application: [ssl, web-browsing, ms-office365]   # Specific app signatures
  service: [application-default]  # App uses its default port
  action: allow
  # Allows: only apps that PAN-OS identifies as SSL/web-browsing/Office365
  # Blocks: malware using port 443 for C2 that doesn't match SSL signatures
```

---

## 21.6 — PAN-OS Module Reference

### `panos_facts` — Device Information

```yaml
paloaltonetworks.panos.panos_facts:
  provider: "{{ panos_provider }}"
# Returns: ansible_net_hostname, ansible_net_version, ansible_net_model,
#          ansible_net_serial, ansible_net_uptime
```

### `panos_op` — Run Operational Commands

```yaml
paloaltonetworks.panos.panos_op:
  provider: "{{ panos_provider }}"
  cmd: "show interface all"
  # OR XML format:
  cmd_is_xml: false
register: op_result
# result in op_result.stdout (text) or op_result.xml (XML)
```

### `panos_security_rule` — Security Policy

```yaml
paloaltonetworks.panos.panos_security_rule:
  provider: "{{ panos_provider }}"
  rule_name: ALLOW-WEB
  description: "Permit web traffic"
  source_zone: [trust]
  destination_zone: [untrust]
  source_ip: [any]          # Or list of address object/group names
  destination_ip: [any]
  application: [web-browsing, ssl]
  service: [application-default]
  action: allow             # allow | deny | drop | reset-client | reset-server
  log_end: true
  log_setting: SYSLOG-FORWARDING
  location: bottom          # top | bottom | before | after
  state: present            # present | absent
```

### `panos_nat_rule` — NAT Policy

```yaml
paloaltonetworks.panos.panos_nat_rule:
  provider: "{{ panos_provider }}"
  rule_name: OUTBOUND-PAT
  source_zone: [trust]
  destination_zone: [untrust]
  source_ip: [INTERNAL-NET]
  destination_ip: [any]
  # Source NAT — dynamic IP and port (PAT)
  snat_type: dynamic-ip-and-port
  snat_interface: ethernet1/1
  state: present
```

### `panos_address_object` — Address Objects

```yaml
paloaltonetworks.panos.panos_address_object:
  provider: "{{ panos_provider }}"
  name: SERVER-WEB-01
  value: 172.16.10.10/32
  address_type: ip-netmask   # ip-netmask | ip-range | fqdn
  description: "Primary web server"
  tag: [dmz, web, production]
  state: present
```

### `panos_commit_firewall` — Commit

```yaml
# Standard commit
paloaltonetworks.panos.panos_commit_firewall:
  provider: "{{ panos_provider }}"
  description: "Deployed by Ansible"

# Commit only specific admin's changes
paloaltonetworks.panos.panos_commit_firewall:
  provider: "{{ panos_provider }}"
  admins: ["ansible"]
```

---

## 21.7 — Panorama: What Changes

Panorama is Palo Alto's centralized management platform — it manages multiple firewalls from a single pane. The automation model is the same but with two differences: the target is Panorama (not the firewall directly), and configuration lives in device groups and templates rather than directly on the firewall.

### Connection Change

```yaml
# group_vars/panorama.yml
ansible_host: 172.16.0.60    # Panorama management IP (not firewall)
ansible_connection: local

# host_vars/panorama-01.yml
panos_provider:
  ip_address: "{{ ansible_host }}"
  api_key: "{{ vault_panorama_api_key }}"
```

### Device Groups and Templates

```yaml
# On Panorama, address objects live in device groups, not vsys
paloaltonetworks.panos.panos_address_object:
  provider: "{{ panos_provider }}"
  name: SERVER-WEB-01
  value: 172.16.10.10/32
  address_type: ip-netmask
  device_group: PRODUCTION-DG    # ← Device group, not vsys
  state: present

# Security rules also target device groups
paloaltonetworks.panos.panos_security_rule:
  provider: "{{ panos_provider }}"
  rule_name: ALLOW-WEB
  source_zone: [trust]
  destination_zone: [untrust]
  application: [web-browsing]
  action: allow
  device_group: PRODUCTION-DG    # ← All rules go to a device group
  rulebase: pre-rulebase         # pre-rulebase | post-rulebase
  state: present
```

### Panorama Commit — Two Steps

Panorama commits have two stages: commit to Panorama (saves to Panorama), then push to devices (deploys to managed firewalls):

```yaml
# Step 1: Commit to Panorama
paloaltonetworks.panos.panos_commit_panorama:
  provider: "{{ panos_provider }}"
  description: "Committed to Panorama by Ansible"

# Step 2: Push from Panorama to managed firewalls
paloaltonetworks.panos.panos_push_to_devices:
  provider: "{{ panos_provider }}"
  device_groups: [PRODUCTION-DG]
  description: "Pushed from Panorama by Ansible"
  # This deploys the Panorama config to all firewalls in the device group
```

### When to Use Panorama vs Standalone

```
Standalone firewall automation:
  - Target: firewall management IP directly
  - Objects: in vsys1 (default vsys)
  - Commit: panos_commit_firewall
  - Use when: single firewall or small number of independently managed firewalls

Panorama automation:
  - Target: Panorama IP
  - Objects: in device groups and templates
  - Commit: panos_commit_panorama → panos_push_to_devices
  - Use when: managing many firewalls centrally, shared policy across sites
```

---

## 21.8 — Common PAN-OS Automation Gotchas

### ### 🪲 Gotcha — Rule ordering requires explicit management

`panos_security_rule` with `state: present` creates the rule but doesn't guarantee its position in the rulebase. A new rule added to the middle of the data model list may be created at the bottom of the rulebase. Always enforce order explicitly:

```yaml
# After creating all rules, enforce order by moving each to bottom in reverse
- name: "Security | Enforce rule ordering"
  paloaltonetworks.panos.panos_security_rule:
    provider: "{{ panos_provider }}"
    rule_name: "{{ item.name }}"
    location: bottom
    state: present
  loop: "{{ security_rules | default([]) | reverse | list }}"
  loop_control:
    label: "Order: {{ item.name }}"
```

Moving in reverse order from bottom achieves the correct final order: move last rule to bottom, move second-to-last to bottom (it goes above the last), and so on.

### ### 🪲 Gotcha — Nothing is active until commit — test in check mode first

All `panos_*` module tasks succeed even if the configuration they produce would cause a commit failure. The commit is where PAN-OS validates consistency. A rule referencing a non-existent address group succeeds as a task but fails at commit time:

```bash
# Always run --check first to see what would change without committing
ansible-playbook deploy_panos.yml --check

# Then run without --check — objects and rules pushed to candidate
ansible-playbook deploy_panos.yml --skip-tags commit

# Review the candidate diff via panos_op
ansible panos-fw01 -m paloaltonetworks.panos.panos_op \
  -a "provider={{ panos_provider }} cmd='show config diff'"

# Only commit when satisfied
ansible-playbook deploy_panos_commit.yml
```

### ### 🪲 Gotcha — Commit jobs are asynchronous — always poll for completion

`panos_commit_firewall` returns a job ID immediately but the commit may take 30-120 seconds. Tasks that run after commit but before it completes will interact with the pre-commit state:

```yaml
# Always wait for commit completion using panos_op to poll the job
- name: "Wait for commit job to finish"
  paloaltonetworks.panos.panos_op:
    provider: "{{ panos_provider }}"
    cmd: "show jobs id {{ commit_result.jobid }}"
  register: job_status
  until: "'FIN' in job_status.stdout or 'FAIL' in job_status.stdout"
  retries: 30
  interval: 10
  changed_when: false

- name: "Assert commit succeeded"
  ansible.builtin.assert:
    that:
      - "'FIN' in job_status.stdout"
      - "'FAIL' not in job_status.stdout"
    fail_msg: "Commit job {{ commit_result.jobid }} failed"
```

### ### 🪲 Gotcha — `application-default` service vs specific service objects

In security rules, `service: [application-default]` tells PAN-OS to match traffic on whatever port the App-ID expects. `service: [SVC-HTTPS]` tells PAN-OS to match traffic on TCP/443 regardless of App-ID. Mixing them causes unexpected behavior:

```yaml
# ✅ Consistent — app-default means the app's known port
application: [web-browsing]
service: [application-default]    # Web browsing on port 80

# ✅ Consistent — specific service overrides app-default
application: [any]
service: [SVC-HTTPS]              # Anything on TCP/443

# ❌ Inconsistent — may not match what you expect
application: [web-browsing]       # Web browsing expected on port 80
service: [SVC-HTTPS]              # But you're restricting to port 443
# Result: only matches web browsing on port 443 (HTTPS web browsing)
# May miss HTTP web browsing entirely
```

---



PAN-OS automation is complete — the XML API model, the object hierarchy, application-based security rules, NAT, log forwarding, and the two commit patterns for simple and production use. Part 22 shifts from platform-specific automation to network-wide operations: compliance checking, bulk fact collection, and generating reports across the entire multi-vendor lab.

