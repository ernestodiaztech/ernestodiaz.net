---
draft: false
title: '10 - Playbooks'
description: "Part 10 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 10
---

{{< badge "Ansible" >}}
{{< badge content="Linux" color="red" >}}

---

{{< lab-callout type="info" >}}
This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.
{{< /lab-callout >}}

---

## Playbooks

A playbook is a YAML file containing 1 of more "plays". Each "play" targets a group of hosts and defines a list of tasks to run against them.

---

{{< subtle-label >}}Structure{{< /subtle-label >}}

{{< codeblock lang="YAML" syntax="yaml" lines="true" >}}
---

- name: "Play name"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  vars:
    my_variable: value

  vars_files:
    - vars/bgp_policy.yml

  pre_tasks:
    - name: "Pre-task name"
      module_name:
        parameter: value
  tasks:
    - name: "Task name"
      module_name:
        parameter: value
      register: result_var
      when: condition
      loop: "{{ list }}"
      notify: handler_name
      tags:
        - tag_name

  post_tasks:
    - name: "Post-task name"
      module_name:
        parameter: value

  handlers:
    - name: handler_name
      module_name:
        parameter: value
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: YAML document start marker.

Line 3:
: A play begins with a dash

Line 4:
: Which inventory group or host to target

Line 5:
: Whether to run the facts module before tasks

Line 6:
: How to connect to devices

Line 7:
: Enable privilege escalation (`enable mode` for IOS)

Line 8:
: How to escalate (`enable` for IOS, `sudo` for Linux)

Line 10:
: Play level variables (optional)

Line 13:
: Load variables from external files (optional)

Line 16:
: Tasks that run before roles and main tasks

Line 21:
: The main task list

Line 22:
: Always required (appears in playbook output)

Line 23:
: The Ansible module to use

Line 24:
: Module parameters

Line 25:
: Store the task's return value in a variable

Line 26:
: Only run thistask if condition is true

Line 27:
: Run this task once for each item in the list

Line 28:
: Trigger a handler if this task reports changed

Line 29:
: Tag for selective playbook execution
{{< /line-explain >}}

---

{{< subtle-label >}}Required Fields{{< /subtle-label >}}

These 6 fields appear in every network device playbook I write.

- `- name: "Name"` - Required for readable output
- `hosts: cisco_ios` - Required to state who to run against
- `gather_facts: false` - Required because network devices can't run Python facts
- `connection: network_cli` - Required because this tells Ansible to use CLI over SSH
- `become: true` - Required for IOS so that Ansible can enter 'enable mode'
- `become_method: enable` - Required for IOS since it's how Ansible gets to 'enable`

---

{{< subtle-label >}}Multiple Plays{{< /subtle-label >}}

A single playbook file can contain multiple plays, each targeting different devices.

{{< codeblock lang="YAML" syntax="yaml" >}}
---
- name: "Play 1 - Configure IOS devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  tasks:
    - ....

- name: "Play 2 - Configure NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tasks:
    - ...


- name: "Play 3 - Verify Linux hosts"
  hosts: linux_hosts
  gather_facts: true
  tasks:
    - ....
{{< /codeblock >}}

---

## Network_Cli Plugin

`connection: network_cli` is the engine behind all SSH-based network device automation.

When Ansible encounters a task targeting a host with `connection: network_cli`:

1. Opens an SSH connection to ansible_host using Paramiko
2. Authenticates with ansible_user / ansible_password (or SSH key)
3. Detects the device prompt using ansible_network_os
4. Handles login banners, pagination (--More--), and prompt patterns
5. Enters enable mode if ansible_become: true (sends `enable` + become_password)
6. Sends the CLI command(s) from the module
7. Reads back the output until the device prompt returns
8. Returns the output to the module for parsing
9. Keeps the SSH connection alive for subsequent tasks in the same play

Ansible doesn't open and close an SSH session for every task.

---

`ansible_network_os` is what tells the `network_cli` plugin which terminal handler to load (how to recognize the device prompt, how ot enter config mode, how to handle pagination).

Examples:

```yaml
ansible_network_os: cisco.ios.ios

ansible_network_os: cisco.nxos.nxos

