---
draft: false
title: '24 - Uninstall'
description: "Part 24 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 24
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 24: Writing an Uninstall Playbook from an Existing Install Playbook

> *Every playbook written so far installs, configures, or deploys something. None of them clean up after themselves. This is a gap — not just an aesthetic one. Automation that can't be reversed creates technical debt: lab environments that can't be reset cleanly, device configs that accumulate stale entries, control nodes that can't be reprovisioned from scratch. The discipline of reversible automation is simple: if you wrote the task to create something, write the task to remove it. This part builds that discipline into the project.*

---

## 24.1 — The Philosophy of Reversible Automation

A task that creates a resource has three states from Ansible's perspective:

```
present  → resource exists (create it if missing)
absent   → resource does not exist (remove it if present)
```

Most Ansible modules support both. The uninstall playbook is not a new playbook written from scratch — it's the install playbook read in reverse, with `state: present` replaced by `state: absent` and destructive actions guarded appropriately.

### What Makes a Task Reversible

Not every task is trivially reversible. Before writing the uninstall counterpart, each install task falls into one of four categories:

```
Category 1: Directly reversible — state: absent
  Examples: apt package, pip package, user account, file, cron job
  Pattern:  same module, same parameters, state: absent

Category 2: Config-reversible — remove or negate
  Examples: interface IP address, BGP neighbor, NTP server
  Pattern:  resource module with state: deleted, or ios_config with 'no' commands

Category 3: Restore-reversible — replace with previous value
  Examples: hostname, banner, motd
  Pattern:  restore a known default or blank value

Category 4: Not reversible without side effects
  Examples: 'clear ip bgp *', crypto key deletion, interface shutdown on live link
  Pattern:  gate behind a confirmation variable, document the risk explicitly
```

Identifying which category each install task falls into before writing the uninstall counterpart is the first step. Skipping this analysis leads to uninstall playbooks that silently leave things behind.

---

## 24.2 — The Single-File Tag Pattern

The primary pattern is a single playbook with two mutually exclusive tag sets: `install` and `uninstall`. Running with `--tags install` deploys. Running with `--tags uninstall` removes. Every task belongs to exactly one tag. No task belongs to both.

```
ansible-playbook site.yml --tags install      → full deployment
ansible-playbook site.yml --tags uninstall    → full removal
ansible-playbook site.yml --tags install,ntp  → only NTP install tasks
ansible-playbook site.yml --tags uninstall,ntp → only NTP removal tasks
```

The tag hierarchy mirrors the functional sections of the playbook:

```
tags:
  install tasks:   [install, <section>]    e.g. [install, ntp]
  uninstall tasks: [uninstall, <section>]  e.g. [uninstall, ntp]
```

This gives precise control: `--tags install,ntp` runs only the NTP install tasks, `--tags uninstall` runs all removal tasks regardless of section.

---

## 24.3 — Example 1: Control Node Setup (Simple Pattern)

The control node setup playbook from Part 2 installed: apt packages, Python pip packages, Ansible collections, configuration files, and a service account. Here it is restructured as a single install/uninstall playbook.

