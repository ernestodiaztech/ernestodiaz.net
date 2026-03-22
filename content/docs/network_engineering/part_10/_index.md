---
draft: true
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

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 10: Writing Your First Network Playbooks

> *Ad-hoc commands from Part 9 are powerful for quick checks, but they're not repeatable, they're not version-controlled, and they can't express logic — conditions, loops, decisions based on what a device returns. Playbooks are where all of that lives. This part builds a single cohesive playbook that starts simple and grows — adding one concept at a time — until it's a complete, production-quality network automation file. Every concept introduced here is used in every playbook for the rest of this guide.*

---

## 10.1 — Anatomy of a Playbook

A playbook is a YAML file containing one or more **plays**. Each play targets a group of hosts and defines a list of tasks to run against them.

### The Structure

```yaml
---                          # ← YAML document start marker (always include this)

- name: "Play name"          # ← A play begins with a dash
  hosts: cisco_ios           # ← Which inventory group or host to target
  gather_facts: false        # ← Whether to run the facts module before tasks
  connection: network_cli    # ← How to connect to devices
  become: true               # ← Enable privilege escalation (enable mode for IOS)
  become_method: enable      # ← How to escalate (enable for IOS, sudo for Linux)

  vars:                      # ← Play-level variables (optional)
    my_variable: value

  vars_files:                # ← Load variables from external files (optional)
    - vars/bgp_policy.yml

  pre_tasks:                 # ← Tasks that run before roles and main tasks
    - name: "Pre-task name"
      module_name:
        parameter: value

  tasks:                     # ← The main task list
    - name: "Task name"      # ← Always required — appears in playbook output
      module_name:           # ← The Ansible module to use
        parameter: value     # ← Module parameters (YAML key/value pairs)
      register: result_var   # ← Store the task's return value in a variable
      when: condition        # ← Only run this task if condition is true
      loop: "{{ list }}"     # ← Run this task once for each item in the list
      notify: handler_name   # ← Trigger a handler if this task reports changed
      tags:                  # ← Tag for selective playbook execution
        - tag_name

  post_tasks:                # ← Tasks that run after all other tasks
    - name: "Post-task name"
      module_name:
        parameter: value

  handlers:                  # ← Tasks triggered by notify, run once at play end
    - name: handler_name
      module_name:
        parameter: value
```

### The Six Required Fields for Every Network Play

These five fields appear in every network device play I write. Omitting any of them causes problems:

```yaml
- name: "Always a descriptive play name"   # Required for readable output
  hosts: cisco_ios                          # Required — who to run against
  gather_facts: false                       # Required — network devices can't run Python facts
  connection: network_cli                   # Required — tells Ansible to use CLI over SSH
  become: true                              # Required for IOS — needed for enable mode
  become_method: enable                     # Required for IOS — how to get to enable
```

### ### ℹ️ Info

> `gather_facts: false` is mandatory for network devices. If I leave it as the default (`true`), Ansible tries to run the `setup` module on the remote device — which requires Python to be installed and executable on that device. Cisco IOS doesn't have Python available for Ansible to run. The play fails immediately with a Python-not-found error before any of my tasks even run. Always set `gather_facts: false` for network device plays.

### Multiple Plays in One Playbook

A single playbook file can contain multiple plays, each targeting different devices:

```yaml
---
- name: "Play 1 — Configure IOS devices"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  tasks:
    - ...

- name: "Play 2 — Configure NX-OS devices"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tasks:
    - ...

- name: "Play 3 — Verify Linux hosts"
  hosts: linux_hosts
  gather_facts: true           # Linux hosts CAN gather facts
  tasks:
    - ...
```

Each play runs in sequence — Play 1 completes on all targeted hosts before Play 2 begins.

---

## 10.2 — The `network_cli` Connection Plugin

`connection: network_cli` is the engine behind all SSH-based network device automation. It's worth understanding what it actually does — not just copying it into every playbook without knowing why.

### What `network_cli` Does

When Ansible encounters a task targeting a host with `connection: network_cli`:

```
1. Opens an SSH connection to ansible_host using Paramiko
2. Authenticates with ansible_user / ansible_password (or SSH key)
3. Detects the device prompt using ansible_network_os
4. Handles login banners, pagination (--More--), and prompt patterns
5. Enters enable mode if ansible_become: true (sends 'enable' + become_password)
6. Sends the CLI command(s) from the module
7. Reads back the output until the device prompt returns
8. Returns the output to the module for parsing
9. Keeps the SSH connection alive for subsequent tasks in the same play
```

