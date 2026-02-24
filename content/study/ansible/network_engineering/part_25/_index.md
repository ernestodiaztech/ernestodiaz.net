---
draft: true
title: '25 - Users & AAA'
weight: 25
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 25: Users, Privilege Levels & AAA Automation

> *User and AAA configuration is where automation intersects directly with security. A playbook that manages device users must be correct — a bug that removes the wrong account or pushes a bad password to every device in the fleet can lock out operations entirely. This part builds the user management system carefully: IOS privilege levels first, NX-OS role model second, TACACS+ AAA configuration third, password rotation with per-device verification, and a fleet-wide audit report that shows who actually exists on every device.*

---

## 25.1 — IOS User and Privilege Model

### The IOS Privilege Level System

IOS uses privilege levels 0–15 to control command access. The three levels with built-in meaning are:

```
Level 0   — Minimal access: logout, enable, disable, help, exit only
Level 1   — User EXEC: show commands, ping, traceroute (default unprivileged)
Level 15  — Privileged EXEC: all commands, global config mode (default admin)
```

Levels 2–14 are customizable — specific commands can be assigned to specific levels. In practice, most environments use only levels 1 and 15, with level 15 being the target for any automation service account.

```
# On IOS, privilege levels work like this:
# User logs in → enters user EXEC (level 1 by default)
# User types 'enable' → enters privileged EXEC (level 15)
# 'enable secret' protects the transition from 1 → 15

# A user configured with privilege 15 skips 'enable' entirely:
# User logs in → immediately in privileged EXEC
```

### IOS User Data Model

The user data model in `group_vars` defines the standard set of accounts across all IOS devices:

```bash
cat >> ~/projects/ansible-network/inventory/group_vars/cisco_ios.yml << 'EOF'

# ── User management ───────────────────────────────────────────────
# Standard user accounts deployed to all IOS devices
# Sensitive passwords come from vault — see group_vars/all/vault.yml

ios_local_users:
  - username: ansible
    privilege: 15
    password: "{{ vault_ansible_device_password }}"
    sshkey: "{{ lookup('file', '~/.ssh/ansible_ed25519.pub') }}"
    state: present
    description: "Ansible automation service account — do not remove"

  - username: netadmin
    privilege: 15
    password: "{{ vault_netadmin_password }}"
    state: present
    description: "Network administrator account"

  - username: netops
    privilege: 1
    password: "{{ vault_netops_password }}"
    state: present
    description: "Network operations read-only account"

  - username: auditor
    privilege: 1
    password: "{{ vault_auditor_password }}"
    state: present
    description: "Security auditor read-only account"

# Enable secret — protects privilege escalation
ios_enable_secret: "{{ vault_ios_enable_secret }}"

# VTY line configuration
ios_vty_config:
  login_local: true       # Use local user database for authentication
  transport_input: ssh    # SSH only — no telnet
  exec_timeout_minutes: 10
  exec_timeout_seconds: 0
  access_class: VTY-ACCESS  # Optional ACL to restrict source IPs
EOF
```

```bash
# Add vault variables
cat >> ~/projects/ansible-network/inventory/group_vars/all/vault.yml << 'EOF'
# (encrypt this file with ansible-vault encrypt)
vault_ansible_device_password: "AnsibleService#2025"
vault_netadmin_password: "NetAdmin#2025"
vault_netops_password: "NetOps#ReadOnly2025"
vault_auditor_password: "Auditor#ReadOnly2025"
vault_ios_enable_secret: "Enable#Secret2025"
EOF
```

---

## 25.2 — IOS User and AAA Playbook