```bash
cat > ~/projects/ansible-network/playbooks/setup/control_node.yml << 'EOF'
---
# =============================================================
# control_node.yml — Ansible control node setup and teardown
#
# INSTALL:   ansible-playbook control_node.yml --tags install
# UNINSTALL: ansible-playbook control_node.yml --tags uninstall
#
# Section tags (combine with install/uninstall):
#   packages    — apt system packages
#   venv        — Python virtualenv and pip packages
#   collections — Ansible collections
#   config      — ansible.cfg and project config files
#   user        — ansible service account
#
# Examples:
#   Install only packages:         --tags install,packages
#   Remove only the service user:  --tags uninstall,user
#   Remove everything:             --tags uninstall
#
# UNINSTALL SAFETY:
#   The uninstall play removes the virtualenv, collections, and
#   service account. It does NOT remove apt packages that may be
#   used by other software (python3, git, openssh-client).
#   Set remove_shared_packages: true to override this safety.
#
# Reversibility notes per section:
#   packages:    apt state: absent — safe, reversible
#   venv:        rm -rf the venv directory — reversible (rebuild from requirements.txt)
#   collections: rm -rf collection path — reversible (reinstall from requirements.yml)
#   config:      file state: absent — reversible (recreated by install)
#   user:        user state: absent — reversible, but HOME DIR IS DELETED
# =============================================================

- name: "Control node | Setup and teardown"
  hosts: localhost
  gather_facts: true
  become: true

  vars:
    # ── Paths ──────────────────────────────────────────────────
    venv_path: "/home/{{ ansible_user_id }}/ansible-venv"
    project_path: "/home/{{ ansible_user_id }}/projects/ansible-network"
    collections_path: "/home/{{ ansible_user_id }}/.ansible/collections"

    # ── Safety gates ───────────────────────────────────────────
    remove_shared_packages: false    # Set true to also remove python3, git
    remove_project_files: false      # Set true to also remove the project directory
    # Override on CLI: -e "remove_shared_packages=true"

    # ── Package lists ──────────────────────────────────────────
    # Packages exclusive to Ansible use — safe to remove
    ansible_exclusive_packages:
      - python3-venv
      - python3-pip
      - sshpass

    # Packages shared with other software — only remove if explicitly requested
    shared_packages:
      - python3
      - git
      - openssh-client
      - curl

    # ── Service account ────────────────────────────────────────
    service_user: ansible-svc
    service_user_home: "/home/{{ service_user }}"

  tasks:

    # ══════════════════════════════════════════════════════════
    # SECTION: System packages
    # ══════════════════════════════════════════════════════════

    - name: "Packages | INSTALL | Update apt cache"
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      tags: [install, packages]

    - name: "Packages | INSTALL | Install Ansible-exclusive apt packages"
      ansible.builtin.apt:
        name: "{{ ansible_exclusive_packages }}"
        state: present
      tags: [install, packages]
      # Reversal: state: absent in the uninstall task below

    - name: "Packages | INSTALL | Install shared system packages"
      ansible.builtin.apt:
        name: "{{ shared_packages }}"
        state: present
      tags: [install, packages]
      # Reversal: guarded — only removed if remove_shared_packages: true

    - name: "Packages | UNINSTALL | Remove Ansible-exclusive apt packages"
      ansible.builtin.apt:
        name: "{{ ansible_exclusive_packages }}"
        state: absent
        autoremove: true
        purge: true
      tags: [uninstall, packages]
      # Direct reversal of the install task above
      # purge: true also removes config files left by the package

    - name: "Packages | UNINSTALL | Remove shared packages (if remove_shared_packages)"
      ansible.builtin.apt:
        name: "{{ shared_packages }}"
        state: absent
        autoremove: true
      tags: [uninstall, packages]
      when: remove_shared_packages | bool
      # Guarded: python3 and git may be needed by other software
      # Only runs when explicitly requested via -e "remove_shared_packages=true"

    # ══════════════════════════════════════════════════════════
    # SECTION: Python virtualenv
    # ══════════════════════════════════════════════════════════

    - name: "Venv | INSTALL | Create virtualenv"
      ansible.builtin.pip:
        virtualenv: "{{ venv_path }}"
        virtualenv_command: python3 -m venv
        name: pip
        state: latest
      become_user: "{{ ansible_user_id }}"
      tags: [install, venv]

    - name: "Venv | INSTALL | Install pip packages from requirements.txt"
      ansible.builtin.pip:
        requirements: "{{ project_path }}/requirements.txt"
        virtualenv: "{{ venv_path }}"
        state: present
      become_user: "{{ ansible_user_id }}"
      tags: [install, venv]

    - name: "Venv | UNINSTALL | Remove virtualenv directory"
      ansible.builtin.file:
        path: "{{ venv_path }}"
        state: absent
      tags: [uninstall, venv]
      # Removes entire venv including all pip packages
      # Reversal: re-run install tasks to rebuild from requirements.txt
      # The requirements.txt file is NOT deleted here — it's the source of truth

    # ══════════════════════════════════════════════════════════
    # SECTION: Ansible collections
    # ══════════════════════════════════════════════════════════

    - name: "Collections | INSTALL | Install collections from requirements.yml"
      ansible.builtin.command:
        cmd: >
          {{ venv_path }}/bin/ansible-galaxy collection install
          -r {{ project_path }}/collections/requirements.yml
          --force
      become_user: "{{ ansible_user_id }}"
      register: collection_install
      changed_when: "'Installing' in collection_install.stdout"
      tags: [install, collections]

    - name: "Collections | UNINSTALL | Remove installed collections directory"
      ansible.builtin.file:
        path: "{{ collections_path }}"
        state: absent
      become_user: "{{ ansible_user_id }}"
      tags: [uninstall, collections]
      # Removes ~/.ansible/collections entirely
      # Reversal: re-run install tasks — collections reinstalled from requirements.yml

    # ══════════════════════════════════════════════════════════
    # SECTION: Configuration files
    # ══════════════════════════════════════════════════════════

    - name: "Config | INSTALL | Deploy ansible.cfg"
      ansible.builtin.template:
        src: templates/ansible.cfg.j2
        dest: "{{ project_path }}/ansible.cfg"
        owner: "{{ ansible_user_id }}"
        mode: '0644'
      become_user: "{{ ansible_user_id }}"
      tags: [install, config]

    - name: "Config | INSTALL | Deploy .vault/lab.txt password file placeholder"
      ansible.builtin.file:
        path: "{{ project_path }}/.vault"
        state: directory
        owner: "{{ ansible_user_id }}"
        mode: '0700'
      become_user: "{{ ansible_user_id }}"
      tags: [install, config]

    - name: "Config | UNINSTALL | Remove ansible.cfg"
      ansible.builtin.file:
        path: "{{ project_path }}/ansible.cfg"
        state: absent
      tags: [uninstall, config]
      # Reversal: re-running install recreates from template
      # Note: the template source file is NOT removed

    - name: "Config | UNINSTALL | Remove vault directory"
      ansible.builtin.file:
        path: "{{ project_path }}/.vault"
        state: absent
      tags: [uninstall, config]

    - name: "Config | UNINSTALL | Remove project directory (if remove_project_files)"
      ansible.builtin.file:
        path: "{{ project_path }}"
        state: absent
      tags: [uninstall, config]
      when: remove_project_files | bool
      # DESTRUCTIVE: deletes all playbooks, inventory, templates, host_vars
      # Only runs when explicitly requested via -e "remove_project_files=true"

    # ══════════════════════════════════════════════════════════
    # SECTION: Service account
    # ══════════════════════════════════════════════════════════

    - name: "User | INSTALL | Create ansible service account"
      ansible.builtin.user:
        name: "{{ service_user }}"
        comment: "Ansible automation service account"
        shell: /bin/bash
        create_home: true
        state: present
      tags: [install, user]

    - name: "User | INSTALL | Configure sudoers for service account"
      ansible.builtin.copy:
        content: "{{ service_user }} ALL=(ALL) NOPASSWD: ALL\n"
        dest: "/etc/sudoers.d/{{ service_user }}"
        owner: root
        group: root
        mode: '0440'
        validate: visudo -cf %s
      tags: [install, user]

    - name: "User | INSTALL | Deploy SSH authorized key"
      ansible.posix.authorized_key:
        user: "{{ service_user }}"
        state: present
        key: "{{ lookup('file', '~/.ssh/ansible_ed25519.pub') }}"
      tags: [install, user]

    - name: "User | UNINSTALL | Remove sudoers file"
      ansible.builtin.file:
        path: "/etc/sudoers.d/{{ service_user }}"
        state: absent
      tags: [uninstall, user]
      # Always remove sudoers BEFORE removing the user account
      # Avoids orphaned sudoers entry that could cause visudo warnings

    - name: "User | UNINSTALL | Remove service account and home directory"
      ansible.builtin.user:
        name: "{{ service_user }}"
        state: absent
        remove: true    # remove: true deletes the home directory
        force: true     # force: true kills running processes owned by user
      tags: [uninstall, user]
      # DESTRUCTIVE: home directory is permanently deleted
      # Documented in the header block above

    # ══════════════════════════════════════════════════════════
    # POST-INSTALL VERIFICATION (install tags only)
    # ══════════════════════════════════════════════════════════

    - name: "Verify | INSTALL | Confirm virtualenv is functional"
      ansible.builtin.command:
        cmd: "{{ venv_path }}/bin/python3 -c \"import ansible; print(ansible.__version__)\""
      become_user: "{{ ansible_user_id }}"
      register: venv_check
      changed_when: false
      tags: [install, verify]

    - name: "Verify | INSTALL | Display installed ansible version"
      ansible.builtin.debug:
        msg: "Ansible {{ venv_check.stdout }} installed in {{ venv_path }}"
      tags: [install, verify]

    # ══════════════════════════════════════════════════════════
    # POST-UNINSTALL VERIFICATION (uninstall tags only)
    # ══════════════════════════════════════════════════════════

    - name: "Verify | UNINSTALL | Confirm virtualenv is gone"
      ansible.builtin.stat:
        path: "{{ venv_path }}"
      register: venv_stat
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Assert venv removed"
      ansible.builtin.assert:
        that:
          - not venv_stat.stat.exists
        fail_msg: "Virtualenv still exists at {{ venv_path }}"
        success_msg: "PASS: Virtualenv removed"
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Confirm service account is gone"
      ansible.builtin.getent:
        database: passwd
        key: "{{ service_user }}"
        fail_key: false
      register: user_check
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Assert service account removed"
      ansible.builtin.assert:
        that:
          - service_user not in (user_check.ansible_facts.getent_passwd | default({}))
        fail_msg: "Service account {{ service_user }} still exists"
        success_msg: "PASS: Service account removed"
      tags: [uninstall, verify]
EOF
```