Step 9 is important — the connection persists for the entire play. Ansible doesn't open and close an SSH session for every task. This makes multi-task plays much faster than manual SSH session management.

### Setting `ansible_network_os` Correctly

`ansible_network_os` is what tells the `network_cli` plugin which terminal handler to load — how to recognize the device prompt, how to enter config mode, how to handle pagination.

```yaml
# In group_vars/cisco_ios.yml (already set from Part 7)
ansible_network_os: cisco.ios.ios

# In group_vars/cisco_nxos.yml
ansible_network_os: cisco.nxos.nxos

# In group_vars/paloalto.yml (uses httpapi, not network_cli)
ansible_network_os: paloaltonetworks.panos.panos
```

The format is always `<collection_namespace>.<collection_name>.<platform>`. Using the old short-form (`ios`, `nxos`) fails with current collection-based Ansible.

### ### 🪲 Gotcha — Mixing `connection` and `ansible_network_os` Incorrectly

```yaml
# ❌ Wrong — PAN-OS uses httpapi, not network_cli
- hosts: paloalto
  connection: network_cli    # Will fail — PAN-OS has no SSH CLI for Ansible modules
  tasks:
    - paloaltonetworks.panos.panos_op: ...

# ✅ Correct — PAN-OS uses httpapi
- hosts: paloalto
  connection: ansible.netcommon.httpapi
  tasks:
    - paloaltonetworks.panos.panos_op: ...
```

Since `connection: network_cli` is set in `group_vars/`, I don't need to specify it in the play if the `group_vars` are loaded — but being explicit in the play header is clearer and makes the playbook self-documenting.

---

## 10.3 — Building the First Playbook: Gather and Display Facts

I'll build one playbook file that grows throughout this part. I start with the simplest version — gathering facts — and add capabilities section by section.

```bash
nano ~/projects/ansible-network/playbooks/report/first_playbook.yml
```

**Version 1 — Gather and display facts:**

```yaml
---
# =============================================================
# first_playbook.yml
# A growing playbook built through Part 10.
# Run with: ansible-playbook playbooks/report/first_playbook.yml
# =============================================================

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
          - default         # hostname, version, model, serial
          - interfaces      # interface names, IPs, states
          - hardware        # memory, flash
      # ios_facts automatically populates ansible_net_* variables
      # No register needed — facts go directly into the play's variable space

    - name: "Facts | Display device summary"
      ansible.builtin.debug:
        msg:
          - "======================================"
          - "Device:    {{ inventory_hostname }}"
          - "Hostname:  {{ ansible_net_hostname }}"
          - "Version:   {{ ansible_net_version }}"
          - "Model:     {{ ansible_net_model | default('unknown') }}"
          - "Serial:    {{ ansible_net_serialnum | default('unknown') }}"
          - "Interfaces:{{ ansible_net_interfaces | length }} configured"
          - "======================================"
```

### Running It

```bash
cd ~/projects/ansible-network
ansible-playbook playbooks/report/first_playbook.yml
```

Expected output:
```yaml
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
```

### Understanding the PLAY RECAP

The play recap is the summary at the bottom of every playbook run:

```
wan-r1 : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
         │         │             │                │            │            │            │
         │         │             │                │            │            │            └── Tasks with ignore_errors: true that failed
         │         │             │                │            │            └── Tasks rescued by block/rescue
         │         │             │                │            └── Tasks skipped due to when: condition
         │         │             │                └── Tasks that returned an error
         │         │             └── Hosts unreachable via SSH
         │         └── Tasks that made a configuration change
         └── Total tasks that completed successfully (including changed)
```

A successful playbook run that made no changes shows `ok=N changed=0 failed=0`. A run that made changes shows `changed=N`. If `failed > 0` or `unreachable > 0`, something went wrong.

---

## 10.4 — Understanding `register` — Capturing Task Output

`register` saves a task's return value into a variable for use in subsequent tasks. Without `register`, the output of a task is displayed in the play recap but then discarded — I can't use it later.

### Adding `register` to the Playbook

I add a task that runs a show command and registers the output:

```yaml
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
      register: interface_brief    # ← Save the entire return dict to this variable

    - name: "Facts | Display interface brief output"
      ansible.builtin.debug:
        msg: "{{ interface_brief.stdout_lines[0] }}"
        # interface_brief is the dict returned by ios_command
        # .stdout_lines is a list — one entry per command
        # [0] is the first command's output, already split into lines
```

