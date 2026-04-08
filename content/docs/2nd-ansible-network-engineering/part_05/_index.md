---
draft: false
title: '5. Playbooks'
description: "Part 5 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 5
---

{{< badge "Ansible" >}}
{{< badge content="Git" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

Here I will create a static inventory, (for now), go over read-only show commands, fact gathering, configuration pushes, and create a structured config backup playbook.

{{< objectives title="What I Will Be Completing In This Part" >}}
- Create a static YAML inventory file with `wan-r1` and `wan-r2`
- Verify connenctivity from Ansible using an ad-hoc ping
- Write and run a read-only playbook using `ios_command`
- Gather structured device facts using `ios_facts`
- Push a configuration change using `ios_config` and observe idempotency
- Save the running configuration to startup
- Build a structured backup playbook that stores configs locally with timestamps
{{< /objectives >}}

---

## <span class="section-num">01</span> Static Inventory

I decided on YAML when creating the static inventory file. This file defines the hosts and groups that Ansible targets when running playbooks. 

{{< codeblock file="inventory/hosts.yml" syntax="yaml" lines="true" >}}
---
all:
  children:
    ios:
      hosts:
        wan-r1:
          ansible_host: 172.20.20.11
        wan-r2:
          ansible_host: 172.20.20.12
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
`all` is the top-level implicit group that contains every host.

Lines 4-5:
: Any host here inherits the variables from `inventory/group_vars/ios/vars.yml` and `inventory/group_vars/ios/vault.yml` automatically.

Lines 7, 9:
: `ansible_host` tells Ansible the actual IP address to connect to.
{{< /line-explain >}}

{{< lab-callout type="info" >}}
This static inventory is temporary. Once I setup NetBox I'll also configure the dynamic inventory plugin, which generates the inventory automatically from NetBox's device database.
{{< /lab-callout >}}

I then verified Ansible can pars the inventory file before running playbooks:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-inventory --list -y
{{< /codeblock >}}

This will print the full inventory as Ansible sees it. 

---

### <span class="section-num">01a</span> Connectivity Test

I confirmed that Ansible can reach both devices using an ad-hoc command.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible ios -m cisco.ios.ios_command -a "commands='show version | include uptime'"
{{< /codeblock >}}

{{< line-explain >}}
ios:
: The target group.

-m cisco.ios.ios_command:
: The module to run.

-a "commands=":
: The module arguments.
{{< /line-explain >}}

{{< codeblock lang="Expected Output" copy="false" >}}
wan-r1 | SUCCESS => {
    "changed": false,
    "stdout": [
        "wan-r1 uptime is 2 hours, 15 minutes"
    ],
    "stdout_lines": [
        [
            "wan-r1 uptime is 2 hours, 15 minutes"
        ]
    ]
}
wan-r2 | SUCCESS => {
    "changed": false,
    "stdout": [
        "wan-r2 uptime is 2 hours, 14 minutes"
    ],
    "stdout_lines": [
        [
            "wan-r2 uptime is 2 hours, 14 minutes"
        ]
    ]
}
{{< /codeblock >}}

---

## <span class="section-num">02</span> Playbook

### <span class="section-num">02a</span> Show Commands

I wrote a read-only playbook that runs several show commands and displays the output.

{{< codeblock file="playbooks/ios_show.yml" syntax="yaml" lines="true" >}}
---
- name: IOS-XE show commands
  hosts: ios
  gather_facts: false

  tasks:
    - name: Run show commands
      cisco.ios.ios_command:
        commands:
          - show  version | include Software
          - show ip interface brief
          - show running-config | section line vty
      register: show_output

    - name: Display output
      ansible.builtin.debug:
        msg: "{{ show_output.stdout_lines }}"
{{< /codeblock >}}

{{< line-explain >}}
Line 3:
: Targets every device in the `ios` inventory group.

Line 4:
: Disables Ansible's default fact-gathering step.

Lines 9-12:
: These commands run sequentially on each device and the output is collected in a list.

Line 13:
: Captures the module's return data into a variable.

Lines 15-17:
: Prints the registered variable to the Ansible output.
{{< /line-explain >}}

Then I ran the playbook:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-playbook playbooks/ios_show.yml
{{< /codeblock >}}

The results will be formatted in YAML.

---

### <span class="section-num">02b</span> Gathering Facts

The `ios_facts` module collects structured data about devices. The data is returned as Ansible facts, which means it's accessible as variables in subsequent tasks without needing to parse show command output manually.

{{< codeblock file="playbooks/ios_facts.yml" syntax="yaml" lines="true" >}}
---
- name: Gather IOS-XE device facts
  hosts: ios
  gather_facts: false

  tasks:
    - name: Collect all facts
      cisco.ios.ios_facts:
        gather_subset:
          - all
      register: facts_output

    - name: Display hostname and version
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ ansible_net_hostname }} running {{ ansible_net_version }}"

    - name: Display all interfaces
      ansible.builtin.debug:
        msg: "{{ ansible_net_interfaces | to_nice_yaml }}"
{{< /codeblock >}}

{{< line-explain >}}
Lines 9-10:
: This tells the module to collect every category of facts.

Line 15:
: After `ios_facts` runs it populates Ansible facts as variables prefixed with `ansible_net_`.