---

## 24.4 — Example 2: Network Device Configuration (Complex Pattern)

The network device case is more nuanced. On a router, "uninstall" doesn't mean wiping the device — it means removing the specific configuration pushed by the automation and returning those elements to a known default or absent state. The install playbook from Parts 17-18 pushed NTP, ACLs, users, OSPF, and more. Here's the paired uninstall logic.

```bash
cat > ~/projects/ansible-network/playbooks/deploy/deploy_ios_ccna_reversible.yml << 'EOF'
---
# =============================================================
# deploy_ios_ccna_reversible.yml — IOS CCNA config with uninstall
#
# INSTALL:   ansible-playbook deploy_ios_ccna_reversible.yml --tags install
# UNINSTALL: ansible-playbook deploy_ios_ccna_reversible.yml --tags uninstall
#
# Section tags (always combine with install or uninstall):
#   identity    — hostname, banner
#   interfaces  — L3 interface IPs
#   ntp         — NTP servers
#   users       — local user accounts
#   acl         — access control lists
#   ospf        — OSPF process and area assignments
#   bgp         — BGP process and neighbors
#   snmp        — SNMP community and settings
#
# REVERSIBILITY NOTES:
#
#   identity (hostname):
#     INSTALL:   sets hostname from data model
#     UNINSTALL: restores to 'Router' (IOS default)
#     Category:  Restore-reversible
#
#   identity (banner):
#     INSTALL:   sets login banner
#     UNINSTALL: removes banner (ios_banner state: absent)
#     Category:  Directly reversible
#
#   interfaces:
#     INSTALL:   sets IP address on each interface
#     UNINSTALL: removes IP (no ip address) and re-adds shutdown
#     Category:  Config-reversible
#     WARNING:   Removing IP from active interface disrupts traffic
#
#   ntp:
#     INSTALL:   adds NTP servers
#     UNINSTALL: removes NTP servers (ios_ntp_global state: deleted)
#     Category:  Directly reversible
#
#   users:
#     INSTALL:   creates local user accounts
#     UNINSTALL: removes local user accounts (ios_user state: absent)
#     Category:  Directly reversible
#     WARNING:   Removing all users while logged in as one of them
#                will not remove your own active session's user until
#                you log out. Always keep one emergency access method.
#
#   acl:
#     INSTALL:   creates named ACLs and applies to interfaces
#     UNINSTALL: removes ACL application then deletes ACLs
#     Category:  Config-reversible (must unapply before delete)
#     ORDER MATTERS: always remove interface application before deleting ACL
#
#   ospf:
#     INSTALL:   creates OSPF process and assigns interfaces
#     UNINSTALL: deletes OSPF process (ios_ospfv2 state: deleted)
#     Category:  Directly reversible
#     WARNING:   Removing OSPF drops all OSPF-learned routes immediately
#
#   bgp:
#     INSTALL:   creates BGP process and neighbors
#     UNINSTALL: deletes BGP process (ios_bgp_global state: deleted)
#     Category:  Directly reversible
#     WARNING:   Removing BGP drops all BGP-learned routes immediately
#
#   snmp:
#     INSTALL:   sets SNMP community and strings
#     UNINSTALL: removes via ios_config 'no snmp-server' commands
#     Category:  Config-reversible
# =============================================================

- name: "IOS CCNA | Reversible configuration deploy and remove"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true

  tasks:

    # ── Pre-flight (always runs) ───────────────────────────────
    - name: "Pre-flight | Gather IOS facts"
      cisco.ios.ios_facts:
        gather_subset: [default]
      tags: always

    - name: "Pre-flight | UNINSTALL | Warn about traffic impact"
      ansible.builtin.pause:
        prompt: |
          ⚠  UNINSTALL WARNING for {{ inventory_hostname }}
          This will remove:
            - All configured IP addresses on managed interfaces
            - OSPF process {{ ospf.process_id | default('N/A') }}
            - BGP AS {{ bgp.as_number | default('N/A') }}
            - All local users except the current session user
            - All ACLs deployed by this playbook

          Routing and management access WILL be disrupted.
          Ensure console/OOB access is available.

          Press ENTER to continue or Ctrl+C then A to abort
      tags: [uninstall]
      when: not ansible_check_mode

    # ══════════════════════════════════════════════════════════
    # SECTION: Device identity
    # ══════════════════════════════════════════════════════════

    - name: "Identity | INSTALL | Set hostname"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      tags: [install, identity]
      notify: Save IOS configuration

    - name: "Identity | INSTALL | Set login banner"
      cisco.ios.ios_banner:
        banner: login
        text: |
          **************************************************************************
          *  Authorized access only. All activity is monitored and logged.         *
          **************************************************************************
        state: present
      tags: [install, identity]
      notify: Save IOS configuration

    - name: "Identity | UNINSTALL | Restore default hostname"
      cisco.ios.ios_hostname:
        config:
          hostname: Router    # IOS factory default hostname
        state: merged
      tags: [uninstall, identity]
      notify: Save IOS configuration
      # Restore-reversible: 'Router' is the IOS default
      # Alternative: set to inventory_hostname if a different default is expected

    - name: "Identity | UNINSTALL | Remove login banner"
      cisco.ios.ios_banner:
        banner: login
        state: absent
      tags: [uninstall, identity]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: Interfaces
    # ══════════════════════════════════════════════════════════

    - name: "Interfaces | INSTALL | Configure interface descriptions and state"
      cisco.ios.ios_interfaces:
        config:
          - name: "{{ item.key }}"
            description: "{{ item.value.description | default('') }}"
            enabled: "{{ not item.value.shutdown | default(false) }}"
        state: merged
      loop: "{{ interfaces | dict2items }}"
      loop_control:
        label: "{{ item.key }}"
      tags: [install, interfaces]
      notify: Save IOS configuration

    - name: "Interfaces | INSTALL | Configure interface IP addresses"
      cisco.ios.ios_l3_interfaces:
        config:
          - name: "{{ item.key }}"
            ipv4:
              - address: "{{ item.value.ip }}/{{ item.value.prefix }}"
        state: merged
      loop: "{{ interfaces | dict2items | selectattr('value.ip', 'defined') | list }}"
      loop_control:
        label: "{{ item.key }} — {{ item.value.ip }}/{{ item.value.prefix }}"
      tags: [install, interfaces]
      notify: Save IOS configuration

    - name: "Interfaces | UNINSTALL | Remove interface IP addresses"
      cisco.ios.ios_l3_interfaces:
        config:
          - name: "{{ item.key }}"
            ipv4: []    # Empty list = remove all IPv4 addresses
        state: replaced
      loop: "{{ interfaces | dict2items | selectattr('value.ip', 'defined') | list }}"
      loop_control:
        label: "{{ item.key }}"
      tags: [uninstall, interfaces]
      notify: Save IOS configuration
      # Config-reversible: state: replaced with empty ipv4 list removes addresses

    - name: "Interfaces | UNINSTALL | Shutdown managed interfaces"
      cisco.ios.ios_interfaces:
        config:
          - name: "{{ item.key }}"
            description: ""
            enabled: false    # shutdown
        state: merged
      loop: "{{ interfaces | dict2items }}"
      loop_control:
        label: "{{ item.key }}"
      tags: [uninstall, interfaces]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: NTP
    # ══════════════════════════════════════════════════════════

    - name: "NTP | INSTALL | Configure NTP servers"
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
              prefer: "{{ true if loop.index == 1 else false }}"
        state: merged
      loop: "{{ ntp_servers | default([]) }}"
      loop_control:
        label: "NTP: {{ item }}"
        index_var: loop
      tags: [install, ntp]
      notify: Save IOS configuration

    - name: "NTP | UNINSTALL | Remove all NTP servers"
      cisco.ios.ios_ntp_global:
        state: deleted
      tags: [uninstall, ntp]
      notify: Save IOS configuration
      # Directly reversible: state: deleted removes all NTP configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: Local users
    # NOTE: ORDER MATTERS on uninstall — see warning in header
    # ══════════════════════════════════════════════════════════

    - name: "Users | INSTALL | Configure local user accounts"
      cisco.ios.ios_users:
        config:
          - name: "{{ item.username }}"
            privilege: "{{ item.privilege | default(15) }}"
            configured_password: "{{ item.password }}"
            password_type: secret
        state: merged
      loop: "{{ local_users | default([]) }}"
      loop_control:
        label: "User: {{ item.username }}"
      no_log: true
      tags: [install, users]
      notify: Save IOS configuration

    - name: "Users | UNINSTALL | Remove automation-managed user accounts"
      cisco.ios.ios_users:
        config:
          - name: "{{ item.username }}"
        state: deleted
      loop: >-
        {{ local_users | default([])
           | rejectattr('username', 'equalto', ansible_user) | list }}
      loop_control:
        label: "Remove user: {{ item.username }}"
      tags: [uninstall, users]
      notify: Save IOS configuration
      # rejectattr filter: never remove the user Ansible is currently logged in as
      # This prevents locking yourself out during uninstall

    # ══════════════════════════════════════════════════════════
    # SECTION: ACLs
    # Uninstall order: unapply from interfaces FIRST, then delete
    # ══════════════════════════════════════════════════════════

    - name: "ACL | INSTALL | Create ACLs"
      cisco.ios.ios_acls:
        config: "{{ acls | default([]) }}"
        state: merged
      tags: [install, acl]
      notify: Save IOS configuration

    - name: "ACL | INSTALL | Apply ACLs to interfaces"
      cisco.ios.ios_config:
        parents: "interface {{ item.interface }}"
        lines:
          - "ip access-group {{ item.acl_name }} {{ item.direction }}"
      loop: "{{ acl_applications | default([]) }}"
      loop_control:
        label: "{{ item.acl_name }} {{ item.direction }} on {{ item.interface }}"
      tags: [install, acl]
      notify: Save IOS configuration

    - name: "ACL | UNINSTALL | Remove ACL from interfaces FIRST"
      cisco.ios.ios_config:
        parents: "interface {{ item.interface }}"
        lines:
          - "no ip access-group {{ item.acl_name }} {{ item.direction }}"
      loop: "{{ acl_applications | default([]) }}"
      loop_control:
        label: "Remove {{ item.acl_name }} from {{ item.interface }}"
      tags: [uninstall, acl]
      notify: Save IOS configuration
      # ORDER-CRITICAL: must unapply from interface before deleting ACL
      # IOS allows deleting an applied ACL but it leaves a broken reference

    - name: "ACL | UNINSTALL | Delete ACLs"
      cisco.ios.ios_acls:
        config: "{{ acls | default([]) }}"
        state: deleted
      tags: [uninstall, acl]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: OSPF
    # ══════════════════════════════════════════════════════════

    - name: "OSPF | INSTALL | Configure OSPF process"
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf.process_id }}"
              router_id: "{{ ospf.router_id | default(omit) }}"
        state: merged
      when: ospf is defined
      tags: [install, ospf]
      notify: Save IOS configuration

    - name: "OSPF | INSTALL | Assign interfaces to OSPF areas"
      cisco.ios.ios_config:
        parents: "interface {{ item.0.name }}"
        lines:
          - "ip ospf {{ ospf.process_id }} area {{ item.1.area_id }}"
      loop: "{{ ospf.areas | subelements('interfaces') | map('reverse') | list }}"
      loop_control:
        label: "{{ item.1.name }} → area {{ item.0.area_id }}"
      when: ospf is defined
      tags: [install, ospf]
      notify: Save IOS configuration

    - name: "OSPF | UNINSTALL | Remove OSPF interface assignments"
      cisco.ios.ios_config:
        parents: "interface {{ item.0.name }}"
        lines:
          - "no ip ospf {{ ospf.process_id }} area {{ item.1.area_id }}"
      loop: "{{ ospf.areas | subelements('interfaces') | map('reverse') | list }}"
      loop_control:
        label: "{{ item.1.name }}"
      when: ospf is defined
      tags: [uninstall, ospf]
      notify: Save IOS configuration
      # Remove interface-level OSPF config before deleting the process

    - name: "OSPF | UNINSTALL | Delete OSPF process"
      cisco.ios.ios_ospfv2:
        config:
          processes:
            - process_id: "{{ ospf.process_id }}"
        state: deleted
      when: ospf is defined
      tags: [uninstall, ospf]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # SECTION: BGP
    # ══════════════════════════════════════════════════════════

    - name: "BGP | INSTALL | Configure BGP process and neighbors"
      cisco.ios.ios_bgp_global:
        config:
          as_number: "{{ bgp.as_number }}"
          router_id: "{{ bgp.router_id | default(omit) }}"
          neighbor:
            - neighbor_address: "{{ item.ip }}"
              remote_as: "{{ item.remote_as }}"
              description: "{{ item.description | default(omit) }}"
        state: merged
      loop: "{{ bgp.neighbors | default([]) }}"
      loop_control:
        label: "BGP neighbor {{ item.ip }}"
      when: bgp is defined
      tags: [install, bgp]
      notify: Save IOS configuration

    - name: "BGP | UNINSTALL | Delete entire BGP process"
      cisco.ios.ios_bgp_global:
        config:
          as_number: "{{ bgp.as_number }}"
        state: deleted
      when: bgp is defined
      tags: [uninstall, bgp]
      notify: Save IOS configuration
      # state: deleted removes the entire BGP process including all neighbors
      # WARNING: all BGP-learned routes drop immediately

    # ══════════════════════════════════════════════════════════
    # SECTION: SNMP
    # ══════════════════════════════════════════════════════════

    - name: "SNMP | INSTALL | Configure SNMP"
      cisco.ios.ios_config:
        lines:
          - "snmp-server community {{ snmp_community_ro | default('public') }} RO"
          - "snmp-server location {{ snmp_location | default('Unknown') }}"
          - "snmp-server contact {{ snmp_contact | default('netops@lab.local') }}"
      no_log: true
      tags: [install, snmp]
      notify: Save IOS configuration

    - name: "SNMP | UNINSTALL | Remove all SNMP configuration"
      cisco.ios.ios_config:
        lines:
          - "no snmp-server"    # Removes entire SNMP config in one command
      tags: [uninstall, snmp]
      notify: Save IOS configuration

    # ══════════════════════════════════════════════════════════
    # POST-UNINSTALL VERIFICATION
    # ══════════════════════════════════════════════════════════

    - name: "Verify | UNINSTALL | Collect running config for review"
      cisco.ios.ios_command:
        commands: [show running-config]
      register: post_uninstall_config
      changed_when: false
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Assert OSPF process is gone"
      ansible.builtin.assert:
        that:
          - "'router ospf' not in post_uninstall_config.stdout[0]"
        fail_msg: "OSPF process still present in running config"
        success_msg: "PASS: OSPF process removed"
      when: ospf is defined
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Assert BGP process is gone"
      ansible.builtin.assert:
        that:
          - "'router bgp' not in post_uninstall_config.stdout[0]"
        fail_msg: "BGP process still present in running config"
        success_msg: "PASS: BGP process removed"
      when: bgp is defined
      tags: [uninstall, verify]

    - name: "Verify | UNINSTALL | Assert managed ACLs are gone"
      ansible.builtin.assert:
        that:
          - "item.name not in post_uninstall_config.stdout[0]"
        fail_msg: "ACL '{{ item.name }}' still present in running config"
        success_msg: "PASS: ACL '{{ item.name }}' removed"
      loop: "{{ acls | default([]) }}"
      loop_control:
        label: "{{ item.name }}"
      tags: [uninstall, verify]

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration
EOF
```