### The Anatomy of a `register` Variable

When I register the output of `ios_command`, the variable contains a dictionary with many keys. Here's what's inside `interface_brief` after the task runs:

```python
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
```

I access parts of this dictionary using dot notation or bracket notation in Jinja2:

```yaml
"{{ interface_brief.stdout[0] }}"           # Raw string — first command output
"{{ interface_brief.stdout_lines[0] }}"     # List of lines — first command output
"{{ interface_brief.stdout_lines[0][0] }}"  # First line of first command output
"{{ interface_brief.failed }}"              # Boolean — did the task fail?
"{{ interface_brief.changed }}"             # Boolean — did the task change anything?
```

### ### 💡 Tip

> When I'm not sure what's inside a registered variable, I use `ansible.builtin.debug` with `var:` instead of `msg:` to dump the entire structure:
> ```yaml
> - name: "Debug | Show full register structure"
>   ansible.builtin.debug:
>     var: interface_brief    # Dumps the entire dict — use during development only
> ```
> This shows me the exact structure and key names, so I know which keys to reference. I remove or comment out these debug dumps before committing to Git.

---

## 10.5 — Using `debug` Effectively

`ansible.builtin.debug` is the `print()` statement of playbooks. It outputs information during a run without making any changes. I use it constantly during development and for building reporting playbooks.

### The Two Forms

```yaml
# Form 1: msg — display a string or list of strings
- name: "Debug | Show a message"
  ansible.builtin.debug:
    msg: "The hostname is {{ ansible_net_hostname }}"

# Form 2: var — dump an entire variable's contents
- name: "Debug | Dump variable contents"
  ansible.builtin.debug:
    var: interface_brief

# Form 3: msg with a list — displays each item on its own line
- name: "Debug | Show multiple lines"
  ansible.builtin.debug:
    msg:
      - "Hostname: {{ ansible_net_hostname }}"
      - "Version:  {{ ansible_net_version }}"
      - "Uptime:   {{ ansible_net_all_ipv4_addresses | join(', ') }}"
```

### Verbosity-Gated Debug Messages

I can set a `verbosity:` level on debug tasks so they only print when I run the playbook with `-v` or higher. This lets me leave development-time debug output in the playbook without cluttering normal runs:

```yaml
- name: "Debug | Full interface_brief dump (dev only)"
  ansible.builtin.debug:
    var: interface_brief
    verbosity: 2    # Only prints when running with -vv or higher
```

Normal run: this task is silently skipped.
With `-vv`: this task prints the full variable dump.

This is how I leave "developer instrumentation" in production playbooks — it's always there for debugging but invisible during normal operation.

---

## 10.6 — Adding a Configuration Change: Push a Hostname

Now I add the first configuration-changing task to the playbook — setting the hostname to match what's in `host_vars`. This introduces the pattern of using inventory variables in configuration tasks.

```yaml
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
```

### Running the Updated Playbook

```bash
ansible-playbook playbooks/report/first_playbook.yml
```

First run (hostname was already correct):
```
TASK [Config | Set hostname from inventory] ****************************
ok: [wan-r1]      ← ok means no change needed — already correct
ok: [wan-r2]

PLAY RECAP *************************************************************
wan-r1 : ok=6    changed=0    ...
wan-r2 : ok=6    changed=0    ...
```

If I manually set a wrong hostname on `wan-r1` and run again:
```
TASK [Config | Set hostname from inventory] ****************************
changed: [wan-r1]   ← changed means it applied the correct hostname

PLAY RECAP *************************************************************
wan-r1 : ok=6    changed=1    ...
```

---

## 10.7 — Deep Dive: Idempotency

Idempotency is one of the most important concepts in Ansible. An operation is idempotent if running it multiple times produces the same result as running it once. In Ansible terms: if I run a playbook twice with the same inputs and the device is already in the desired state, the second run makes zero changes.

### Why Idempotency Matters

Without idempotency, automation is dangerous:
- Run a playbook twice → duplicate VLAN entries in the config
- Run a backup playbook twice → configuration gets applied twice, potentially corrupting state
- Scheduled automation runs a playbook every hour → device slowly accumulates duplicate lines