```bash
cat > ~/projects/ansible-network/playbooks/security/manage_users_ios.yml << 'EOF'
---
# =============================================================
# manage_users_ios.yml — IOS/IOS-XE user and AAA management
#
# INSTALL:   ansible-playbook manage_users_ios.yml --tags install
# UNINSTALL: ansible-playbook manage_users_ios.yml --tags uninstall
#
# Section tags:
#   users     — local user accounts and privilege levels
#   enable    — enable secret configuration
#   ssh       — SSH public key deployment
#   vty       — VTY line authentication and transport settings
#   aaa       — TACACS+/RADIUS AAA configuration
#
# Safety:
#   The ansible service account is NEVER removed by uninstall tasks.
#   Password rotation uses serial: 1 — see rotate_passwords_ios.yml.
# =============================================================

- name: "IOS | User and AAA management"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:

    # ── Pre-flight ────────────────────────────────────────────
    - name: "Pre-flight | Gather IOS facts"
      cisco.ios.ios_facts:
        gather_subset: [default]
      tags: always

    # ══════════════════════════════════════════════════════════
    # SECTION: Local users
    # ══════════════════════════════════════════════════════════

    - name: "Users | INSTALL | Configure local user accounts"
      cisco.ios.ios_user:
        name: "{{ item.username }}"
        privilege: "{{ item.privilege | default(1) }}"
        configured_password: "{{ item.password }}"
        password_type: secret      # Always use 'secret' (type 9 hash) not 'password'
        update_password: always    # Re-push password on every run
                                   # Required for rotation to work correctly
        state: "{{ item.state | default('present') }}"
      loop: "{{ ios_local_users | default([]) }}"
      loop_control:
        label: "User: {{ item.username }} (priv {{ item.privilege | default(1) }})"
      no_log: true                 # Never log passwords — vault content must stay hidden
      tags: [install, users]
      notify: Save IOS configuration
      # IOS difference: 'username X secret Y' stores a type 9 (scrypt) hash
      # 'password_type: secret' ensures this — never use password_type: password
      # (type 7 encoding is trivially reversible — not encryption)

    - name: "Users | UNINSTALL | Remove non-essential user accounts"
      cisco.ios.ios_user:
        name: "{{ item.username }}"
        state: absent
      loop: >-
        {{ ios_local_users | default([])
           | rejectattr('username', 'equalto', 'ansible')
           | rejectattr('username', 'equalto', ansible_user) | list }}
      loop_control:
        label: "Remove user: {{ item.username }}"
      tags: [uninstall, users]
      notify: Save IOS configuration
      # Two accounts are never removed:
      #   'ansible' — the service account that runs these playbooks
      #   ansible_user — whoever is currently connected

    # ══════════════════════════════════════════════════════════
    # SECTION: Enable secret
    # ══════════════════════════════════════════════════════════

    - name: "Enable | INSTALL | Set enable secret"
      cisco.ios.ios_config:
        lines:
          - "enable secret 9 {{ ios_enable_secret | password_hash('sha512') }}"
        # IOS 15.x+ supports type 9 (scrypt) enable secret
        # Older IOS: use 'enable secret 5' with MD5 hash or plain text (IOS hashes it)
      no_log: true
      tags: [install, enable]
      notify: Save IOS configuration

    - name: "Enable | UNINSTALL | Remove enable secret (leaves enable unprotected)"
      cisco.ios.ios_config:
        lines:
          - "no enable secret"
      tags: [uninstall, enable]
      notify: Save IOS configuration
      # WARNING: removing enable secret means 'enable' requires no password
      # Only appropriate when AAA handles privilege escalation

    # ══════════════════════════════════════════════════════════
    # SECTION: SSH public key authentication
    # ══════════════════════════════════════════════════════════

    - name: "SSH | INSTALL | Enable SSH and generate RSA keys"
      cisco.ios.ios_config:
        lines:
          - "crypto key generate rsa modulus 4096"
          - "ip ssh version 2"
          - "ip ssh time-out 60"
          - "ip ssh authentication-retries 3"
        # crypto key generate is not idempotent on older IOS
        # On IOS-XE 16.x+: use 'crypto key generate rsa modulus 4096 label ANSIBLE'
      changed_when: false          # Key generation never reports changed reliably
      tags: [install, ssh]
      notify: Save IOS configuration

    - name: "SSH | INSTALL | Deploy SSH public key for ansible user"
      cisco.ios.ios_config:
        lines:
          - "ip ssh pubkey-chain"
          - " username ansible"
          - "  key-string"
          - "  {{ ansible_ssh_pub_key | default(lookup('file', '~/.ssh/ansible_ed25519.pub')) }}"
          - "  exit"
          - " exit"
        # IOS SSH public key config uses 'ip ssh pubkey-chain'
        # The key must be split if longer than 254 chars (RSA keys need splitting)
        # Ed25519 keys are short enough to fit on a single line
      tags: [install, ssh]
      notify: Save IOS configuration

    - name: "SSH | UNINSTALL | Remove SSH public key for ansible user"
      cisco.ios.ios_config:
        lines:
          - "ip ssh pubkey-chain"
          - " no username ansible"
          - " exit"
      tags: [uninstall, ssh]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: VTY line configuration
    # ══════════════════════════════════════════════════════════

    - name: "VTY | INSTALL | Configure VTY lines"
      cisco.ios.ios_config:
        parents: "line vty 0 15"
        lines:
          - "login local"
          - "transport input ssh"
          - "exec-timeout {{ ios_vty_config.exec_timeout_minutes | default(10) }} \
             {{ ios_vty_config.exec_timeout_seconds | default(0) }}"
      tags: [install, vty]
      notify: Save IOS configuration

    - name: "VTY | INSTALL | Apply VTY access-class if defined"
      cisco.ios.ios_config:
        parents: "line vty 0 15"
        lines:
          - "access-class {{ ios_vty_config.access_class }} in"
      when: ios_vty_config.access_class is defined
      tags: [install, vty]
      notify: Save IOS configuration

    - name: "VTY | INSTALL | Configure console line"
      cisco.ios.ios_config:
        parents: "line con 0"
        lines:
          - "login local"
          - "exec-timeout 15 0"
          - "logging synchronous"
      tags: [install, vty]
      notify: Save IOS configuration

    - name: "VTY | UNINSTALL | Reset VTY lines to permissive defaults"
      cisco.ios.ios_config:
        parents: "line vty 0 15"
        lines:
          - "login"              # Default: console password, not local db
          - "transport input all"  # Allow telnet and SSH
          - "no exec-timeout"    # No timeout (IOS default)
          - "no access-class"
      tags: [uninstall, vty]
      notify: Save IOS configuration

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration
EOF
```