---

## 24.5 — Containerlab Test Workflow

The full cycle: deploy → uninstall → verify clean state → re-deploy → confirm idempotency.

```bash
# ── Step 1: Deploy to Containerlab ────────────────────────────────
echo "═══ Step 1: Full installation ═══"
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags install \
  --limit wan-r1

# Confirm deployment succeeded
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config | include router ospf|router bgp|ntp server'"

# ── Step 2: Capture baseline state before uninstall ───────────────
echo "═══ Step 2: Capture pre-uninstall state ═══"
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config'" \
  | grep -A 5 "router ospf" > /tmp/pre_uninstall_ospf.txt
echo "Pre-uninstall OSPF config saved to /tmp/pre_uninstall_ospf.txt"

# ── Step 3: Dry-run the uninstall first ───────────────────────────
echo "═══ Step 3: Dry-run uninstall (--check) ═══"
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags uninstall \
  --limit wan-r1 \
  --check

# ── Step 4: Run the actual uninstall ──────────────────────────────
echo "═══ Step 4: Execute uninstall ═══"
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags uninstall \
  --limit wan-r1

# ── Step 5: Verify clean state ────────────────────────────────────
echo "═══ Step 5: Verify clean state ═══"

# Check OSPF is gone
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config | include router ospf'" \
  | grep -q "router ospf" && echo "FAIL: OSPF still present" || echo "PASS: OSPF removed"

# Check BGP is gone
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config | include router bgp'" \
  | grep -q "router bgp" && echo "FAIL: BGP still present" || echo "PASS: BGP removed"

# Check NTP is gone
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config | include ntp server'" \
  | grep -q "ntp server" && echo "FAIL: NTP still present" || echo "PASS: NTP removed"

# Check hostname reset
ansible wan-r1 -m cisco.ios.ios_command \
  -a "commands='show running-config | include hostname'" \
  | grep -q "hostname Router" && echo "PASS: Hostname reset to Router" \
  || echo "FAIL: Hostname not reset"

# ── Step 6: Re-deploy and verify idempotency ──────────────────────
echo "═══ Step 6: Re-deploy to confirm idempotency ═══"
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags install \
  --limit wan-r1

# Run again — should show all 'ok', no 'changed'
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags install \
  --limit wan-r1

# Count changed tasks in second run (should be 0)
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags install \
  --limit wan-r1 2>&1 \
  | grep -E "^(ok|changed|failed|skipped)=" \
  | grep "changed=" | grep -v "changed=0" \
  && echo "FAIL: Idempotency broken — tasks still showing changed" \
  || echo "PASS: Play is idempotent"

# ── Step 7: Full cycle in Containerlab (destroy and redeploy) ─────
echo "═══ Step 7: Full Containerlab reset ═══"

# Destroy the lab entirely (resets all device state to factory)
cd ~/projects/containerlab
sudo containerlab destroy -t topology.yml

# Redeploy the lab from scratch
sudo containerlab deploy -t topology.yml

# Wait for devices to come up
sleep 30

# Run full deployment — must work on a clean device
ansible-playbook playbooks/deploy/deploy_ios_ccna_reversible.yml \
  --tags install

echo "═══ Full cycle complete ═══"
```

### What to Check at Each Stage

```
After --tags install:
  ✓ OSPF process exists with correct router-id
  ✓ BGP process exists with correct neighbors
  ✓ NTP servers configured
  ✓ Local users created
  ✓ ACLs created and applied to interfaces
  ✓ Hostname set correctly

After --tags uninstall:
  ✓ 'router ospf' not in running-config
  ✓ 'router bgp' not in running-config
  ✓ 'ntp server' not in running-config
  ✓ Managed user accounts not in running-config
  ✓ ACLs not in running-config
  ✓ ACL not applied to interfaces (no 'ip access-group')
  ✓ Hostname shows 'Router' (factory default)
  ✓ Managed interfaces show 'shutdown' and no IP address

After re-deploy (second --tags install):
  ✓ All above install checks pass again
  ✓ Second run shows changed=0 (idempotency confirmed)
```

---

## 24.6 — Documentation Pattern

### The Header Block

Every reversible playbook gets a complete header block documenting both modes, all section tags, and the reversibility category of each section:

```yaml
# =============================================================
# playbook_name.yml — [One-line description]
#
# INSTALL:   ansible-playbook playbook_name.yml --tags install
# UNINSTALL: ansible-playbook playbook_name.yml --tags uninstall
#
# Section tags (combine with install or uninstall):
#   [section]  — [description]
#   [section]  — [description]
#
# Safety variables:
#   [var]=true — [description of what it unlocks]
#
# REVERSIBILITY NOTES:
#   [section] ([Category]):
#     INSTALL:   [what install does]
#     UNINSTALL: [what uninstall does]
#     WARNING:   [any side effects or risks]
# =============================================================
```