With idempotency, I can:
- Run a playbook any number of times safely
- Use automation as continuous enforcement (run every hour to keep devices compliant)
- Retry failed playbooks without worrying about partial-apply side effects

### Idempotent Modules — Resource Modules

**Resource modules** (like `ios_hostname`, `ios_vlans`, `ios_bgp_global`) are idempotent by design. They compare the desired state against the current state and only push changes when there's a difference.

```bash
# First run — hostname is wrong, module fixes it
ansible-playbook playbooks/report/first_playbook.yml
```

```yaml
TASK [Config | Set hostname from inventory]
changed: [wan-r1]    # ← Applied the correct hostname

PLAY RECAP: wan-r1 : ok=6  changed=1  failed=0
```

```bash
# Second run — hostname is now correct, no change needed
ansible-playbook playbooks/report/first_playbook.yml
```

```yaml
TASK [Config | Set hostname from inventory]
ok: [wan-r1]         # ← Already correct, nothing to do

PLAY RECAP: wan-r1 : ok=6  changed=0  failed=0
```

The `changed: 0` on the second run is the proof of idempotency. The module checked the current state, found it already matched the desired state, and made no changes.

### Non-Idempotent Modules — `ios_config` with Raw Lines

`ios_config` with raw configuration lines is **not reliably idempotent**. It compares the lines I want to push against the running config, but this comparison is text-based and can miss equivalent configurations expressed differently.

```yaml
# This task with ios_config
- name: "Config | Set banner (ios_config — NOT reliably idempotent)"
  cisco.ios.ios_config:
    lines:
      - "banner login ^"
      - "Authorized access only"
      - "^"
```

```bash
# First run
changed: [wan-r1]     # ← Applied the banner

# Second run — ios_config checks if the exact lines exist
# If they do, it reports ok. But sometimes the comparison fails
# and it reports changed: even when the config is identical.
changed: [wan-r1]     # ← May still report changed on second run
```

This false-positive `changed` is a known limitation of `ios_config`. Every `changed: true` in a playbook run is supposed to mean "I made a change." False positives erode trust in the output — I stop believing that `changed: 1` means something actually changed.

### The Side-by-Side Comparison

| Module Type | Example | Idempotent? | How It Checks |
|---|---|---|---|
| Resource module | `ios_vlans` | ✅ Yes | Structured state comparison |
| Resource module | `ios_hostname` | ✅ Yes | Structured state comparison |
| Resource module | `ios_bgp_global` | ✅ Yes | Structured state comparison |
| Config module | `ios_config` | ⚠️ Partial | Text line matching |
| Command module | `ios_command` | ✅ Always `ok` | Read-only, never changes |

### The Rule I Follow

> **Use resource modules whenever they exist for what I'm trying to configure. Fall back to `ios_config` only when no resource module covers it.**

Resource module coverage for IOS: `ios_vlans`, `ios_interfaces`, `ios_l2_interfaces`, `ios_l3_interfaces`, `ios_bgp_global`, `ios_bgp_address_family`, `ios_ospfv2`, `ios_ospf_interfaces`, `ios_acls`, `ios_static_routes`, `ios_ntp_global`, `ios_logging_global`, `ios_hostname`, `ios_banner`, `ios_users`, and more.

The list of resource modules grows with every collection release. When I need to configure something new, I check `ansible-doc cisco.ios.ios_<tab>` to see if a resource module exists before reaching for `ios_config`.

### Making `ios_config` More Idempotent

When `ios_config` is necessary, I use the `parents:` parameter to narrow the comparison scope, and `save_when: changed` to only save when a change was actually made:

```yaml
- name: "Config | Configure interface description (ios_config)"
  cisco.ios.ios_config:
    parents: "interface GigabitEthernet1"
    lines:
      - "description WAN | To FW-01 eth1"
    save_when: changed    # Only write memory if something changed
```

### ### 🏢 Real-World Scenario

> A network team I know had a scheduled Ansible job that ran every night at midnight and pushed "standard" configuration snippets to 300 routers using `ios_config`. The playbook was running fine — `changed` count in the 10-20 range every night, which they assumed meant a few devices had drifted from standard. Eight months later, a junior engineer noticed that several router configs had duplicate NTP server entries — the same NTP server listed five times in the running config. `ios_config` had been checking for the line and finding the config lines in the config but not recognizing they were already there due to trailing space differences, and adding them again every night. The fix was switching to `ios_ntp_global` (a resource module) which properly managed NTP state. The lesson: `changed` in a playbook should always mean something actually changed. Unexplained nightly changes are a sign of an idempotency problem.