Line 19:
: Is a dictionary containing every interface on the device with its operational state, IP addresses, MTU, speed, and more.
{{< /line-explain >}}

Then tested the playbook:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-playbook playbooks/ios_facts.yml
{{< /codeblock >}}

---

### <span class="section-num">03b</span> Pushing Configuration

I used `ios_config` to push a simple change: adding a login banner and disabling DNS lookups.

{{< codeblock file="playbooks/ios_base_config.yml" syntax="yaml" lines="true" >}}
---
- name: Apply base confiugration to IOS-XE devices
  hosts: ios
  gather_facts: false

  tasks:
    - name: Disable DNS lookups
      cisco.ios.ios_config:
        lines:
          - no ip domain-lookup

    - name: Set login banner
      cisco.ios.ios_config:
        lines:
          - banner motd ^Unauthorized access is prohibited.^
      
    - name: Set logging and service timestamps
      cisco.ios.ios_config:
        lines:
          - service timestamps debug datetime msec localtime
          - service timestamps log datetime msec localtime
          - logging buffered 16384 informational
{{< /codeblock >}}

Then ran the playbook:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-playbook playbooks/ios_base_config.yml
{{< /codeblock >}}

The 1st run should show `changed=3` for each device. To check idempotency I ran the playbook again, which it should show `changed=0`.

---

### <span class="section-num">03c</span> Saving Configuration

Next, I wrote a playbook that saves the running configuration to startup.

{{< codeblock file="playbooks/ios_save.yml" syntax="yaml" lines="true" >}}
---
- name: Save running config to startup
  hosts: ios
  gather_facts: false

  tasks:
    - name: Save configuration
      cisco.ios.ios_config:
        save_when: always
{{< /codeblock >}}

{{< line-explain >}}
Line 9:
: This triggers a `write memory` every time this task runs.
{{< /line-explain >}}

I then tested the playbook:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-playbook playbooks/ios_save.yml
{{< /codeblock >}}

---

### <span class="section-num">03d</span> Structured Backup

I then made a playbook that retrieves the full running configuration from each device and saves it to a local file on the control node, with a timestamp. This is the manual way of doing it, later on I'll setup Oxidized which does it automatically.

{{< codeblock file="playbooks/ios_backup.yml" syntax="yaml" lines="true" >}}
---
- name: Backup IOS-XE running configuration
  hosts: ios
  gather_facts: false

  vars:
    backup_root: "{{ playbook_dir }}/../backups"
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d-%H%M%S') }}"

  tasks:
    - name: Create backup directory
      ansible.builtin.file:
        path: "{{ backup_root }}/{{ inventory_hostname }}"
        state: directory
        mode: "0755"
      delegate_to: localhost
      run_once: false

    - name: Retrieve running configuration
      cisco.ios.ios_command:
        commands:
          - show running-config
      register: running_config

    - name: Save configuration to file
      ansible.builtin.copy:
        content: "{{ running_config.stdout[0] }}"
        dest: "{{ backup_root }}/{{ inventory_hostname }}/{{ inventory_hostname }}_{{ timestamp }}.cfg"
        mode: "0644"
      delegate_to: localhost

    - name: Save a latest copy (overwritten each run)
      ansible.builtin.copy:
        content: "{{ running_config.stdout[0] }}"
        dest: "{{ backup_root }}/{{ inventory_hostname }}/{{ inventory_hostname }}_latest.cfg"
        mode: "0644"
      delegate_to: localhost
{{< /codeblock >}}

{{< line-explain >}}
Line 7:
: `backup_root` resolves to a `backups/` directory in the project root. 

Line 8:
: This plugin runs a shell command on the control node and captures the output.

Lines 11-17:
: Creates a per-device subdirectory under `backups/`.

Lines 19-23:
: Retrieves the full running config.

Lines 25-30:
: Writes the timestamped backup file.

Lines 32-37:
: Writes a `_latest.cfg` copy that is overwritten on every run.
{{< /line-explain >}}

Then ran the playbook:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ansible-playbook playbooks/ios_backup.yml
{{< /codeblock >}}

---

## <span class="section-num">04</span> Updating .gitignore

The `backups/` directory contains device running configurations which may include sensitive information. I added it to the `.gitignore` so backup files are not committed to the repo.

{{< codeblock file=".gitignore" syntax="ini" >}}
backups/
{{< /codeblock >}}

---

## <span class="section-num">05</span> Commit and Push

Lastly, I committed the inventory file, all four playbooks, and updated the `.gitignore` using the feature branch workflow:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout -b feat/first-playbooks
(.venv) $ git add -A
(.venv) $ git status
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
modified:   .gitignore
new file:   inventory/hosts.yml
new file:   playbooks/ios_show.yml
new file:   playbooks/ios_facts.yml
new file:   playbooks/ios_base_config.yml
new file:   playbooks/ios_save.yml
new file:   playbooks/ios_backup.yml
{{< /codeblock >}}

I committed and pushed:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git commit -m "feat: add static inventory and first IOS-XE playbooks"
(.venv) $ git push -u origin feat/first-playbooks
{{< /codeblock >}}

I then went to Gitea and approved and merged.

After merging:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout main
(.venv) $ git pull origin main
{{< /codeblock >}}

---

I now have a working static inventory with 2 Cat9kv nodes in the `ios` group, 5 playbooks covering the core Ansible network automation.