### Per-Task Inline Comments

Each uninstall task carries a comment explaining its reversal logic and any ordering dependencies:

```yaml
- name: "ACL | UNINSTALL | Remove ACL from interfaces FIRST"
  cisco.ios.ios_config:
    ...
  # ORDER-CRITICAL: must unapply from interface before deleting ACL
  # IOS allows deleting an applied ACL but leaves a broken reference
  # that causes log noise and confuses show ip interface output
```

```yaml
- name: "Users | UNINSTALL | Remove automation-managed user accounts"
  cisco.ios.ios_users:
    ...
  loop: >-
    {{ local_users | default([])
       | rejectattr('username', 'equalto', ansible_user) | list }}
  # rejectattr filter: never remove the user we're currently logged in as
  # Without this guard, Ansible removes its own SSH session user and
  # the connection drops mid-play leaving the device in partial state
```

### The README Pairing

```bash
cat > ~/projects/ansible-network/playbooks/deploy/README.md << 'EOF'
# Deploy Playbooks

Each playbook in this directory supports both `--tags install` and
`--tags uninstall` modes. The install and uninstall operations are
contained in a single file to keep them in sync — when a new feature
is added to the install section, the uninstall counterpart must be
added in the same commit.

## Playbook Index

| Playbook | Install | Uninstall | Platforms | Notes |
|---|---|---|---|---|
| `deploy_ios_ccna_reversible.yml` | `--tags install` | `--tags uninstall` | IOS/IOS-XE | Full uninstall supported |
| `deploy_ios_ccnp.yml` | `--tags install` | partial | IOS/IOS-XE | BGP/OSPF/VRF reversible; QoS manual |
| `deploy_nxos.yml` | `--tags install` | `--tags uninstall` | NX-OS | VPC uninstall requires care |
| `deploy_junos.yml` | `--tags push` | rollback | Junos | Use `junos_config rollback: 1` |
| `deploy_panos.yml` | `--tags install` | `--tags uninstall` | PAN-OS | Rules/objects removable; zones manual |

## Tag Reference

All playbooks follow this convention:

```
--tags install              Run all install tasks
--tags uninstall            Run all uninstall tasks
--tags install,<section>    Run install tasks for one section
--tags uninstall,<section>  Run uninstall tasks for one section
```

## Reversibility Categories

- **Directly reversible**: `state: absent` in the same module (packages, users, files)
- **Config-reversible**: `state: deleted` or `no` prefix commands (protocols, ACLs, NTP)
- **Restore-reversible**: Replace with known default value (hostname → 'Router')
- **Not reversible**: Guarded behind a confirmation variable (destructive operations)

## Testing Uninstall Playbooks

Always test uninstall in Containerlab first:

```bash
# Full cycle test
./scripts/test_uninstall_cycle.sh deploy_ios_ccna_reversible.yml wan-r1
```

See `scripts/test_uninstall_cycle.sh` for the automated test workflow.

## Adding New Features

When adding a new configuration section to an install playbook:

1. Add the install tasks with `tags: [install, <section>]`
2. Immediately add the uninstall counterpart with `tags: [uninstall, <section>]`
3. Add the reversibility note to the playbook header block
4. Add the section to the README table above
5. Test the full install → uninstall → re-install cycle in Containerlab

**Never merge an install task without its uninstall counterpart.**
EOF
```