---

## 25.3 — TACACS+ AAA Configuration (Full) and RADIUS (Brief)

### Understanding AAA on Cisco IOS

AAA stands for Authentication, Authorization, and Accounting:

```
Authentication — Who are you? (username + password check)
Authorization  — What can you do? (privilege level, command sets)
Accounting     — What did you do? (command logging, session tracking)

Without AAA:  device uses local user database for all three
With AAA:     device queries TACACS+ or RADIUS server for auth/authz/acct
              local database is fallback if server is unreachable
```

The order matters: `aaa authentication login default group tacacs+ local` means try TACACS+ first, fall back to local if TACACS+ is unreachable. Getting this wrong locks out access entirely.

### TACACS+ Data Model

```bash
cat >> ~/projects/ansible-network/inventory/group_vars/cisco_ios.yml << 'EOF'

# ── AAA / TACACS+ configuration ───────────────────────────────────
aaa:
  enabled: false       # Set true to enable AAA — false = local auth only
                       # Never enable on a device where TACACS+ is unverified

  tacacs_servers:
    - name: TACACS-PRIMARY
      address: 172.16.0.200
      key: "{{ vault_tacacs_key }}"
      port: 49          # TACACS+ default port
      timeout: 5        # Seconds before trying next server
    - name: TACACS-SECONDARY
      address: 172.16.0.201
      key: "{{ vault_tacacs_key }}"
      port: 49
      timeout: 5

  # Authentication: who can log in
  # 'group tacacs+' = try TACACS+, 'local' = fallback to local DB
  authentication:
    login: "group tacacs+ local"    # VTY and console login
    enable: "group tacacs+ enable"  # 'enable' command authentication

  # Authorization: what commands are permitted
  authorization:
    exec: "group tacacs+ local"     # Exec mode authorization
    commands_1: "group tacacs+"     # Authorize all level-1 commands
    commands_15: "group tacacs+"    # Authorize all level-15 commands

  # Accounting: log what commands were run
  accounting:
    exec: "start-stop group tacacs+"
    commands_1: "start-stop group tacacs+"
    commands_15: "start-stop group tacacs+"
EOF
```

```bash
# Add TACACS+ vault variable
cat >> ~/projects/ansible-network/inventory/group_vars/all/vault.yml << 'EOF'
vault_tacacs_key: "TacacsSharedSecret#2025"
vault_radius_key: "RadiusSharedSecret#2025"
EOF
```

### TACACS+ and AAA Playbook Tasks

```bash
cat >> ~/projects/ansible-network/playbooks/security/manage_users_ios.yml << 'EOF'

    # ══════════════════════════════════════════════════════════
    # SECTION: TACACS+ and AAA
    # ══════════════════════════════════════════════════════════
    # CRITICAL ORDERING NOTE:
    # 1. TACACS+ server config must be pushed FIRST
    # 2. Verify TACACS+ is reachable BEFORE enabling AAA
    # 3. Local fallback must always be in every AAA method list
    # 4. Never run AAA config tasks without console/OOB access available
    # ══════════════════════════════════════════════════════════

    - name: "AAA | INSTALL | Pre-flight TACACS+ reachability check"
      cisco.ios.ios_command:
        commands:
          - "ping {{ item.address }} repeat 3"
        wait_for:
          - result[0] contains !!
        retries: 2
        interval: 3
      loop: "{{ aaa.tacacs_servers | default([]) }}"
      loop_control:
        label: "Ping TACACS+ {{ item.address }}"
      changed_when: false
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      # NEVER enable AAA if TACACS+ servers are unreachable
      # A misconfigured AAA with unreachable server + no local fallback = lockout

    - name: "AAA | INSTALL | Configure TACACS+ server groups"
      cisco.ios.ios_config:
        lines:
          - "tacacs server {{ item.name }}"
          - " address ipv4 {{ item.address }}"
          - " key 7 {{ item.key }}"
          - " port {{ item.port | default(49) }}"
          - " timeout {{ item.timeout | default(5) }}"
      loop: "{{ aaa.tacacs_servers | default([]) }}"
      loop_control:
        label: "TACACS+ server: {{ item.name }} ({{ item.address }})"
      no_log: true
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      notify: Save IOS configuration

    - name: "AAA | INSTALL | Create TACACS+ server group"
      cisco.ios.ios_config:
        parents: "aaa group server tacacs+ TACACS-GROUP"
        lines: >-
          {{ aaa.tacacs_servers | default([])
             | map(attribute='name')
             | map('regex_replace', '^', 'server name ')
             | list }}
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      notify: Save IOS configuration

    - name: "AAA | INSTALL | Enable AAA new-model"
      cisco.ios.ios_config:
        lines:
          - "aaa new-model"
        # 'aaa new-model' is the global switch that activates AAA
        # Once enabled, ALL login uses AAA method lists
        # This is the point of no return — have local fallback ready
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      notify: Save IOS configuration

    - name: "AAA | INSTALL | Configure authentication method lists"
      cisco.ios.ios_config:
        lines:
          - "aaa authentication login default {{ aaa.authentication.login }}"
          - "aaa authentication enable default {{ aaa.authentication.enable }}"
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      notify: Save IOS configuration
      # 'local' MUST appear in every method list as final fallback
      # 'group tacacs+ local' = TACACS+ first, local DB if server unreachable

    - name: "AAA | INSTALL | Configure authorization method lists"
      cisco.ios.ios_config:
        lines:
          - "aaa authorization exec default {{ aaa.authorization.exec }}"
          - "aaa authorization commands 1 default {{ aaa.authorization.commands_1 }}"
          - "aaa authorization commands 15 default {{ aaa.authorization.commands_15 }}"
          - "aaa authorization config-commands"  # Authorize config mode commands too
      when:
        - aaa.enabled | default(false) | bool
        - aaa.authorization is defined
      tags: [install, aaa]
      notify: Save IOS configuration

    - name: "AAA | INSTALL | Configure accounting method lists"
      cisco.ios.ios_config:
        lines:
          - "aaa accounting exec default {{ aaa.accounting.exec }}"
          - "aaa accounting commands 1 default {{ aaa.accounting.commands_1 }}"
          - "aaa accounting commands 15 default {{ aaa.accounting.commands_15 }}"
      when:
        - aaa.enabled | default(false) | bool
        - aaa.accounting is defined
      tags: [install, aaa]
      notify: Save IOS configuration

    - name: "AAA | INSTALL | Verify AAA authentication is working"
      cisco.ios.ios_command:
        commands:
          - "test aaa group tacacs+ {{ ansible_user }} {{ ansible_password }} legacy"
        # 'test aaa' verifies TACACS+ authentication without logging out
        # Returns: "Attempting authentication test to server-group tacacs+..."
        # Success output contains "User was successfully authenticated."
      register: aaa_test
      changed_when: false
      no_log: true
      failed_when: "'successfully authenticated' not in aaa_test.stdout[0]"
      when: aaa.enabled | default(false) | bool
      tags: [install, aaa]
      # This test MUST pass before moving on
      # If it fails, the play stops — AAA config is not saved (handler won't fire)
      # The device still uses local auth until 'write memory' is called

    - name: "AAA | UNINSTALL | Disable AAA and remove TACACS+ config"
      cisco.ios.ios_config:
        lines:
          - "no aaa new-model"
          - "no aaa authentication login default"
          - "no aaa authentication enable default"
          - "no aaa authorization exec default"
          - "no aaa authorization commands 1 default"
          - "no aaa authorization commands 15 default"
          - "no aaa accounting exec default"
          - "no aaa accounting commands 1 default"
          - "no aaa accounting commands 15 default"
      tags: [uninstall, aaa]
      notify: Save IOS configuration

    - name: "AAA | UNINSTALL | Remove TACACS+ server definitions"
      cisco.ios.ios_config:
        lines:
          - "no tacacs server {{ item.name }}"
      loop: "{{ aaa.tacacs_servers | default([]) }}"
      loop_control:
        label: "Remove TACACS+ server: {{ item.name }}"
      tags: [uninstall, aaa]
      notify: Save IOS configuration
EOF
```

### RADIUS: Brief Reference

RADIUS uses a different protocol (UDP 1812/1813 vs TCP 49) and has less granular authorization than TACACS+. The IOS configuration follows the same `aaa new-model` pattern:

```yaml
# RADIUS equivalent of the TACACS+ config above
# Key differences:
#   - Protocol: UDP (not TCP)
#   - Port: 1812 auth, 1813 accounting (not 49)
#   - Authorization: less granular — no per-command authorization
#   - Shared secret: same key for auth and acct

- name: "AAA | INSTALL | Configure RADIUS server"
  cisco.ios.ios_config:
    lines:
      - "radius server RADIUS-PRIMARY"
      - " address ipv4 172.16.0.202 auth-port 1812 acct-port 1813"
      - " key {{ vault_radius_key }}"
      - " timeout 5"
      - " retransmit 3"
  no_log: true

- name: "AAA | INSTALL | Configure RADIUS authentication"
  cisco.ios.ios_config:
    lines:
      - "aaa new-model"
      - "aaa authentication login default group radius local"
      - "aaa authentication enable default group radius enable"
      # RADIUS authorization is limited — cannot authorize individual commands
      # Use TACACS+ when per-command authorization is required
```

---

## 25.4 — NX-OS User and Role Management

NX-OS replaces privilege levels with named roles. The two built-in roles that matter:

```
network-admin    — Full access, equivalent to IOS privilege 15
network-operator — Read-only access, equivalent to IOS privilege 1
```

Custom roles can be created for finer-grained access, but `network-admin` and `network-operator` cover most needs.

```bash
cat > ~/projects/ansible-network/playbooks/security/manage_users_nxos.yml << 'EOF'
---
# manage_users_nxos.yml — NX-OS user and AAA management

- name: "NX-OS | User and AAA management"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  vars:
    # NX-OS uses roles, not privilege levels
    nxos_local_users:
      - username: ansible
        role: network-admin    # Full access — needed for automation
        password: "{{ vault_ansible_device_password }}"
        state: present
      - username: netadmin
        role: network-admin
        password: "{{ vault_netadmin_password }}"
        state: present
      - username: netops
        role: network-operator  # Read-only
        password: "{{ vault_netops_password }}"
        state: present

  tasks:

    - name: "Users | Configure NX-OS local user accounts"
      cisco.nxos.nxos_user:
        name: "{{ item.username }}"
        role: "{{ item.role }}"
        configured_password: "{{ item.password }}"
        update_password: always
        state: "{{ item.state | default('present') }}"
      loop: "{{ nxos_local_users }}"
      loop_control:
        label: "User: {{ item.username }} ({{ item.role }})"
      no_log: true
      notify: Save NX-OS configuration
      # NX-OS difference: 'role' instead of 'privilege'
      # 'network-admin' ≈ privilege 15 on IOS
      # 'network-operator' ≈ privilege 1 on IOS

    - name: "SSH | Deploy SSH public key for ansible user"
      cisco.nxos.nxos_config:
        lines:
          - "username ansible sshkey {{ lookup('file', '~/.ssh/ansible_ed25519.pub') }}"
      notify: Save NX-OS configuration

    - name: "AAA | Configure TACACS+ on NX-OS"
      cisco.nxos.nxos_config:
        lines:
          - "feature tacacs+"
          - "tacacs-server host {{ item.address }} key {{ item.key }}"
          - "aaa group server tacacs+ TACACS-GROUP"
          - "  server {{ item.address }}"
          - "  use-vrf management"    # NX-OS difference: route AAA via management VRF
      loop: "{{ aaa.tacacs_servers | default([]) }}"
      loop_control:
        label: "TACACS+ {{ item.address }}"
      no_log: true
      when: aaa.enabled | default(false) | bool
      notify: Save NX-OS configuration
      # NX-OS difference: 'use-vrf management' routes TACACS+ traffic via mgmt VRF
      # Without this, TACACS+ uses the default VRF (data plane routing)

    - name: "AAA | Configure NX-OS AAA authentication"
      cisco.nxos.nxos_config:
        lines:
          - "aaa authentication login default group TACACS-GROUP local"
          - "aaa authentication login console local"  # NX-OS: console always uses local
      when: aaa.enabled | default(false) | bool
      notify: Save NX-OS configuration

  handlers:
    - name: Save NX-OS configuration
      cisco.nxos.nxos_command:
        commands: [copy running-config startup-config]
      listen: Save NX-OS configuration
EOF
```

---

## 25.5 — Password Rotation with Rolling Update

Password rotation is the highest-risk user management operation. A rolling update applies the new password to one device at a time, verifies the new password works before proceeding, and rolls back the individual device if verification fails. If any device fails, the play stops — preventing partial rotation across the fleet.

```bash
cat > ~/projects/ansible-network/playbooks/security/rotate_passwords_ios.yml << 'EOF'
---
# =============================================================
# rotate_passwords_ios.yml — IOS password rotation
# Rolling update: one device at a time with verification
#
# WORKFLOW:
#   1. Update vault with new passwords FIRST (see below)
#   2. Run this playbook — it pushes new password then verifies
#   3. If verification fails on any device, play stops
#   4. Fix the failed device manually, then re-run
#
# HOW TO ROTATE:
#   Step 1 — Update vault:
#     ansible-vault edit inventory/group_vars/all/vault.yml
#     Change vault_ansible_device_password to new value
#     Save and exit
#
#   Step 2 — Run rotation (staging first):
#     ansible-playbook rotate_passwords_ios.yml --limit staging
#
#   Step 3 — Verify staging, then run production:
#     ansible-playbook rotate_passwords_ios.yml --limit production
#
# ROLLBACK:
#   If a device fails verification, the playbook STOPS.
#   The failed device has the new password pushed but NOT verified.
#   Manual recovery: connect via console using the old password
#   (IOS stores the new hash immediately on push — 'write memory'
#   was not called, so reload reverts to old config)
#   OR: connect with the new password if it was accepted correctly.
#
# serial: 1 means one device at a time — never batch password changes
# =============================================================

- name: "Password rotation | IOS rolling update"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  serial: 1              # ONE DEVICE AT A TIME — never change this
  max_fail_percentage: 0 # Stop immediately if any device fails

  vars:
    rotation_user: ansible    # The account being rotated
    new_password: "{{ vault_ansible_device_password }}"
    # The new password comes from vault — change it there before running

  tasks:

    - name: "Rotate | Pre-flight | Verify current connectivity"
      cisco.ios.ios_facts:
        gather_subset: [default]
      register: pre_rotate_facts

    - name: "Rotate | Pre-flight | Display current device state"
      ansible.builtin.debug:
        msg:
          - "Device: {{ inventory_hostname }}"
          - "Hostname: {{ pre_rotate_facts.ansible_facts.ansible_net_hostname }}"
          - "Rotating password for user: {{ rotation_user }}"

    - name: "Rotate | Push new password to device"
      cisco.ios.ios_user:
        name: "{{ rotation_user }}"
        configured_password: "{{ new_password }}"
        password_type: secret
        update_password: always
        state: present
      no_log: true
      register: rotation_result
      # Password is pushed to running config but NOT saved yet
      # If verification fails, NOT saving means a reload restores old password

    - name: "Rotate | Verify new password works (close and reopen connection)"
      cisco.ios.ios_command:
        commands:
          - show version
      vars:
        # Override credentials to test with the new password
        ansible_password: "{{ new_password }}"
        ansible_become_pass: "{{ new_password }}"
      register: verify_result
      changed_when: false
      no_log: true
      # This task opens a new SSH connection using the new password
      # If the new password is wrong, this task fails
      # The play stops, 'write memory' never runs, reload = old password

    - name: "Rotate | Assert verification succeeded"
      ansible.builtin.assert:
        that:
          - verify_result is succeeded
          - "'Version' in verify_result.stdout[0]"
        fail_msg: |
          PASSWORD ROTATION VERIFICATION FAILED on {{ inventory_hostname }}
          New password did not authenticate successfully.
          Do NOT proceed to other devices.
          Recovery options:
            1. Reload the device (restores old password from startup-config)
            2. Connect via console with old password and 'write memory' to save new one
        success_msg: "PASS: New password verified on {{ inventory_hostname }}"

    - name: "Rotate | Save configuration (commits new password permanently)"
      cisco.ios.ios_command:
        commands: [write memory]
      # Only runs if verification passed
      # This is the point of no return — new password is now in startup-config

    - name: "Rotate | Update local known_hosts if SSH host key changed"
      ansible.builtin.known_hosts:
        name: "{{ ansible_host }}"
        state: present
        key: "{{ lookup('pipe', 'ssh-keyscan -t ed25519 ' + ansible_host) }}"
      delegate_to: localhost
      changed_when: false
      failed_when: false    # Non-critical — don't fail rotation if this errors

    - name: "Rotate | Log successful rotation"
      ansible.builtin.lineinfile:
        path: "logs/password_rotation.log"
        line: "{{ lookup('pipe', 'date') }} | {{ inventory_hostname }} | {{ rotation_user }} | SUCCESS"
        create: true
        mode: '0600'
      delegate_to: localhost

    - name: "Rotate | Display per-device result"
      ansible.builtin.debug:
        msg:
          - "✓ Password rotated successfully on {{ inventory_hostname }}"
          - "  User: {{ rotation_user }}"
          - "  Verification: PASSED"
          - "  Config saved: YES"

  rescue:
    - name: "Rotate | RESCUE | Log rotation failure"
      ansible.builtin.lineinfile:
        path: "logs/password_rotation.log"
        line: "{{ lookup('pipe', 'date') }} | {{ inventory_hostname }} | {{ rotation_user }} | FAILED — manual recovery required"
        create: true
        mode: '0600'
      delegate_to: localhost

    - name: "Rotate | RESCUE | Display recovery instructions"
      ansible.builtin.debug:
        msg:
          - "✗ ROTATION FAILED on {{ inventory_hostname }}"
          - "  The play has stopped. Other devices are NOT affected."
          - "  Recovery steps:"
          - "  1. Check console access to {{ inventory_hostname }}"
          - "  2. If device reloads, old password (from startup-config) is restored"
          - "  3. If new password was accepted: test login with new password"
          - "  4. Run: ansible-playbook rotate_passwords_ios.yml --limit {{ inventory_hostname }}"

    - name: "Rotate | RESCUE | Fail the play explicitly"
      ansible.builtin.fail:
        msg: "Password rotation failed on {{ inventory_hostname }}. See recovery instructions above."
EOF
```

### Running the Rotation Safely

```bash
# Step 1: Update the vault with the new password
ansible-vault edit inventory/group_vars/all/vault.yml
# Change: vault_ansible_device_password: "AnsibleService#NewPassword2025"
# Save and exit

# Step 2: Test against a single lab device first (never a production device)
ansible-playbook playbooks/security/rotate_passwords_ios.yml \
  --limit wan-r1 \
  --check    # Dry run first

# Step 3: Run on one lab device
ansible-playbook playbooks/security/rotate_passwords_ios.yml \
  --limit wan-r1

# Step 4: Verify you can SSH with the new password
ssh -i ~/.ssh/ansible_ed25519 ansible@172.16.0.11

# Step 5: Run on all lab devices (serial: 1 ensures one at a time)
ansible-playbook playbooks/security/rotate_passwords_ios.yml

# Step 6: Review the rotation log
cat logs/password_rotation.log
```

---

## 25.6 — Fleet-Wide User Audit Playbook