---

## 10.8 — Adding a Backup Task

I add the backup task to the growing playbook. This reinforces the `register` pattern and introduces `delegate_to: localhost`:

```yaml
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

    - name: "Config | Set hostname from inventory"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      register: hostname_result

    - name: "Config | Report hostname change result"
      ansible.builtin.debug:
        msg: "Hostname change — changed: {{ hostname_result.changed }}"

    # ── NEW: backup tasks ─────────────────────────────────────────────
    - name: "Backup | Create backup directory on control node"
      ansible.builtin.file:
        path: "backups/cisco_ios/{{ inventory_hostname }}"
        state: directory
        mode: '0755'
      delegate_to: localhost    # ← Run this task on the Ubuntu VM, not the device

    - name: "Backup | Gather running configuration"
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: running_config  # ← Save the full running config output

    - name: "Backup | Write configuration to file on control node"
      ansible.builtin.copy:
        content: "{{ running_config.stdout[0] }}"
        dest: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: '0644'
      delegate_to: localhost    # ← Write the file on the Ubuntu VM

    - name: "Backup | Confirm backup location"
      ansible.builtin.debug:
        msg: "Backup saved: backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
```

### The `delegate_to: localhost` Pattern Explained

Every task in a play normally runs on the remote host (the network device). But creating directories and writing files needs to happen on the **control node** (the Ubuntu VM). `delegate_to: localhost` tells Ansible to run that specific task on the control node instead of the remote device:

```
Normal task flow:                With delegate_to: localhost:
Ubuntu VM                        Ubuntu VM
    │                                │
    │ SSH                            │ (runs locally, no SSH)
    ▼                                ▼
  wan-r1 ← task runs here         Ubuntu VM ← task runs here
```

The `inventory_hostname` variable still refers to `wan-r1` even when the task runs on localhost — this is what makes the backup file path `backups/cisco_ios/wan-r1/wan-r1_...cfg` correct. The variable context stays attached to the device even though the task execution moved to localhost.

---

## 10.9 — Understanding `when` Conditionals

The `when:` keyword makes a task conditional. It evaluates a Jinja2 expression and skips the task if the result is false. This is how I write playbooks that handle multiple platforms or make decisions based on device state.

### The Basic Pattern

```yaml
- name: "Task name"
  module:
    parameter: value
  when: condition_expression    # Must evaluate to true for task to run
```

### `when` Based on `ansible_network_os`

The most common use in multi-platform playbooks — run different tasks depending on which platform is being targeted:

```yaml
- name: "Config | Configure NTP (IOS-specific module)"
  cisco.ios.ios_ntp_global:
    config:
      servers:
        - server: "{{ ntp_servers[0] }}"
    state: merged
  when: ansible_network_os == 'cisco.ios.ios'

- name: "Config | Configure NTP (NX-OS-specific module)"
  cisco.nxos.nxos_ntp_global:
    config:
      servers:
        - server: "{{ ntp_servers[0] }}"
    state: merged
  when: ansible_network_os == 'cisco.nxos.nxos'
```

If I run this against a group that has both IOS and NX-OS devices, each device only runs the task that matches its platform. The other task is skipped with `skipping: [hostname]` in the output.

### `when` Based on a Registered Result

I can gate a task on the result of a previous task:

```yaml
- name: "Facts | Check if BGP is running"
  cisco.ios.ios_command:
    commands:
      - show ip bgp summary
  register: bgp_check
  ignore_errors: true    # Don't fail the play if BGP isn't running

- name: "Report | BGP is configured on this device"
  ansible.builtin.debug:
    msg: "BGP is active on {{ inventory_hostname }}"
  when: bgp_check.failed == false    # Only print if BGP command succeeded

- name: "Report | BGP is NOT configured on this device"
  ansible.builtin.debug:
    msg: "BGP is NOT running on {{ inventory_hostname }}"
  when: bgp_check.failed == true     # Only print if BGP command failed
```

### `when` Based on a Fact Variable

```yaml
# Only run on devices with a specific version or higher
- name: "Config | Apply IOS-XE 17.x specific configuration"
  cisco.ios.ios_config:
    lines:
      - "platform punt-keepalive disable-kernel-core"
  when: ansible_net_version is version('17.0', '>=')

# Only run on CSR1000V (not on physical hardware)
- name: "Config | CSR-specific tuning"
  cisco.ios.ios_config:
    lines:
      - "platform hardware throughput level MB 1000"
  when: "'CSR' in ansible_net_model"
```

### `when` with the `device_role` Inventory Variable

This is the pattern I use most often — using the `device_role` variable from `host_vars` to control which tasks run on which devices:

```yaml
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
```

### Adding `when` to the Growing Playbook

I add a conditional that only runs the hostname task when the current hostname doesn't match what's in inventory:

```yaml
    - name: "Config | Set hostname (only if it needs to change)"
      cisco.ios.ios_hostname:
        config:
          hostname: "{{ device_hostname }}"
        state: merged
      register: hostname_result
      when: ansible_net_hostname != device_hostname
      # This task is skipped entirely if the hostname is already correct
      # The resource module would handle this anyway, but the explicit
      # when: makes the intent clear and saves an unnecessary API call
```

---

## 10.10 — Looping Over Tasks with `loop` and `with_items`

Many network configuration tasks involve applying the same operation to a list of items — configuring multiple VLANs, multiple NTP servers, multiple BGP neighbors, multiple interface descriptions. Loops avoid writing repetitive tasks.

### The Modern Approach: `loop`

`loop` is the current standard. It iterates over a list, running the task once for each item. The current item is available inside the task as `{{ item }}`:

```yaml
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
```

Or loop over a variable list from inventory:

```yaml
# leaf_switches group_vars defines base_vlans as a list
# This loop reads that list and configures each VLAN

- name: "Config | Configure all base VLANs from group_vars"
  cisco.nxos.nxos_vlans:
    config:
      - vlan_id: "{{ item.id }}"
        name: "{{ item.name }}"
    state: merged
  loop: "{{ base_vlans }}"    # base_vlans is defined in group_vars/leaf_switches.yml
  loop_control:
    label: "VLAN {{ item.id }} — {{ item.name }}"    # Cleaner output (shows label instead of full dict)
```

### Loop Output

Without `loop_control.label`, each iteration shows the full `item` dict in the output — which can be verbose. With `label`, I get clean readable output:

```yaml
# Without label:
TASK [Config | Configure all base VLANs] ***
changed: [leaf-01] => (item={'id': 10, 'name': 'MGMT', 'subnet': '192.168.10.0/24'})
changed: [leaf-01] => (item={'id': 20, 'name': 'APP_SERVERS', 'subnet': '192.168.20.0/24'})

# With label: "VLAN {{ item.id }} — {{ item.name }}"
TASK [Config | Configure all base VLANs] ***
changed: [leaf-01] => (item=VLAN 10 — MGMT)
changed: [leaf-01] => (item=VLAN 20 — APP_SERVERS)
```

### Looping Over a Simple List

```yaml
# Loop over a flat list of strings
- name: "Config | Enable required NX-OS features"
  cisco.nxos.nxos_feature:
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
```

### Looping Over `bgp_neighbors` from `host_vars`

This is a realistic network automation pattern — the BGP neighbors list in `host_vars/wan-r1.yml` drives the BGP neighbor configuration:

```yaml
# host_vars/wan-r1.yml defines:
# bgp_neighbors:
#   - neighbor_ip: 10.10.10.2
#     remote_as: 65200
#     description: "eBGP to FW-01"

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
```

### The Legacy Approach: `with_items`

`with_items` is the older loop syntax. It works identically to `loop` for simple lists. I won't write new playbooks using it, but I will encounter it in existing playbooks, documentation, and Stack Overflow answers — so I need to recognize it:

```yaml
# Legacy syntax — with_items (still works, just older)
- name: "Config | Enable features (legacy with_items)"
  cisco.nxos.nxos_feature:
    feature: "{{ item }}"
    state: enabled
  with_items:
    - bgp
    - ospf
    - interface-vlan

# Modern equivalent — loop
- name: "Config | Enable features (modern loop)"
  cisco.nxos.nxos_feature:
    feature: "{{ item }}"
    state: enabled
  loop:
    - bgp
    - ospf
    - interface-vlan
```

The key differences:

| Feature | `loop` | `with_items` |
|---|---|---|
| Status | Current standard | Legacy (deprecated in spirit) |
| Nested lists | Doesn't flatten | Flattens one level automatically |
| Complex iteration | Works with `loop_control` | Limited options |
| `loop_control.label` | ✅ Supported | ❌ Not available |
| Reading old playbooks | May see `with_items` | ✅ Needs recognition |