### The Uninstall Test Script

```bash
cat > ~/projects/ansible-network/scripts/test_uninstall_cycle.sh << 'EOF'
#!/bin/bash
# test_uninstall_cycle.sh — Full install/uninstall/reinstall cycle test
# Usage: ./scripts/test_uninstall_cycle.sh <playbook> <host>
# Example: ./scripts/test_uninstall_cycle.sh deploy_ios_ccna_reversible.yml wan-r1

set -e

PLAYBOOK="playbooks/deploy/${1}"
HOST="${2}"
RESULT_LOG="/tmp/uninstall_cycle_$(date +%Y%m%d_%H%M%S).log"

if [ -z "$PLAYBOOK" ] || [ -z "$HOST" ]; then
    echo "Usage: $0 <playbook> <host>"
    exit 1
fi

log() { echo "[$(date +%H:%M:%S)] $1" | tee -a "$RESULT_LOG"; }

log "════════════════════════════════════════════════════"
log " Uninstall Cycle Test"
log " Playbook: $PLAYBOOK"
log " Host:     $HOST"
log "════════════════════════════════════════════════════"

# Stage 1: Install
log ""
log "── Stage 1: INSTALL ─────────────────────────────────"
ansible-playbook "$PLAYBOOK" --tags install --limit "$HOST" 2>&1 | tee -a "$RESULT_LOG"
INSTALL_CHANGED=$(tail -5 "$RESULT_LOG" | grep "changed=" | grep -oP 'changed=\K\d+')
log "Install complete. Tasks changed: ${INSTALL_CHANGED:-unknown}"

# Stage 2: Dry-run uninstall
log ""
log "── Stage 2: DRY-RUN UNINSTALL ───────────────────────"
ansible-playbook "$PLAYBOOK" --tags uninstall --limit "$HOST" --check 2>&1 | tee -a "$RESULT_LOG"
log "Dry-run complete — no changes made"

# Stage 3: Execute uninstall
log ""
log "── Stage 3: UNINSTALL ───────────────────────────────"
ansible-playbook "$PLAYBOOK" --tags uninstall --limit "$HOST" 2>&1 | tee -a "$RESULT_LOG"
UNINSTALL_FAILED=$(tail -5 "$RESULT_LOG" | grep "failed=" | grep -oP 'failed=\K[^0]\d*' || echo "0")
if [ "$UNINSTALL_FAILED" != "0" ] && [ -n "$UNINSTALL_FAILED" ]; then
    log "FAIL: Uninstall had failures. Check log: $RESULT_LOG"
    exit 1
fi
log "Uninstall complete"

# Stage 4: Re-install
log ""
log "── Stage 4: RE-INSTALL ──────────────────────────────"
ansible-playbook "$PLAYBOOK" --tags install --limit "$HOST" 2>&1 | tee -a "$RESULT_LOG"
log "Re-install complete"

# Stage 5: Idempotency check
log ""
log "── Stage 5: IDEMPOTENCY CHECK ───────────────────────"
ansible-playbook "$PLAYBOOK" --tags install --limit "$HOST" 2>&1 | tee -a "$RESULT_LOG"
IDEMPOTENT_CHANGED=$(tail -5 "$RESULT_LOG" | grep "changed=" | grep -oP 'changed=\K\d+')
if [ "${IDEMPOTENT_CHANGED:-1}" != "0" ]; then
    log "FAIL: Idempotency broken — second install shows changed=${IDEMPOTENT_CHANGED}"
    exit 1
fi
log "PASS: Idempotency confirmed (changed=0)"

log ""
log "════════════════════════════════════════════════════"
log " ALL STAGES PASSED"
log " Full log: $RESULT_LOG"
log "════════════════════════════════════════════════════"
EOF
chmod +x ~/projects/ansible-network/scripts/test_uninstall_cycle.sh
```