```bash
cat > ~/projects/ansible-network/playbooks/security/audit_users.yml << 'EOF'
---
# =============================================================
# audit_users.yml — Fleet-wide user account audit and report
# Safe: read-only, makes no changes, runs against all platforms
#
# Usage: ansible-playbook audit_users.yml
# Output: reports/security/user_audit_TIMESTAMP.txt
#         reports/security/user_audit_TIMESTAMP.json
# =============================================================

- name: "Audit | IOS user accounts"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:
    - name: "IOS | Gather user-related show commands"
      cisco.ios.ios_command:
        commands:
          - show running-config | section username
          - show users
          - show privilege
          - show ip ssh
      register: ios_user_data
      changed_when: false

    - name: "IOS | Parse configured usernames from running config"
      ansible.builtin.set_fact:
        ios_configured_users: >-
          {{ ios_user_data.stdout[0]
             | regex_findall('username (\S+)', multiline=True) }}
        ios_active_sessions: "{{ ios_user_data.stdout[1] }}"
        ios_current_privilege: >-
          {{ ios_user_data.stdout[2]
             | regex_search('Current privilege level is (\d+)', '\\1')
             | first | default('unknown') }}
        ios_ssh_version: >-
          {{ ios_user_data.stdout[3]
             | regex_search('SSH Enabled.*version (\S+)', '\\1')
             | default('disabled') }}

    - name: "IOS | Build per-device audit record"
      ansible.builtin.set_fact:
        _device_audit:
          device: "{{ inventory_hostname }}"
          platform: ios
          timestamp: "{{ lookup('pipe', 'date') }}"
          configured_users: "{{ ios_configured_users }}"
          user_count: "{{ ios_configured_users | length }}"
          active_sessions: "{{ ios_active_sessions }}"
          ansible_privilege: "{{ ios_current_privilege }}"
          ssh_version: "{{ ios_ssh_version }}"
          flags:
            has_ansible_account: "{{ 'ansible' in ios_configured_users }}"
            privilege_correct: "{{ ios_current_privilege == '15' }}"
            ssh_v2_only: "{{ '2' in ios_ssh_version }}"

    - name: "IOS | Store audit record for report generation"
      ansible.builtin.copy:
        content: "{{ _device_audit | to_json }}"
        dest: "/tmp/audit_{{ inventory_hostname }}.json"
        mode: '0600'
      delegate_to: localhost

- name: "Audit | NX-OS user accounts"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "NX-OS | Gather user-related show commands"
      cisco.nxos.nxos_command:
        commands:
          - show running-config | section username
          - show users
          - show role
      register: nxos_user_data
      changed_when: false

    - name: "NX-OS | Parse configured users"
      ansible.builtin.set_fact:
        nxos_configured_users: >-
          {{ nxos_user_data.stdout[0]
             | regex_findall('username (\S+)', multiline=True) }}
        nxos_roles_output: "{{ nxos_user_data.stdout[2] }}"

    - name: "NX-OS | Build per-device audit record"
      ansible.builtin.set_fact:
        _device_audit:
          device: "{{ inventory_hostname }}"
          platform: nxos
          timestamp: "{{ lookup('pipe', 'date') }}"
          configured_users: "{{ nxos_configured_users }}"
          user_count: "{{ nxos_configured_users | length }}"
          active_sessions: "{{ nxos_user_data.stdout[1] }}"
          flags:
            has_ansible_account: "{{ 'ansible' in nxos_configured_users }}"

    - name: "NX-OS | Store audit record"
      ansible.builtin.copy:
        content: "{{ _device_audit | to_json }}"
        dest: "/tmp/audit_{{ inventory_hostname }}.json"
        mode: '0600'
      delegate_to: localhost

- name: "Audit | Junos user accounts"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf

  tasks:
    - name: "Junos | Gather user information"
      junipernetworks.junos.junos_command:
        commands:
          - show system login
          - show system users
        display: text
      register: junos_user_data
      changed_when: false

    - name: "Junos | Parse configured users"
      ansible.builtin.set_fact:
        junos_configured_users: >-
          {{ junos_user_data.stdout[0]
             | regex_findall('user (\S+) {', multiline=True) }}

    - name: "Junos | Build per-device audit record"
      ansible.builtin.set_fact:
        _device_audit:
          device: "{{ inventory_hostname }}"
          platform: junos
          timestamp: "{{ lookup('pipe', 'date') }}"
          configured_users: "{{ junos_configured_users }}"
          user_count: "{{ junos_configured_users | length }}"
          active_sessions: "{{ junos_user_data.stdout[1] }}"
          flags:
            has_ansible_account: "{{ 'ansible' in junos_configured_users }}"

    - name: "Junos | Store audit record"
      ansible.builtin.copy:
        content: "{{ _device_audit | to_json }}"
        dest: "/tmp/audit_{{ inventory_hostname }}.json"
        mode: '0600'
      delegate_to: localhost

- name: "Audit | Generate consolidated report"
  hosts: localhost
  gather_facts: false

  vars:
    report_dir: "reports/security"
    report_timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Report | Create report directory"
      ansible.builtin.file:
        path: "{{ report_dir }}"
        state: directory
        mode: '0750'

    - name: "Report | Collect all device audit records"
      ansible.builtin.find:
        paths: /tmp
        patterns: "audit_*.json"
      register: audit_files

    - name: "Report | Load all audit records"
      ansible.builtin.set_fact:
        all_audit_records: >-
          {{ all_audit_records | default([]) +
             [lookup('file', item.path) | from_json] }}
      loop: "{{ audit_files.files }}"
      loop_control:
        label: "{{ item.path | basename }}"

    - name: "Report | Identify flags requiring attention"
      ansible.builtin.set_fact:
        flagged_devices: >-
          {{ all_audit_records | default([])
             | selectattr('flags.has_ansible_account', 'defined')
             | selectattr('flags.has_ansible_account', 'equalto', false)
             | list }}
        no_ssh_v2: >-
          {{ all_audit_records | default([])
             | selectattr('flags.ssh_v2_only', 'defined')
             | selectattr('flags.ssh_v2_only', 'equalto', false)
             | list }}
        wrong_privilege: >-
          {{ all_audit_records | default([])
             | selectattr('flags.privilege_correct', 'defined')
             | selectattr('flags.privilege_correct', 'equalto', false)
             | list }}

    - name: "Report | Write text report"
      ansible.builtin.copy:
        content: |
          Network Device User Audit Report
          ==================================
          Generated: {{ lookup('pipe', 'date') }}
          Devices audited: {{ all_audit_records | default([]) | length }}

          ── DEVICE SUMMARY ────────────────────────────────────────────
          {% for rec in all_audit_records | default([]) %}
          {{ '%-20s' | format(rec.device) }} [{{ rec.platform | upper }}]
            Users configured ({{ rec.user_count }}): {{ rec.configured_users | join(', ') }}
            Ansible account present: {{ 'YES' if rec.flags.has_ansible_account else 'NO ⚠' }}
          {% if rec.flags.ssh_v2_only is defined %}
            SSH v2 only: {{ 'YES' if rec.flags.ssh_v2_only else 'NO ⚠' }}
          {% endif %}
          {% if rec.flags.privilege_correct is defined %}
            Ansible privilege=15: {{ 'YES' if rec.flags.privilege_correct else 'NO ⚠' }}
          {% endif %}

          {% endfor %}

          ── FLAGS REQUIRING ATTENTION ─────────────────────────────────
          {% if flagged_devices | length > 0 %}
          ⚠ MISSING ansible ACCOUNT:
          {% for dev in flagged_devices %}
            - {{ dev.device }} ({{ dev.platform }})
          {% endfor %}
          {% else %}
          ✓ ansible account present on all devices
          {% endif %}

          {% if no_ssh_v2 | length > 0 %}
          ⚠ SSH v2 NOT ENFORCED:
          {% for dev in no_ssh_v2 %}
            - {{ dev.device }}
          {% endfor %}
          {% else %}
          ✓ SSH v2 enforced on all IOS devices
          {% endif %}

          {% if wrong_privilege | length > 0 %}
          ⚠ ANSIBLE NOT RUNNING AT PRIVILEGE 15:
          {% for dev in wrong_privilege %}
            - {{ dev.device }}
          {% endfor %}
          {% else %}
          ✓ Ansible running at correct privilege level on all IOS devices
          {% endif %}

          ── ACTIVE SESSIONS ───────────────────────────────────────────
          {% for rec in all_audit_records | default([]) %}
          {{ rec.device }}:
          {{ rec.active_sessions | indent(2, true) }}
          {% endfor %}
        dest: "{{ report_dir }}/user_audit_{{ report_timestamp }}.txt"
        mode: '0640'

    - name: "Report | Write JSON report"
      ansible.builtin.copy:
        content: "{{ all_audit_records | default([]) | to_nice_json }}"
        dest: "{{ report_dir }}/user_audit_{{ report_timestamp }}.json"
        mode: '0640'

    - name: "Report | Display summary to console"
      ansible.builtin.debug:
        msg:
          - "═══════════════════════════════════════════════"
          - " USER AUDIT COMPLETE"
          - " Devices audited: {{ all_audit_records | default([]) | length }}"
          - " Flags: {{ (flagged_devices | length) + (no_ssh_v2 | length) + (wrong_privilege | length) }} item(s) need attention"
          - " Report: {{ report_dir }}/user_audit_{{ report_timestamp }}.txt"
          - "═══════════════════════════════════════════════"

    - name: "Report | Clean up temp files"
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ audit_files.files }}"
      loop_control:
        label: "{{ item.path | basename }}"
EOF
```

---


User and AAA automation is complete — IOS privilege levels with full TACACS+ AAA and local fallback, NX-OS role-based access with management VRF routing, rolling password rotation with per-device verification and rollback, and a cross-platform audit report that flags security gaps fleet-wide. Part 26 extends the guide into network-wide reporting: using Ansible to collect facts from every device in the lab and generate a complete topology and inventory document.