### ### ⚠️ Warning

> One subtle difference: `with_items` automatically flattens lists one level deep, while `loop` does not. If I'm converting a `with_items` loop that passes a nested list, the behavior may change. For simple flat lists (the vast majority of network automation use cases), they're identical. For nested lists, I use `with_items` only when the flattening is intentional and document it with a comment.

---

## 10.11 — The Complete Playbook

Here is the complete `first_playbook.yml` with all concepts integrated:

```yaml
---
# =============================================================
# first_playbook.yml — Complete version from Part 10
# Demonstrates: facts, register, debug, idempotent config,
# backup, when conditionals, and loop.
#
# Run: ansible-playbook playbooks/report/first_playbook.yml
# Dry run: ansible-playbook playbooks/report/first_playbook.yml --check
# Single device: ansible-playbook playbooks/report/first_playbook.yml -l wan-r1
# =============================================================

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
      loop: "{{ ntp_servers }}"    # ntp_servers defined in group_vars/all.yml
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
```

### Running the Complete Playbook

```bash
# Full run
ansible-playbook playbooks/report/first_playbook.yml

# Limit to one device
ansible-playbook playbooks/report/first_playbook.yml -l wan-r1

# See what would change without applying
ansible-playbook playbooks/report/first_playbook.yml --check --diff

# Run only specific tagged sections
ansible-playbook playbooks/report/first_playbook.yml --tags backup

# Show developer debug output
ansible-playbook playbooks/report/first_playbook.yml -vv
```

---

## 10.12 — Common Gotchas in This Section

### ### 🪲 Gotcha — `when` on a registered variable that was skipped

If a task was skipped (because its own `when:` condition was false), its registered variable is in a special `skipped` state. Accessing attributes on it causes errors:

```yaml
- name: "Task A"
  cisco.ios.ios_hostname:
    config:
      hostname: "{{ device_hostname }}"
    state: merged
  register: hostname_result
  when: ansible_net_hostname != device_hostname

# ❌ This errors if Task A was skipped — hostname_result.changed doesn't exist
- name: "Task B"
  ansible.builtin.debug:
    msg: "Changed: {{ hostname_result.changed }}"

# ✅ Guard with 'is not skipped' check
- name: "Task B"
  ansible.builtin.debug:
    msg: "Changed: {{ hostname_result.changed }}"
  when: hostname_result is not skipped
```

### ### 🪲 Gotcha — `loop` with `ios_ntp_global` adds instead of replacing

Resource modules with `state: merged` add to existing configuration — they don't remove what's already there. If I loop over NTP servers and run the playbook twice with different server lists, the old servers remain:

```yaml
# First run with servers: [A, B]  → config has A, B
# Second run with servers: [A, C] → config has A, B, C (B was not removed!)
```

The fix is `state: replaced` (replaces the entire NTP config) or `state: overridden` (replaces the global config). Understanding `merged` vs `replaced` vs `overridden` is covered fully in Part 15.

### ### 🪲 Gotcha — `loop` variable name conflicts

Inside a loop, `item` is the current iteration variable. If a task outside the loop also uses `item` (perhaps from a `set_fact` or a Jinja2 expression), there can be a naming conflict. I always use descriptive variable names in `set_fact` and never name a variable `item`.

### ### 🪲 Gotcha — `gather_facts: false` but trying to use `ansible_net_*` variables

`ios_facts` populates `ansible_net_*` variables, but `gather_facts: true/false` controls the built-in `setup` module (for Linux hosts). These are separate mechanisms:

```yaml
# ❌ This doesn't gather ansible_net_* variables
- hosts: cisco_ios
  gather_facts: true    # This runs setup — which fails on IOS
  tasks:
    - debug: var=ansible_net_hostname    # ansible_net_hostname doesn't exist yet

# ✅ Correct approach
- hosts: cisco_ios
  gather_facts: false    # Don't run setup
  tasks:
    - cisco.ios.ios_facts:    # This populates ansible_net_* variables
        gather_subset: default
    - debug: var=ansible_net_hostname    # Now it exists
```

---


The core playbook concepts are now in place — play anatomy, register, debug, idempotency, when conditionals, and loops. These patterns appear in every playbook from here on. Part 11 introduces Jinja2 templating — the mechanism that turns variable data from inventory into actual device configuration text.