---

## 24.7 — Reversibility Reference Card

Quick reference for the most common IOS configuration elements and their uninstall patterns:

| Config element | Install | Uninstall | Notes |
|---|---|---|---|
| Hostname | `ios_hostname state: merged` | `ios_hostname` with default value | Restore to `Router` |
| Banner | `ios_banner state: present` | `ios_banner state: absent` | Direct reversal |
| Interface IP | `ios_l3_interfaces state: merged` | `ios_l3_interfaces` with `ipv4: []` + `state: replaced` | Empty list removes IPs |
| Interface state | `ios_interfaces enabled: true` | `ios_interfaces enabled: false` | Shutdown the interface |
| NTP | `ios_ntp_global state: merged` | `ios_ntp_global state: deleted` | Removes all NTP |
| OSPF process | `ios_ospfv2 state: merged` | `ios_ospfv2 state: deleted` | Remove interface config first |
| BGP process | `ios_bgp_global state: merged` | `ios_bgp_global state: deleted` | All neighbors removed |
| Static routes | `ios_static_routes state: merged` | `ios_static_routes state: deleted` | Direct reversal |
| ACL | `ios_acls state: merged` | Remove app first, then `ios_acls state: deleted` | Order matters |
| Local users | `ios_users state: merged` | `ios_users state: deleted` | Never remove current user |
| SNMP | `ios_config lines: [snmp-server ...]` | `ios_config lines: [no snmp-server]` | One command removes all |
| VRF | `ios_vrf state: present` | `ios_vrf state: absent` | Remove all member interfaces first |
| VLAN | `ios_vlans state: merged` | `ios_vlans state: deleted` | Remove port assignments first |

---


Reversible automation is now a first-class practice in the project — every installation has a documented removal path, uninstall tasks are gated where destructive, and the full test cycle runs in Containerlab before anything approaches production. Part 25 closes the guide with a capstone: pulling everything together into a complete end-to-end automation run of the entire multi-vendor lab from a clean state.