ansible_network_os: paloaltonetworks.panos.panos
```

The format is always <collection_namespace>.<collection_name>.<platform>

---

## First Playbook

I'll build 1 playbook file that grows throughout this part. Starting with gathering facts.

{{< codeblock file="playbooks/report/first_playbook.yml" syntax="yaml" >}}
---
- name: "Report | Gather and display facts from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:

    - name: "Facts | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - interfaces
          - hardware

    - name: "Facts | Deplay device summary"
      ansible.builtin.debug:
        msg:
          - "============================"
          - "Device: {{ inventory_hostname }}"
          - "Hostname: {{ ansible_net_hostname }}"
          - "Version: {{ ansible_net_version }}"
          - "Model: {{ ansible_net_model | default{'unknown'} }}"
          - "Serial: {{ ansible_net_serialnum | default{'unknown'} }}"
          - "Interfaces: {{ansible_net_interfaces | length }} configured"
          - "============================"
{{< /codeblock >}}

---

Running it:

{{< codeblock lang="Bash" syntax="bash" >}}
cd ~/projects/ansible-network
ansible-playbook playbooks/report/first_playbook.yml
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
PLAY [Report | Gather and display facts from IOS-XE devices] ***

TASK [Facts | Gather IOS device facts] *********************************
ok: [wan-r1]
ok: [wan-r2]

TASK [Facts | Display device summary] **********************************
ok: [wan-r1] => {
    "msg": [
        "======================================",
        "Device:    wan-r1",
        "Hostname:  wan-r1",
        "Version:   17.06.01",
        "Model:     CSR1000V",
        "Serial:    ...",
        "Interfaces:3 configured",
        "======================================"
    ]
}
ok: [wan-r2] => {
    "msg": [
        "======================================",
        "Device:    wan-r2",
        ...
    ]
}

PLAY RECAP *************************************************************
wan-r1    : ok=2    changed=0    unreachable=0    failed=0
wan-r2    : ok=2    changed=0    unreachable=0    failed=0
{{< /codeblock >}}

---

The play recap is the summary at the bottom of every playbook run:

{{< codeblock lang="" copy="false" >}}
wan-r1 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
         │         │             │                │            │            │            │
         │         │             │                │            │            │            └── Tasks with ignore_errors: true that failed
         │         │             │                │            │            └── Tasks rescued by block/rescue
         │         │             │                │            └── Tasks skipped due to when: condition
         │         │             │                └── Tasks that returned an error
         │         │             └── Hosts unreachable via SSH
         │         └── Tasks that made a configuration change
         └── Total tasks that completed successfully (including changed)
{{< /codeblock >}}

---

## Capturing Task Output

`register` saves a task's return value into a variable for use in subsequent tasks. Without `register` the output of a task is displayed in the play recap but then discarded.

---

{{< subtle-label >}}Adding Register{{< /subtle-label >}}

I add a task that runs a show command and registers the output:

{{< codeblock lang="YAML" syntax="yaml" >}}
---
- name: "Report | Gather and display facts from IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:

    - name: "Facts | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - interfaces
          - hardware

    - name: "Facts | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device:    {{ inventory_hostname }}"
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Version:   {{ ansible_net_version }}"
          - "Model:     {{ ansible_net_model | default('unknown') }}"

    # ── NEW: register a show command output ──────────────────────────
    - name: "Facts | Run show ip interface brief"
      cisco.ios.ios_command:
        commands:
          - show ip interface brief
      register: interface_brief

    - name: "Facts | Display interface brief output"
      ansible.builtin.debug:
        msg: "{{ interface_brief.stdout_lines[0] }}"
        # interface_brief is the dict returned by ios_command
        # .stdout_lines is a list — one entry per command
        # [0] is the first command's output, already split into lines
{{< /codeblock >}}

---

{{< subtle-label >}}Anatomy of a Register Variable{{< /subtle-label >}}

When I register the ouput of `ios_command` the variable contains a dictionary with many keys. Here's what's inside `interface_brief` after the task runs:

{{< codeblock lang="Bash" syntax="bash" >}}
interface_brief = {
    "changed": False,           # Did this task change anything?
    "failed": False,            # Did this task fail?
    "stdout": [                 # Raw output — one string per command
        "Interface              IP-Address      OK? Method Status    Protocol\n
         GigabitEthernet1       10.10.10.1      YES manual up        up\n
         GigabitEthernet2       10.10.20.1      YES manual up        up\n
         Loopback0              10.255.0.1      YES manual up        up"
    ],
    "stdout_lines": [           # Same output, split into lines — easier to work with
        [
            "Interface              IP-Address      OK? Method Status    Protocol",
            "GigabitEthernet1       10.10.10.1      YES manual up        up",
            "GigabitEthernet2       10.10.20.1      YES manual up        up",
            "Loopback0              10.255.0.1      YES manual up        up"
        ]
    ]
}
{{< /codeblock >}}

I access parts of this dictionary using dot notation or bracket notation in Jinja2

{{< codeblock lang="YAML" syntax="yaml" lines="true" >}}
"{{ interface_brief.stdout[0] }}"           # Raw string — first command output
"{{ interface_brief.stdout_lines[0] }}"     # List of lines — first command output
"{{ interface_brief.stdout_lines[0][0] }}"  # First line of first command output
"{{ interface_brief.failed }}"              # Boolean — did the task fail?
"{{ interface_brief.changed }}"             # Boolean — did the task change anything?
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Raw string

Line 2:
: List of lines

Line 3:
: First line of first command output

Line 4:
: Boolean (did task fail)

Line 5:
: Boolean (did task change anything)
{{< /line-explain >}}

---

{{< subtle-label >}}Using debug Effectively{{< /subtle-label >}}

`ansible.builtin.debug` is the `print()` statement of playbooks. It outputs information during an run without making any changes. I use it constantly during development and for building reporting playbooks.

---

{{< subtle-label >}}3 Forms{{< /subtle-label >}}

Form 1: msg - display a string or list of strings

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Debug | Show a message"
  ansible.builtin.debug:
    msg: "The hostname is {{ ansible_net_hostname }}"
{{< /codeblock >}}

Form 2: var - deump an entire variable's contents

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Debug | Dump variable contents"
  ansible.builtin.debug:
    var: interface_brief
{{< /codeblock >}}

Form 3: msg with a list - displays each item on its own line

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Debug | Show multiple lines"
  ansible.builtin.debug:
    msg:
      - "Hostname: {{ ansible_net_hostname }}"
      - "Version:  {{ ansible_net_version }}"
      - "Uptime:   {{ ansible_net_all_ipv4_addresses | join(', ') }}"
{{< /codeblock >}}

---

{{< subtle-label >}}Verbosity-Gated Debug Messages{{< /subtle-label >}}

I can set a `verbosity:` level on debug tasks so they only print when I run the playbook with `-v` or higher. This lets me leave development debug output in the playbook without cluttering normal runs.

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Debug | Full interface_brief dump (dev only)"
  ansible.builtin.debug:
    var: interface_brief
    verbosity: 2
{{< /codeblock >}}

Normal ru: this task is silently skipped. With `-vv:` this task prints the full variable dump.

---

## Adding a Configuration Change

Now I add the first configuration changing task to the playbook. This will set the hostname to match what's in `host_vars`.

{{< codeblock lang="YAML" file="first_playbook.yml" syntax="yaml" >}}
---
- name: "Report and Configure | IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:

    - name: "Facts | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - interfaces
          - hardware

    - name: "Facts | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device:    {{ inventory_hostname }}"
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Version:   {{ ansible_net_version }}"
          - "Model:     {{ ansible_net_model | default('unknown') }}"

    - name: "Facts | Run show ip interface brief"
      cisco.ios.ios_command:
        commands:
          - show ip interface brief
      register: interface_brief

    - name: "Facts | Display interface brief output"
      ansible.builtin.debug:
        msg: "{{ interface_brief.stdout_lines[0] }}"

    # ── NEW: push hostname configuration ─────────────────────────────
    - name: "Config | Set hostname from inventory (device_hostname variable)"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"    # From host_vars/<hostname>.yml
        state: merged
      register: hostname_result

    - name: "Config | Report hostname change result"
      ansible.builtin.debug:
        msg: "Hostname change — changed: {{ hostname_result.changed }}"
{{< /codeblock >}}

---

{{< subtle-label >}}Running Updated Playbook{{< /subtle-label >}}

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml
{{< /codeblock >}}

First run (hostname was already correct):

{{< codeblock lang="Expected Output" copy="false" >}}
TASK [Config | Set hostname from inventory] ****************************
ok: [wan-r1]      ← ok means no change needed — already correct
ok: [wan-r2]

PLAY RECAP *************************************************************
wan-r1 : ok=6    changed=0    ...
wan-r2 : ok=6    changed=0    ...
{{< /codeblock >}}

If I manually set a wrong hostname on `wan-r` and run again:

{{< codeblock lang="Expected Output" copy="false" >}}
TASK [Config | Set hostname from inventory] ****************************
changed: [wan-r1]   ← changed means it applied the correct hostname

PLAY RECAP *************************************************************
wan-r1 : ok=6    changed=1    ...
{{< /codeblock >}}

---

## Idempotency

Idempotency is one of the most important concepts in Ansible. An operation is idempotent if running it multiple times produces the same result as running it once. If I run a playbook twice with the same inputs and the device is already in the desired state, the second run makes no changes.

**Why Idempotency Matters**

Without it automation is dangerous:
- Run a playbook twice → duplicate VLAN entries in the config
- Run a backup playbook twice → configuration gets applied twice, potentially corrupting state
- Scheduled automation runs a playbook every hour → device slowly accumlates duplicate lines

With it I can:
- Run a playbook any number of times safely
- Use automation as continuous enforcement (run every hour to keep devices compliant)
- Retry failed playbooks without worrying about partial apply side effects

{{< subtle-label >}}Resource Modules{{< /subtle-label >}}

Resource modules are idempotent by design. They compare the desired state against the current state and only pushing changes when there's a difference.

First run - hostname is wrong, module fixes it:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
TASK [Config | Set hostname from inventory]
changed: [wan-r1]    # ← Applied the correct hostname

PLAY RECAP: wan-r1 : ok=6  changed=1  failed=0
{{< /codeblock >}}

Second run - hostname is now correct, no change is made:

{{< codeblock lang="Expected Output" copy="false" >}}
TASK [Config | Set hostname from inventory]
ok: [wan-r1]         # ← Already correct, nothing to do

PLAY RECAP: wan-r1 : ok=6  changed=0  failed=0
{{< /codeblock >}}

This false positive `changed` is a known limitation of `ios_config`. Every `changed: true` in a playbook is supposed to mean "I made a change". False positives erode trust in the output, I stop believing that `changed: 1` means something actually changed.

---

{{< subtle-label >}}Comparison{{< /subtle-label >}}

Module Type | Example | Idempotent? | How it Checks
---|---|---|---|
Resource Module | `ios_vlans` | Yes| Structured state comparison |
Resource Module | `ios_hostname` | Yes | Structured state comparison |
Resource module | `ios_bgp_global` | Yes | Strcutured state comparison |
Config module | `ios_config` | Partial | Text line matching
Command module | `ios_command` | Always `ok` | Read-only, never changes

---

{{< subtle-label >}}Rules to Follow{{< /subtle-label >}}

> Use resource modules whenever they exist for what I'm trying to configure. Fall back to `ios_config` only when no resource module covers it

Resource module coverage for IOS: `ios_vlans`, `ios_interfaces`, `ios_l2_interfaces`, `ios_l3_interfaces`, `ios_bgp_global`, `ios_ospf_interfaces`, and more.

The list of resource modules grows with every collection release. When I need to configure something new I check `ansible-doc cisco.ios.ios_<tab>` to see if a resource module exists before reaching for `ios_config`.

---

When `ios_config` is necessary, I use the `parents:` parameter to narrow the comparison scope, and `save_when: changed` to only save when a change was actually made.

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Configure interface description (ios_config)"
  cisco.ios.ios_config:
    parents: "interface GigabithEthernet1"
    lines:
      - "description WAN | To FW-01 eth1"
    save_when: changed
{{< /codeblock >}}

---

## Adding a Backup Task

I add the backup task to the growing playbook. This reinforces the `register` pattern and introduces `delegate_to: localhost`.

{{< codeblock file="first_playbook.yml" syntax="yaml" >}}
---
- name: "Report, Configure, and Backup | IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:

    - name: "Facts | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - interfaces
          - hardware

    - name: "Facts | Display device summary"
      ansible.builtin.debug:
        msg:
          - "Device: {{ inventory_hostname }}"
          - "Hostname: {{ ansible_net_hostname }}"
          - "Version: {{ ansible_net_version }}"
          - "Model: {{ ansible_net_model | default('unknown') }}"

    - name: "Facts | Run show ip interface brief"
      cisco.ios.ios_command:
        commands:
          - show ip interface brief
      register: interface_brief

    - name: "Facts | Display interface brief output"
      ansible.builtin.debug:
        msg: "{{ interface_brief.stdout_lines[0] }}"

    - name: "Config | Set hostname from inventory"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      register: hostname_result

    - name: "Config | Report hostname change result"
      ansible.builtin.debug:
        msg: "Hostname change — changed: {{ hostname_result.changed }}"

    # ── backup tasks ─────────────────────────────────────────────
    - name: "Backup | Create backup directory on control node"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: running_config

    - name: "Backup | Write configuration to file on control node"
      ansible.builtin.copy:
        content: "{{ running_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Confirm backup location"
      ansible.builtin.debug:
        msg: "Backup saved: backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
{{< /codeblock >}}

---

{{< subtle-label >}}Delegate_to: localhost Pattern Explained{{< /subtle-label >}}

Every task in a play normally runs on the remote host (the network device). But creating directories and writing files needs to happen on the control nose. `delegate_to: localhost` tells Ansible to run that specific task on the control node instead of the remote host.

{{< topology1 src="diagrams/delegate-to-comparison.svg" >}}

The `inventory_hostname` variable still refers to `wan-r1` even when the task runs on localhost (this is what makes the backup file path `backups/cisco_ios/wan-r1...cfg` correct). The variable context stays attached to the device even though the task execution moved to localhost.

---

## When Conditions

The `when:` keyword makes a task conditional. It evaluates Jinja2 expression and skips the task if the result is false. This is how I write playbooks that handle multiple platforms or make decisions based on device state.

---

{{< subtle-label >}}Basic Pattern{{< /subtle-label >}}

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Task name"
  module:
    parameter: value
  when: condition_expression
{{< /codeblock >}}

**`when` Based on `ansible_network_os`**

The most common use in multi-platform playbooks is to run different tasks depending on which platform is being targeted:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Configure NTP (IOS-specific module)"
  cisco.ios.ios_ntp_global:
    config:
      servers:
        - server: "{{ ntp_servers[0] }}"
    state: merged
  when: ansible_network_os == 'cisco.ios.ios'

- name: "Config | Configure NTP (NX-OS specific module)"
  cisco.nxos.nxos_ntp_global:
    config:
      servers:
        - server: "{{ ntp_servers[0] }}"
    state: merged
  when: ansible_network_os == 'cisco.nxos.nxos'
{{< /codeblock >}}

If I run this against a group that has both IOS and NX-OS devices, each device only runs the task that matches its platform. The other task is skipped with `skipping: [hostname]` in the output.

---

**`when` Based on a Registered Result**

I can gate a task on the result of a previous task:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Facts | Check if BGP is running"
  cisco.ios.ios_command:
    commands:
      - show ip bgp summary
  register: bgp_check
  ignore_errors: true

- name: "Report | BGP is configured on this device"
  ansible.builtin.debug:
    msg: "BGP is active on {{ inventory_hostname }}"
  when: bgp_check.failed == false

- name: "Report | BGP is NOT configured on this device"
  ansible.builtin.debug:
    msg: "BGP is NOT running on {{ inventory_hostname }}"
  when: bgp_check.failed == true
{{< /codeblock >}}

---

**`when` Based on a Fact Variable**

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Apply IOS-XE 17.x specific configuration"
  cisco.ios.ios_config:
    lines:
      - "platform punt-keepalive disable-kernel-core"
  when: ansible_net_version is version ('17.0', '>=')

- name: "Config | CSR-specific tuning"
  cisco.ios.ios_config:
    lines:
      - "platform hardware throughput level MB 1000"
  when: "'CSR' in ansible_net_model"
{{< /codeblock >}}

---

**`when` With The `device_role` Inventory Variable**

This is the pattern I use most ofthen when using the `device_role` variable from `host_vars` to control which tasks run on which devices:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Apply spine-specific BGP configuration"
  cisco.nxos.nxos_bgp_global:
    config:
      as_number: "{{ bgp_as }}"
    state: merged
  when: device_role == 'spine'

- name: "Config | Apply leaf-specific VLAN configuration"
  cisco.nxos.nxos_vlans:
    config: "{{ base_vlans }}"
    state: merged
  when: device_role == 'leaf'
{{< /codeblock >}}

---

**Adding `when` to the Growing Playbook**

I add a conditional that only runs the hostname task when the current hostname doesn't match what's in inventory:

{{< codeblock file="first_playbook.yml" syntax="yaml" >}}
    - name: "Config | Set hostname (only if it needs to change)"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      register: hostname_result
      when: ansible_net_hostname != device_hostname
{{< /codeblock >}}

---

## Looping Over Tasks

Many network configuration tasks involve applying the same operation to a list of items (configuring multiple VLANs, multiple NTP servers, etc). Loops avoid writing repetitive tasks.

---

{{< subtle-label >}}Monder Approach{{< /subtle-label >}}

`loop` is the current standard. It iterates over a list, running the task once for each item.

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Add multiple VLANs"
  cisco.nxos.nxos_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
    state: merged
  loop:
    - { id: 10, name: "MGMT" }
    - { id: 20, name: "APP_SERVERS" }
    - { id: 30, name: "DB_SERVERS" }
{{< /codeblock >}}

Or loop over a variable list from inventory:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Configure all base VLANs from group_vars"
  cisco.nxos.nxos_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
    state: merged
  loop: "{{ base_vlans }}"
  loop_control:
    label: "VLAN {{item.id }} - {{ item.name }}"
{{< /codeblock >}}

`Leaf_switches`, `group_vars` defines `base_vlans` as a list. This loop read that list and configure each VLAN.

---

{{< subtle-label >}}Loop Output{{< /subtle-label >}}

Without `loop_control.label` each iteration shows the full `item` dict in the output. With `label` I get clean readable output:

**Without `label`**

{{< codeblock lang="" copy="false" >}}
TASK [Config | Configure all base VLANs] ***
changed: [leaf-01] => (item={'id': 10, 'name': 'MGMT', 'subnet': '192.168.10.0/24'})
changed: [leaf-01] => (item={'id': 20, 'name': 'APP_SERVERS', 'subnet': '192.168.20.0/24'})
{{< /codeblock >}}

**With `label`**

{{< codeblock lang="" copy="false" >}}
TASK [Config | Configure all base VLANs] ***
changed: [leaf-01] => (item=VLAN 10 — MGMT)
changed: [leaf-01] => (item=VLAN 20 — APP_SERVERS)
{{< /codeblock >}}

---

**Looping Over a Simple List**

Loop over a flat list of strings:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Enable required NX-OS features"
  cisco.nxos.nxos_features:
    feature: "{{ item }}"
    state: enabled
  loop:
    - bgp
    - ospf
    - interface-vlan
    - lacp
    - lldp
  loop_control:
    label: "feature: {{ item }}"
{{< /codeblock >}}

---

**Looping Over `bgp_neighbors` from `host_vars`**

BGP neighbors list in `host_vars/wan-r1.yml` drives the BGP neighbor configuration:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Configure BGP neighbors from host_vars"
  cisco.ios.ios_bgp_address_family:
    config:
      as_number: "{{ bgp_as }}"
      address_family:
        - afi: ipv4
          neighbors:
            - neighbor_address: "{{ item.neighbor_ip }}"
              remote_as: "{{ item.remote_as }}"
              description: "{{ item.description }}"
    state: merged
  loop: "{{ bgp_neighbors }}"
  loop_control:
    label: "BGP neighbor {{ item.neighbor_ip }} (AS {{ item.remote_as }})"
{{< /codeblock >}}

---

**Legacy Approach**

`with_items` is the older loop syntax. It works identically to `loop` for simple lists.

Example:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Enable features (legacy with_items)"
  cisco.nxos.nxos_feature:
    feature: "{{ item }}"
    state: enabled
  with_items:
    - bgp
    - ospf
    - interface-vlan
{{< /codeblock >}}

Same but with `loop`:

{{< codeblock lang="YAML" syntax="yaml" >}}
- name: "Config | Enable features (modern loop)"
  cisco.nxos.nxos_feature:
    feature: "{{ item }}"
    state: enabled
  loop:
    - bgp
    - ospf
    - interface-vlan
{{< /codeblock >}}

---

{{< subtle-label >}}Key Differences{{< /subtle-label >}}

Feature | `loop` | `with_items`
--- | --- | ---
Status | Current standard | Legacy
Nested lists | Doesn't flatten | Flattens one level automatically
Complex iteration | Works with `loop_control` | Limited options
`loop_control.label` | Supported | Not avaiable
Reading old playbooks | May see `with_items` | Needs recognition

---

## Complete Playbook

Here is the complete `first_playbook.yml` with all concepts integrated:

{{< codeblock lang="YAML" syntax="yaml" >}}
---

- name: "Part 10 | Gather, configure, and backup IOS-XE devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:

    # ── SECTION 1: Gather Facts ───────────────────────────────────────
    - name: "Facts | Gather IOS device facts"
      cisco.ios.ios_facts:
        gather_subset:
          - default
          - interfaces
          - hardware

    - name: "Facts | Display device summary"
      ansible.builtin.debug:
        msg:
          - "======================================"
          - "Device:    {{ inventory_hostname }}"
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Version:   {{ ansible_net_version }}"
          - "Model:     {{ ansible_net_model | default('unknown') }}"
          - "Serial:    {{ ansible_net_serialnum | default('unknown') }}"
          - "======================================"

    # ── SECTION 2: Register and Debug ────────────────────────────────
    - name: "Facts | Gather interface brief"
      cisco.ios.ios_command:
        commands:
          - show ip interface brief
      register: interface_brief

    - name: "Facts | Display interface brief"
      ansible.builtin.debug:
        msg: "{{ interface_brief.stdout_lines[0] }}"

    - name: "Facts | Full register dump (dev only — gated at -vv)"
      ansible.builtin.debug:
        var: interface_brief
        verbosity: 2    # Only shown with -vv or higher

    # ── SECTION 3: Conditional Hostname Config ────────────────────────
    - name: "Config | Set hostname (when inventory name differs from device)"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      register: hostname_result
      when: ansible_net_hostname != device_hostname

    - name: "Config | Confirm hostname (skipped if no change was needed)"
      ansible.builtin.debug:
        msg: "Hostname set to {{ device_hostname }} — changed: {{ hostname_result.changed }}"
      when: hostname_result is not skipped

    # ── SECTION 4: Loop — Configure NTP Servers ──────────────────────
    - name: "Config | Configure NTP servers from group_vars/all.yml"
      cisco.ios.ios_ntp_global:
        config:
          servers:
            - server: "{{ item }}"
              prefer: "{{ true if loop.index == 1 else false }}"
        state: merged
      loop: "{{ ntp_servers }}"
      loop_control:
        label: "NTP server: {{ item }}"

    # ── SECTION 5: Platform-Conditional Task ─────────────────────────
    - name: "Config | IOS-XE only — disable IP source routing"
      cisco.ios.ios_config:
        lines:
          - no ip source-route
      when: ansible_network_os == 'cisco.ios.ios'
      # This task would be skipped on NX-OS or PAN-OS if this play
      # were extended to target multiple platforms

    # ── SECTION 6: Backup ─────────────────────────────────────────────
    - name: "Backup | Create backup directory"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost

    - name: "Backup | Gather running configuration"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: running_config

    - name: "Backup | Write configuration to control node"
      ansible.builtin.copy:
        content: "{{ running_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost

    - name: "Backup | Report backup location"
      ansible.builtin.debug:
        msg: "Saved: backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
{{< /codeblock >}}

---

{{< subtle-label >}}Running the Playbook{{< /subtle-label >}}

Full run:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml
{{< /codeblock >}}

Limit to 1 device:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml -l wan-r1
{{< /codeblock >}}

Run only specific tagged sections:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml --tags backup
{{< /codeblock >}}

Show developer debug output:

{{< codeblock lang="Bash" syntax="bash" >}}
ansible-playbook playbooks/report/first_playbook.yml -vv
{{< /codeblock >}}

---

The core playbook concepts are now in place. Now onto Jinja2 templating.