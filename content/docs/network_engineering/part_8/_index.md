---
draft: false
title: '8 - Structure'
description: "Part 8 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 8
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}

This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Ansible Directory Structure

This part locks in the definitive project layout, expands `ansible.cfg` with every setting explained, establishes naming conventions I'll follow for the rest of the project, and shows exactly how to separate playbooks by function so the project stays navigable as it grows.

---

## The Project Structure

Here is the complete, final directory structure for the `ansible-network` project.

{{< filetree/container >}}
  {{< filetree/folder name="ansible-network" >}}
    {{< filetree/file name="ansible.cfg" >}}
    {{< filetree/file name=".gitignore" >}}
    {{< filetree/file name=".ansible-lint" >}}
    {{< filetree/file name=".yamllint" >}}
    {{< filetree/file name=".pre-commit-config.yaml" >}}
    {{< filetree/file name="README.md" >}}
    {{< filetree/folder name="inventory" state="closed" >}}
      {{< filetree/file name="hosts.yml" >}}
      {{< filetree/file name="netbox.yml" >}}
      {{< filetree/folder name="group_vars" state="closed" >}}
        {{< filetree/file name="all.yml" >}}
        {{< filetree/file name="cisco_ios.yml" >}}
        {{< filetree/file name="cisco_nxos.yml" >}}
        {{< filetree/file name="paloalto.yml" >}}
        {{< filetree/file name="spine_switches.yml" >}}
        {{< filetree/file name="leaf_switches.yml" >}}
        {{< filetree/file name="wan.yml" >}}
        {{< filetree/file name="linux_hosts.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="host_vars" state="closed" >}}
        {{< filetree/file name="wan-r1.yml" >}}
        {{< filetree/file name="wan-r2.yml" >}}
        {{< filetree/file name="fw-01.yml" >}}
        {{< filetree/file name="spine-01.yml" >}}
        {{< filetree/file name="spine-02.yml" >}}
        {{< filetree/file name="leaf-01.yml" >}}
        {{< filetree/file name="leaf-02.yml" >}}
        {{< filetree/file name="host-01.yml" >}}
        {{< filetree/file name="host-02.yml" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="playbooks" state="closed" >}}
      {{< filetree/folder name="deploy" state="closed" >}}
        {{< filetree/file name="site.yml" >}}
        {{< filetree/file name="deply_base_config.yml" >}}
        {{< filetree/file name="deploy_vlans.yml" >}}
        {{< filetree/file name="deploy_bgp.yml" >}}
        {{< filetree/file name="deploy_ospf.yml" >}}
        {{< filetree/file name="deploy_acls.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="validate" state="closed" >}}
        {{< filetree/file name="validate_connectivity.yml" >}}
        {{< filetree/file name="validate_bgp.yml" >}}
        {{< filetree/file name="validate_vlans.yml" >}}
        {{< filetree/file name="validate_compliance.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="backup" state="closed" >}}
        {{< filetree/file name="backup_all.yml" >}}
        {{< filetree/file name="backup_ios.yml" >}}
        {{< filetree/file name="backup_nxos.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="rollback" state="closed" >}}
        {{< filetree/file name="rollback_config.yml" >}}
        {{< filetree/file name="rollback_bgp.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="report" state="closed" >}}
        {{< filetree/file name="report_facts.yml" >}}
        {{< filetree/file name="report_interfaces.yml" >}}
        {{< filetree/file name="report_bgp_neighbors.yml" >}}
      {{< /filetree/folder >}}

      {{< filetree/folder name="utils" state="closed" >}}
        {{< filetree/file name="update_packages.yml" >}}
        {{< filetree/file name="test_connectivity.yml" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="roles" state="closed" >}}
      {{< filetree/folder name="cisco_ios_base" state="open" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="cisco_nxos_base" state="open" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="panos_base" state="open" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="common_network" state="open" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="collections" state="closed" >}}
      {{< filetree/file name="requirements.yml" >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="templates" state="closed" >}}
      {{< filetree/folder name="ios" state="open" >}}
              {{< filetree/file name="base_config.j2" >}}
              {{< filetree/file name="bgp.j2" >}}
              {{< filetree/file name="acl.j2" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="nxos" state="open" >}}
              {{< filetree/file name="base_config.j2" >}}
              {{< filetree/file name="bgp.j2" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="panos" state="open" >}}
              {{< filetree/file name="security+policy.j2" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="files" state="closed" >}}
      {{< filetree/folder name="ios" state="open" >}}
              {{< filetree/file name="banner.txt" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="ssl" state="open" >}}
              {{< filetree/file name="ca-bundle.crt" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="vars" state="closed" >}}
      {{< filetree/file name="common_vlans.yml" >}}
      {{< filetree/file name="bgp_policy.yml" >}}
      {{< filetree/file name="acl_definitions.yml" >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="backups" state="closed" >}}
      {{< filetree/folder name="cisco_ios" state="open" >}}
        {{< filetree/folder name="wan-r1" state="open" >}}
        {{< /filetree/folder >}}
        {{< filetree/folder name="wan-r2" state="open" >}}
        {{< /filetree/folder >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="cisco_nxos" state="open" >}}
      {{< /filetree/folder >}}
      {{< filetree/folder name="paloalto" state="open" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="scripts" state="closed" >}}
      {{< filetree/file name="test-connectivity.sh" >}}
      {{< filetree/file name="setup.sh" >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name="containerlab" state="closed" >}}
      {{< filetree/file name="test-connectivity.sh" >}}
      {{< filetree/folder name="configs" state="open" >}}
        {{< filetree/file name="wan-r1-base.cfg" >}}
        {{< filetree/file name="spine-01-base.cfg" >}}
      {{< /filetree/folder >}}
    {{< /filetree/folder >}}

    {{< filetree/folder name=".vscode" state="closed" >}}
      {{< filetree/file name="settings.json" >}}
    {{< /filetree/folder >}}

  {{< /filetree/folder >}}
  {{< filetree/file name="hugo.toml" >}}
{{< /filetree/container >}}

- `inventory/` - The source of truth for what devices exist and how to reach them.
- `playbooks/` - Every playbook I wrike organized into subdirectories by function.
- `roles/` - Resuable, self-contained units of automation.
- `collections/` - Project level Ansible collections and the `requirements.yml` file. Keeps collection versions tied to the project.
- `templates/` - Jinja2 template files, organized by platform.
- `files/` - Static files that get deployed to devices unchanged (banner text, certificates, scripts).
- `vars/` - Variable files that don't belong in the inventory because they're not device specific.
- `backups/` - Generated by backup playbooks.
- `scripts/` - Shell and Python help scripts.
- `containerlab/` - Everything related to the lab environment.

---

{{< callout type="info" >}}

The `backups/` directory is committed to `.gitignore`. Usually, configuration backups have their own dedicated storage (either a separate Git repository, an S3 bucket, or dedicated backup platform like Oxidized.)

{{< /callout >}}

---

{{% steps %}}

#### Creating Directory Structure

After creating the structure layout, I create all directories that don't exist yet.

```bash
cd ~/projects/ansible-network
```

```bash
mkdir -p playbooks/{deploy,validate,backup,rollback,report,utils}
mkdir -p roles
mkdir -p templates/{ios,nxos,panos}
mkdir -p files/{ios,ssl}
mkdir -p vars
mkdir -p backups/{cisco_ios,cisco_nxos,paloalto}
mkdir -p scripts
```

I then added `backups/` to `.gitignore`

```bash {filename=".gitignore"}
backups/
```

Then commit it.

```bash
git add .gitignore
git commit -m "chore: exclude backups/ directory from git"
```

Git doesn't track empty directories, so I add `.gitkeep` placeholder files so the directory structure is committed.

```bash
find . -type d -empty -not -path "./.git/*" -not -path "./backups/*" \
    -exec touch {}/.gitkeep \;
```

Then commit the added files.

```bash
git add .
git commit -m "chore: add directory structure with .gitkeep placeholders"
```

---

#### Ansible Config File

The definitive version for this project.

```cfg {filename="ansible.cfg"}

[defaults]

inventory = inventory/

remote_user = ansible

host_key_checking = False

forks = 9

roles_path = roles/

collections_path = collections/:~/.ansible/collections

retry_files_enabled = False

stdout_callback = yaml

callbacks_enabled = ansible.posix.timer, ansible.posix.profile_tasks

deprecation_warnings = False

interpreter_python = auto_silent

vault_password_file = .vault_pass

gathering = explicit

log_path = /var/log/ansible/ansible.log

any_errors_fatal = False

timeout = 30

[inventory]

enable_plugins = yaml, ini, auto, netbox.netbox.nb_inventory

[ssh_connection]

ssh_args = -o ControlMaster=no -o ControlPersist=no -o StrictHostKeyChecking=no

pipelining = False

retries = 3

[persistent_connection]

connect_timeout = 30

command_timeout = 60

connect_retry_timeout = 15

[colors]

highlight = white
verbose = blue
warn = bright purple
error = red
debug = dark gray
deprecate = purple
skip = cyan
unreachable = red
ok = green
changed = yellow
diff_add = green
diff_remove = red
diff_lines = cyan

```

The 3 settings that must change before this config going into production:

```
#Lab                                        Production
host_key_checking = False                 > host_key_checking = True
ssh_args = ... StrictHostKeyChecking=no   > remove that argument
vault_password_file = .vault_pass         > make sure this is set and .vault_pass is secured
```

These settings should be added as well:

```
log_path = /var/log/ansible/ansible.log
any_errors_fatal = False
fact_checking = jsonfile
```

To check which `ansible.cfg` is active

```bash
ansible --version | grep "config file"
```

---

## Naming Conventions

Consistent naming helps navigate projects at any given time, whether it be 1 week or 1 year from now.

---

#### Playbook Files  {class="no-step-marker"}

```
Format: <verb>_<noun>[_<qualifier>].yml
        <verb> = deploy, validate, backup, rollback, report, test, update
        <noun> = what is being acted on
        <qualifier> = optional scope or platform
```

**Good Name Examples:**
- `deploy_bgp.yml`
- `deploy_vlans_nxos.yml`
- `backup_all_configs.yml`

**Not So Good Name Examples:**
- `bgp.yml`
- `fix.yml`
- `FINAL_acls.yml`

---

#### Role Names {class="no-step-marker"}

```
Format: <verb>_<platform>_<function>
        <function>
```

**Good Name Examples:**
- `cisco_ios_base`
- `panos_security_policy`
- `cmmon_ntp`

**Not So Good Name Examples:**
- `my_role`
- `new_cisco_role`
- `bgp_role_2`

---

#### Task Names {class="no-step-marker"}

**Good Name Examples:**
- name: "Deploy | Configure OSP process and area on WAN interfaces"
- name: "Validate | Confirm BGP neighbors are in Established state"

**Not So Good Name Examples:**
- name: task1
- name: configure

---

#### Variable Names {class="no-step-marker"}

```
Format: <scope>_<object>_<attribute>
```

**Good Name Examples:**
- bgp_as
- bgp_router_id
- ospf_area

**Not So Good Name Examples:**
- BGP_AS
- temp_var

---

#### File & Directory Names {class="no-step-marker"}

Always use:
- lowercase-with-hyphens for directories and most files
- lowercase_with_underscores for Python files and variable files

**Good Name Examples:**
- playbooks/deploy/
- cisco_ios_base/

**Not So Good Name Examples:**
- CamelCase/
- spaces in names/

{{% /steps %}}

---

## Playbooks by Function

The `playbooks/` directory is organized into subdirectories by function.

---

{{% steps %}}

#### Master Playbook

The master playbook calls other playbooks in sequence. Running `site.yml` deploys the entire environment.

```yaml {filename="playbooks/deploy/site.yml"}
---

- name: "Deploy | Base configuration on all network devices"
  import_playbook: deploy_base_config.yml
  tags:
    - base
    - always

- name: "Deploy | VLAN configuration on NX-OS switches"
  import_playbook: deploy_vlans.yml
  tags:
    - vlans
    - nxos

- name: "Deploy | OSPF routing on IOS-XE routers"
  import_playbook: deploy_ospf.yml
  tags:
    - ospf
    - routing
    - ios

- name: "Deploy | BGP configuration on all routing devices"
  import_playbook: deploy_bgp.yml
  tags:
    - bgp
    - routing

- name: "Deploy | ACL configuration on IOS-XE"
  import_playbook: deploy_acls.yml
  tags:
    - acls
    - security
    - ios
```

---

#### Base Configuration Playbook

This playbook will configure hostname, domain, NTP, DNS, syslog, SNMP and banners.

```yaml {filename="playbooks/deploy/deploy_base_config.yml"}
---

- name: "Deploy | Base configuration on Cisco IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Deploy | Set hostname"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged

    - name: "Deploy | Configure NTP servers"
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
              prefer: "{{ true if loop.index == 1 else false }}"
        state: merged
      loop: "{{ ntp_servers }}"

    - name: "Deploy | Configure DNS servers"
      cisco.ios.ios_config:
        lines:
          - "ip name-server {{ dns_servers | join {' '} }}"
    
    - name: "Deploy | Configure syslog"
      cisco.ios.ios_logging_global:
        config:
          hosts:
            - hostname: "{{ syslog_server }}"
              severity: informational
        state: merged

    - name: "Deploy | Save configuration"
      cisco.ios.ios_command:
        commands:
          - write memory

- name: "Deploy | Base configuration on Cisco NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Deploy | Set hostname"
      cisco.nxos.nxos_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged

    - name: "Deploy | Configure NTP servers"
      cisco.nxos.nxos_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
        state: merged
      loop: "{{ ntp_servers }}"
    
    - name: "Deploy | Configure syslog"
      cisco.nxos.nxos_logging_global:
        config:
          hosts:
            - name: "{{ syslog_server }}"
              severity: info
        state: merged
      
    - name: "Deploy | Save configuration"
      cisco.nxos.nxos_command:
        commands:
          - copy running-config startup-config
```

---

#### Validate Playbook

This playbook will verify basic device connectivity. It will test ssh reachability, device response, and basic interface state.

```yaml {filename="playbooks/validate/validate_connectivity.yml"}
---

- name: "Validate | Connectivity to all network devices"
  hosts: network_devices
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Validate | Confirm device responds to show version"
      cisco.ios.ios_command:
        commands:
          - show version
      register: show_version_output
      when: ansible_network_os == 'cisco.ios.ios'
    
    - name: "Validate | Asset show version contains expected platform"
      ansible.builtin.asset:
        that:
          - "'Cisco IOS' in show_version_output.stdout[0] or
             'Cisco IOS-XE' in show_version_output.stdout[0]"
        fail_msg: >
          FAIL: {{ inventory_hostname }} did not return expected IOS output.
          Got: {{ show_version_output.stdout[0][:100] }}
        success_msg: "PASS: {{ inventory_hostname }} is running Cisco IOS/IOS-XE"
      when: ansible_network_os == 'cisco.ios.ios'

    - name: "Validate | Confirm NX-OS device responds"
      cisco.nxos.nxos_command:
        command:
          - show version
      register: nxos_version_output
      when: ansible_network_os == 'cisco.nxos.nxos'

    - name: "Validate | Assert NX-OS version output"
      ansible.builtin.assert:
        that:
          - "'Nexus' in nxos_version_output.stdout[0]"
        fail_msg: "FAIL: {{ inventory_hostname }} did not return expected NX-OS output"
        success_msg: "PASS: {{ inventory_hostname }} os rimmomg NX-OS"
      when: ansible_network_os == 'cisco.nxos.nxos'

post_tasks:
  - name: "Validate | Print connectivity summary"
    ansible.builtin.debug:
      msg: "All connectivity checks passed for {{ inventory_hostname }}"
```

---

#### Backup Playbook

This playbook will back up running configurations from all devices.

```yaml {filename="playbooks/backup/backup_all.yml"}
---

- name: "Backup | Running configurations from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory for {{ inventory_hostname }}"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration from {{ inventory_hostname }}"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: ios_running_config

    - name: "Backup | Write configuration to file"
      ansible.builtin.copy:
        content: "{{ ios_running_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Confirm backup was written"
      ansible.builtin.debug:
        msg: "Backup saved: backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"

- name: "Backup | Running configurations from NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Backup | Create backup directory for {{ inventory_hostname }}"
      ansible.builtin.file:
        path: "backups/cisco_nxos/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration from {{ inventory_hostname }}"
      cisco.nxos.nxos_command:
        commands:
          - show running-config
      register: nxos_running_config

    - name: "Backup | Write configuration to file"
      ansible.builtin.copy:
        content: "{{ nxos_running_config.stdout[0] }}"
        dest: "backups/cisco_nxos/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost
```

---

#### Rollback Playbook

This playbook will restore a device configuration from backup.

```yaml {filename="playbooks/rollback/rollback_config.yml"}
---

- name: "Rollback | Restore IOS-XE configuration from backup"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  pre_tasks:
    - name: "Rollback | Verify backup file exists before proceeding"
      ansible.builtin.stat:
        path: "{{ backup_file }}"
      register: backup_stat
      delegate_to: localhost

    - name: "Rollback | Abort if backup file not found"
      ansible.builtin.fail:
        msg: >
          Backup file not found: {{ backup_file }}
          Rollback aborted. Provide a valid backup_file path with -e.
      when: not backup_stat.stat.exists

    - name: "Rollback | Take pre-rollback snapshot for comparison"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: pre_rollback_config

    - name: "Rollback | Save pre-rollback config as safety backup"
      ansible.builtin.copy:
        content: "{{ pre_rollback_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_pre_rollback_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.cfg"
        mode: '0644'
      delegate_to: localhost

  tasks:
    - name: "Rollback | Push backup configuration to {{ inventory_hostname }}"
      cisco.ios.ios_config:
        src: "{{ backup_file }}"
        replace: config    # Replace entire config, not merge
      register: rollback_result

    - name: "Rollback | Save configuration after rollback"
      cisco.ios.ios_command:
        commands:
          - write memory

  post_tasks:
    - name: "Rollback | Confirm device is responsive after rollback"
      cisco.ios.ios_command:
        commands:
          - show version
      register: post_rollback_check

    - name: "Rollback | Report rollback result"
      ansible.builtin.debug:
        msg: >
          Rollback complete for {{ inventory_hostname }}.
          Changed: {{ rollback_result.changed }}
          Device is responsive: {{ post_rollback_check is succeeded }}
```

---

#### Report Playbook

This playbook will gather and display facts from all devices.

```yaml {filename="playbooks/report/report_facts.yml"}
---

- name: "Report | Gather facts from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Report | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - hardware
          - default
      register: ios_facts

    - name: "Report | Display IOS device summary"
      ansible.builtin.debug:
        msg:
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Platform:  {{ ansible_net_model | default('unknown') }}"
          - "Version:   {{ ansible_net_version }}"
          - "Uptime:    {{ ansible_net_stacked_models | default('N/A') }}"
          - "Interfaces:{{ ansible_net_interfaces | length }} total"

- name: "Report | Gather facts from NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli

  tasks:
    - name: "Report | Gather NX-OS device facts"
      cisco.nxos.nxos_facts:
        gather_subset:
          - hardware
          - default

    - name: "Report | Display NX-OS device summary"
      ansible.builtin.debug:
        msg:
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Platform:  {{ ansible_net_platform }}"
          - "Version:   {{ ansible_net_version }}"
          - "Interfaces:{{ ansible_net_interfaces | length }} total"
```

{{% /steps %}}

---

## Host Vars vs Group Vars vs Play-Level Vars

Here's a decision guide on when to use each type:

```
Is this variable the same for every device in the entire inventory?
  Yes → group_vars/all.yml
  No ↓

Is this variable the same for every device in a speicifc platform group?
  Yes → group_vars/<platform>.yml
  No ↓

Is this variable the same for a specific device role or logical group?
  Yes → group_vars/<group>.yml
  No ↓

Is this variable unique to a specific device?
  Yes → host_vars/<hostname>.yml
  No ↓

Is this variable only relevant to a specific play (no reusable)?
  Yes → vars: block inside the playbook
  No ↓

Is this variable shared across multiple playbooks but not device-specific?
  Yes → vars/ directory file
```

---

{{% steps %}}

#### Examples

**Same for every device**

```yaml {filename="group_vars/all.yml"}
ntp_servers:
  - 216.239.35.0
  - 216.239.35.4
ansible_user: ansible
```

---

**Same for every IOS device**

```yaml {filename="group_vars/cisco_ios.yml"}
ansible_network_os: cisco.ios.ios
ansible_become: true
```

---

**Same for every IOS device**

```yaml {filename="group_vars/spine_switches.yml"}
device_role: spine
bgp_as: 65000
fabric_mtu: 9216
```

---

**Unique to Spine-01**

```yaml {filename="host_vars/spine-01.yml"}
ansible_host: 172.16.0.21
loopback0:
  ip: 10.255.1.1
bgp_router_id: 10.255.1.1
```

---

**Relevant only to this play**

vars: inside a playbook

```yaml
- name: Deploy BGP
  hosts: cisco_ios
  vars:
    bgp_timer_keepalive: 30
    bgp_timer_holdtime: 90
  tasks: ...
```

---

**Shared across playbooks**

```yaml {filename="vars/bgp_policy.yml"}
route_map_permit_local:
  - prefix: 10.0.0.0/8
  - prefix: 172.16.0.0/12
```

{{% /steps %}}

---

## Sensitive Data

Sensitive data must never be visible in the directory root since people will be able to see it if they clone the repo or visit the Github page.

**Safe for project root:**
- ansible.cfg
- .gitignore
- README.md
- .ansible-lint
- .yamllint
- requirements files

**Never put in project root:**
- passwords or tokens
- private keys
- .vault_pass (put in .gitignore)
- .env files with secrets
- unencrypted credential files
- production IP addressing details



