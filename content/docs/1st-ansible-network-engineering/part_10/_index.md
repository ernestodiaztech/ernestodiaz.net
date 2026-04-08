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


## Playbooks

A playbook is a YAML file containing 1 of more "plays". Each "play" targets a group of hosts and defines a list of tasks to run against them.

#### The Structure

```yaml {linenos=table}
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
```

- **Line 1** - YAML document start marker.
- **Line 3** - A play begins with a dash
- **Line 4** - Which inventory group or host to target
- **Line 5** - Whether to run the facts module before tasks
- **Line 6** - How to connect to devices
- **Line 7** - Enable privilege escalation (`enable mode` for IOS)
- **Line 8** - How to escalate (`enable` for IOS, `sudo` for Linux)
- **Line 10** - Play level variables (optional)
- **Line 13** - Load variables from external files (optional)
- **Line 16** - Tasks that run before roles and main tasks
- **Line 21** - The main task list
- **Line 22** - Always required (appears in playbook output)
- **Line 23** - The Ansible module to use
- **Line 24** - Module parameters
- **Line 25** - Store the task's return value in a variable
- **Line 26** - Only run thistask if condition is true
- **Line 27** - Run this task once for each item in the list
- **Line 28** - Trigger a handler if this task reports changed
- **Line 29** - Tag for selective playbook execution

---

**The Required Fields**

These 6 fields appear in every network device playbook I write.

- `- name: "Name"` - Required for readable output
- `hosts: cisco_ios` - Required to state who to run against
- `gather_facts: false` - Required because network devices can't run Python facts
- `connection: network_cli` - Required because this tells Ansible to use CLI over SSH
- `become: true` - Required for IOS so that Ansible can enter 'enable mode'
- `become_method: enable` - Required for IOS since it's how Ansible gets to 'enable`

---

**Multiple Plays**

A single playbook file can contain multiple plays, each targeting different devices.

```yaml
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
```

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

```yaml {filename="playbooks/report/first_playbook.yml"}
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
```

---

Running it:

```bash
cd ~/projects/ansible-network
ansible-playbook playbooks/report/first_playbook.yml
```

Expected output:

```
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

---

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

---

## Capturing Task Output

`register` saves a task's return value into a variable for use in subsequent tasks. Without `register` the output of a task is displayed in the play recap but then discarded.

---

#### Adding Register

I add a task that runs a show command and registers the output:

