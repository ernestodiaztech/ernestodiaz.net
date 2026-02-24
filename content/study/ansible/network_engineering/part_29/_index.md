---
draft: true
title: '29 - Advanced Techniques'
weight: 29
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 29: Advanced Playbook Techniques

> *The playbooks written through Parts 17-28 work correctly, but they're written in a style that fits well for a single engineer learning the tooling. This part introduces the techniques that make playbooks scale — modular task files that can be reused without copying, dynamic execution patterns that adapt at runtime, delegation that runs some tasks on the control node while others run on devices, and execution strategies that control how Ansible distributes work across a fleet. These aren't exotic features; they're the standard toolkit for playbooks that survive contact with real infrastructure.*

---

## 29.1 — The Five Most Useful Techniques for Network Automation

Before diving in, here's the priority ordering that shapes this part:

```
1. delegate_to + run_once
   Why: Essential for API calls, report generation, and Netbox updates that
   run on localhost while the play targets network devices. Used in almost
   every multi-host playbook in this guide already — worth understanding deeply.

2. include_tasks vs import_tasks
   Why: The difference causes subtle bugs with tags and loops. Understanding
   it prevents hours of debugging.

3. serial + max_fail_percentage
   Why: Controls blast radius on rolling updates. Critical for password
   rotation, software upgrades, and any change that must not hit all devices
   simultaneously.

4. set_fact from gathered data
   Why: Transforms raw show command output into structured variables that
   later tasks can act on. The bridge between 'get data' and 'use data'.

5. pre_tasks / post_tasks
   Why: Clean separation of setup/teardown from the main play logic.
   Pre-tasks run before roles; post-tasks run regardless of play outcome
   when combined with always tags.
```

The remaining techniques — `include_role`/`import_role`, `strategy: free`, and callbacks — are covered in dedicated reference sections at the end.

---

## 29.2 — The Combined Example Playbook

All five primary techniques are demonstrated in a single realistic playbook: a rolling IOS software upgrade checker that gathers current versions, compares against a target, delegates report generation to localhost, and applies changes one device at a time with verification.

This is a deliberately complex playbook — each section is annotated to show exactly which technique is being used and why.

```bash
cat > ~/projects/ansible-network/playbooks/advanced/rolling_upgrade_check.yml << 'EOF'
---
# =============================================================
# rolling_upgrade_check.yml — Advanced techniques demonstration
# Performs a rolling IOS version check and optional upgrade prep
#
# Techniques demonstrated:
#   29.3 — delegate_to + run_once  (report generation, API calls)
#   29.4 — include_tasks vs import_tasks  (platform branching)
#   29.5 — serial + max_fail_percentage  (rolling execution)
#   29.6 — set_fact from gathered data  (version parsing)
#   29.7 — pre_tasks / post_tasks  (setup and cleanup)
#
# Usage:
#   ansible-playbook rolling_upgrade_check.yml
#   ansible-playbook rolling_upgrade_check.yml -e "apply_upgrade=true"
# =============================================================

- name: "Upgrade Check | Rolling IOS version audit and upgrade prep"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  serial: 2                      # [TECHNIQUE: serial] Process 2 devices at a time
  max_fail_percentage: 25        # Stop if more than 25% of devices fail

  vars:
    target_ios_version: "17.09.04a"
    upgrade_image: "c8000v-universalk9.17.09.04a.SPA.bin"
    report_dir: "reports/upgrade"
    report_timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
    apply_upgrade: false         # Gate — set true via -e to actually stage files

  # ════════════════════════════════════════════════════════════
  # PRE_TASKS: Run before roles and main tasks
  # These execute on every host in the serial batch before any
  # main tasks run. If a pre_task fails, the play stops for
  # that host — no main tasks execute.
  # ════════════════════════════════════════════════════════════
  pre_tasks:                     # [TECHNIQUE: pre_tasks]

    - name: "Pre | Create report directory on localhost"
      ansible.builtin.file:
        path: "{{ report_dir }}"
        state: directory
        mode: '0755'
      delegate_to: localhost     # [TECHNIQUE: delegate_to] — runs on control node
      run_once: true             # [TECHNIQUE: run_once] — runs once for the whole play,
                                 # not once per host in the serial batch
      # Without run_once, this would run 2x per batch (once per host in serial: 2)
      # With run_once, it runs exactly once regardless of serial size or host count

    - name: "Pre | Verify device is reachable before starting"
      cisco.ios.ios_facts:
        gather_subset: [default]
      register: pre_facts
      # If this fails (device unreachable), the host is marked failed
      # max_fail_percentage: 25 determines whether the whole play continues

    - name: "Pre | Log start of upgrade check"
      ansible.builtin.lineinfile:
        path: "{{ report_dir }}/upgrade_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | START | {{ inventory_hostname }} | current={{ pre_facts.ansible_facts.ansible_net_version }}"
        create: true
        mode: '0644'
      delegate_to: localhost     # Log write always happens on control node

  # ════════════════════════════════════════════════════════════
  # MAIN TASKS
  # ════════════════════════════════════════════════════════════
  tasks:

    # ── Step 1: Gather and parse current version ──────────────
    - name: "Facts | Gather detailed IOS version information"
      cisco.ios.ios_command:
        commands:
          - show version
          - show platform
          - show inventory
      register: ios_version_raw
      changed_when: false

    - name: "Facts | Parse version into structured variables"
      ansible.builtin.set_fact:                      # [TECHNIQUE: set_fact]
        ios_current_version: >-
          {{ ios_version_raw.stdout[0]
             | regex_search('Version (\S+),', '\\1')
             | first | default('unknown') }}
        ios_running_image: >-
          {{ ios_version_raw.stdout[0]
             | regex_search('System image file is "(\S+)"', '\\1')
             | first | default('unknown') }}
        ios_uptime: >-
          {{ ios_version_raw.stdout[0]
             | regex_search('uptime is (.+)', '\\1')
             | first | default('unknown') }}
        ios_platform: >-
          {{ ios_version_raw.stdout[0]
             | regex_search('Cisco (\S+) .* processor', '\\1')
             | first | default('unknown') }}
        upgrade_needed: >-
          {{ ios_version_raw.stdout[0]
             | regex_search('Version (\S+),', '\\1')
             | first | default('unknown') != target_ios_version }}
    # set_fact creates new host variables from processed data
    # ios_current_version, upgrade_needed etc. are now available
    # to all subsequent tasks on this host as regular variables

    - name: "Facts | Display parsed version data"
      ansible.builtin.debug:
        msg:
          - "Host:            {{ inventory_hostname }}"
          - "Current version: {{ ios_current_version }}"
          - "Target version:  {{ target_ios_version }}"
          - "Upgrade needed:  {{ upgrade_needed }}"
          - "Running image:   {{ ios_running_image }}"
          - "Uptime:          {{ ios_uptime }}"

    # ── Step 2: Platform-specific checks (include vs import) ──
    - name: "Checks | Run platform-specific storage check"
      ansible.builtin.include_tasks:                 # [TECHNIQUE: include_tasks]
        file: "tasks/check_storage_{{ ios_platform | lower | replace(' ', '_') }}.yml"
      when: upgrade_needed | bool
      # include_tasks is evaluated at RUNTIME — the filename uses a variable
      # (ios_platform) that wasn't known at parse time.
      # import_tasks would FAIL here because it's evaluated at parse time,
      # before ios_platform has been set by the set_fact above.
      #
      # Rule: use include_tasks when the filename or the decision to include
      # depends on a variable set during play execution.
      # Use import_tasks when the file is always the same and known at parse time.

    # ── Step 3: Accumulate results for the report ─────────────
    - name: "Report | Append host result to shared fact"
      ansible.builtin.set_fact:
        upgrade_report_entry:
          device: "{{ inventory_hostname }}"
          platform: "{{ ios_platform }}"
          current_version: "{{ ios_current_version }}"
          target_version: "{{ target_ios_version }}"
          upgrade_needed: "{{ upgrade_needed }}"
          running_image: "{{ ios_running_image }}"
          uptime: "{{ ios_uptime }}"
          checked_at: "{{ lookup('pipe', 'date --iso-8601=seconds') }}"
      # This set_fact creates a per-host variable.
      # To aggregate across all hosts, we use delegate_to + hostvars in post_tasks.

    # ── Step 4: Conditional upgrade staging ───────────────────
    - name: "Upgrade | Stage upgrade image (if apply_upgrade and upgrade_needed)"
      ansible.builtin.include_tasks:
        file: tasks/stage_upgrade_image.yml
      when:
        - apply_upgrade | bool
        - upgrade_needed | bool
      # include_tasks here is also correct — the decision to include depends on
      # runtime variables (apply_upgrade, upgrade_needed set by set_fact above)

  # ════════════════════════════════════════════════════════════
  # POST_TASKS: Run after all main tasks and roles
  # With 'tags: always', post_tasks run even if main tasks fail
  # This guarantees cleanup and report generation always happen
  # ════════════════════════════════════════════════════════════
  post_tasks:                    # [TECHNIQUE: post_tasks]

    - name: "Post | Log completion for this host"
      ansible.builtin.lineinfile:
        path: "{{ report_dir }}/upgrade_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | END   | {{ inventory_hostname }} | {{ 'UPGRADE_NEEDED' if upgrade_needed | bool else 'CURRENT' }}"
        create: true
        mode: '0644'
      delegate_to: localhost
      tags: always               # always tag ensures this runs even if main tasks fail

    - name: "Post | Generate consolidated report when all hosts done"
      ansible.builtin.copy:
        content: |
          IOS Upgrade Readiness Report
          =============================
          Generated: {{ lookup('pipe', 'date') }}
          Target version: {{ target_ios_version }}

          {% for host in ansible_play_hosts_all %}
          {% set entry = hostvars[host].upgrade_report_entry | default({}) %}
          {{ '%-20s' | format(host) }}
            Current:  {{ entry.current_version | default('N/A') }}
            Status:   {{ 'UPGRADE NEEDED' if entry.upgrade_needed | default(false) | bool else 'CURRENT' }}
            Uptime:   {{ entry.uptime | default('N/A') }}
          {% endfor %}

          Summary:
            Total devices:    {{ ansible_play_hosts_all | length }}
            Need upgrade:     {{ ansible_play_hosts_all | map('extract', hostvars, ['upgrade_report_entry', 'upgrade_needed']) | select('bool') | list | length }}
            Already current:  {{ ansible_play_hosts_all | map('extract', hostvars, ['upgrade_report_entry', 'upgrade_needed']) | reject('bool') | list | length }}
        dest: "{{ report_dir }}/upgrade_report_{{ report_timestamp }}.txt"
        mode: '0644'
      delegate_to: localhost     # Report written to control node filesystem
      run_once: true             # Run once after ALL hosts (not once per host)
      tags: always
      # delegate_to: localhost + run_once is the standard pattern for any task
      # that should happen once at the end of a multi-host play:
      #   - API calls (update Netbox, create ITSM ticket)
      #   - Report generation
      #   - Sending notifications
      # hostvars[host] lets the single delegated task access variables from ALL hosts
EOF
```

### The Task File Referenced by include_tasks

```bash
mkdir -p ~/projects/ansible-network/playbooks/advanced/tasks

cat > ~/projects/ansible-network/playbooks/advanced/tasks/check_storage_csr1000v.yml << 'EOF'
---
# check_storage_csr1000v.yml — Storage check for CSR1000v platform
# Included dynamically via include_tasks when ios_platform == 'CSR1000V'

- name: "Storage | Check flash free space on CSR1000v"
  cisco.ios.ios_command:
    commands:
      - show disk0: | include bytes free
  register: storage_check
  changed_when: false

- name: "Storage | Parse free space"
  ansible.builtin.set_fact:
    flash_free_bytes: >-
      {{ storage_check.stdout[0]
         | regex_search('(\d+) bytes free', '\\1')
         | first | default('0') | int }}

- name: "Storage | Assert sufficient space for upgrade image"
  ansible.builtin.assert:
    that:
      - flash_free_bytes > 500000000    # 500MB minimum
    fail_msg: >
      Insufficient flash space on {{ inventory_hostname }}.
      Available: {{ flash_free_bytes | filesizeformat }}
      Required:  500MB minimum
    success_msg: >
      PASS: {{ flash_free_bytes | filesizeformat }} free on flash
EOF
```

---

## 29.3 — delegate_to and run_once in Depth

`delegate_to` changes where a task executes without changing which host it's logically targeted at. The task's variables (hostvars, inventory variables) still come from the targeted host — only the execution location changes.

```yaml
# The pattern that appears throughout this guide:
- name: "Report | Write JSON report"
  ansible.builtin.copy:
    content: "{{ results | to_nice_json }}"
    dest: "reports/validation_{{ inventory_hostname }}.json"
    mode: '0644'
  delegate_to: localhost
# 'results' and 'inventory_hostname' are from the targeted device
# The file is written on the control node (localhost)
# Without delegate_to, this would try to write a file on the network device
```

### delegate_to for API Calls

Any module that makes an HTTP/API call — `netbox.netbox.*`, `uri`, `ansible.builtin.uri` — must run on a host that has network access to the API and the required Python libraries. Network devices don't. The control node does.

```yaml
- name: "Netbox | Update device status"
  netbox.netbox.netbox_device:
    netbox_url: "{{ netbox_url }}"
    netbox_token: "{{ netbox_token }}"
    data:
      name: "{{ inventory_hostname }}"   # inventory_hostname from the device
      status: active
    state: present
  delegate_to: localhost    # API call runs on control node
  # inventory_hostname is still the network device — Netbox gets the right device name
```

### run_once Behaviour with serial

`run_once` runs a task exactly once across the entire play — but when combined with `serial`, the behaviour depends on when the task appears:

```yaml
serial: 2
# Hosts: [wan-r1, wan-r2, wan-r3, wan-r4]
# Batch 1: [wan-r1, wan-r2]
# Batch 2: [wan-r3, wan-r4]

tasks:
  - name: "This runs once — on wan-r1 in batch 1"
    ansible.builtin.debug:
      msg: "Once for the whole play"
    run_once: true
    # Runs during Batch 1 execution, never again in Batch 2

post_tasks:
  - name: "This also runs once — but after ALL batches"
    ansible.builtin.copy:
      content: "..."
      dest: "report.txt"
    delegate_to: localhost
    run_once: true
    tags: always
    # In post_tasks with tags: always, run_once fires after the last batch
    # This is the correct place for final report generation
```

### Aggregating Data from All Hosts in a Delegated Task

The `hostvars` magic variable gives any task access to variables from any host:

```yaml
# This runs on localhost once, but reads variables from every device
- name: "Report | Consolidated report across all devices"
  ansible.builtin.copy:
    content: |
      {% for host in ansible_play_hosts_all %}
      {{ host }}: {{ hostvars[host].ios_current_version | default('N/A') }}
      {% endfor %}
    dest: "reports/version_summary.txt"
  delegate_to: localhost
  run_once: true
  # ansible_play_hosts_all — all hosts targeted by the play (including failed ones)
  # ansible_play_hosts    — only hosts that haven't failed yet
  # Choose based on whether you want to include failed hosts in the report
```

---

## 29.4 — include_tasks vs import_tasks

The single most important thing to understand about this distinction:

```
import_tasks  — Static.  Processed at PARSE TIME, before the play runs.
include_tasks — Dynamic. Processed at RUNTIME, when the task is reached.
```

### When the Difference Matters

**Situation 1: Filename uses a variable set during play**

```yaml
# set_fact runs first, sets ios_platform
- name: "Set platform fact"
  ansible.builtin.set_fact:
    ios_platform: "CSR1000V"

# ❌ FAILS: import_tasks is parsed before set_fact runs
# ios_platform doesn't exist at parse time
- ansible.builtin.import_tasks: "tasks/check_{{ ios_platform }}.yml"

# ✅ WORKS: include_tasks is evaluated at runtime
# ios_platform exists by the time this task is reached
- ansible.builtin.include_tasks: "tasks/check_{{ ios_platform }}.yml"
```

**Situation 2: Tags on the including task**

```yaml
# import_tasks: tags on the import task apply to ALL tasks inside the file
- ansible.builtin.import_tasks: tasks/ntp.yml
  tags: ntp
  # Running --tags ntp runs every task inside ntp.yml ✓

# include_tasks: tags on the include task do NOT propagate inside
- ansible.builtin.include_tasks: tasks/ntp.yml
  tags: ntp
  # Running --tags ntp runs the include_tasks task itself,
  # but tasks inside ntp.yml only run if they have their own tags
  # or no tags at all ✗ (common confusion)

# Fix for include_tasks + tags: apply tags inside the task file itself
# Or use 'apply' to push tags into included tasks:
- ansible.builtin.include_tasks:
    file: tasks/ntp.yml
    apply:
      tags: ntp      # Pushes 'ntp' tag into all tasks inside the file
  tags: ntp
```

**Situation 3: Conditionals with loops**

```yaml
# import_tasks: the when condition applies to each task inside (evaluated per task)
- ansible.builtin.import_tasks: tasks/ospf.yml
  when: ospf is defined
  # The 'when' is copied to every task inside ospf.yml
  # Each task evaluates 'ospf is defined' independently ✓

# include_tasks: the when condition determines whether to include the file at all
- ansible.builtin.include_tasks: tasks/ospf.yml
  when: ospf is defined
  # If ospf is not defined, the entire file is skipped — no tasks run ✓
  # This is actually often what you want for network automation
```

### Decision Guide

```
Use import_tasks when:
  ✓ Filename is a literal string (not a variable)
  ✓ You want --tags to work correctly with tasks inside the file
  ✓ The file should always be included (condition, if any, is on individual tasks)
  ✓ You want syntax checking of the included file at parse time

Use include_tasks when:
  ✓ Filename is constructed from a variable (platform, environment, role)
  ✓ The decision to include the file depends on a runtime variable
  ✓ The include is inside a loop (each iteration can include a different file)
  ✓ You want the entire file skipped when a condition is false
```

---

## 29.5 — serial and max_fail_percentage

`serial` controls how many hosts Ansible processes in parallel within a play. Without it, Ansible runs all hosts simultaneously — task 1 runs on all hosts, then task 2, and so on. With `serial: 2`, Ansible runs the full play on hosts 1-2, then hosts 3-4, and so on.

```
Without serial (default: all hosts parallel):
  Task 1: [r1, r2, r3, r4] simultaneously
  Task 2: [r1, r2, r3, r4] simultaneously
  Task 3: [r1, r2, r3, r4] simultaneously
  ↑ A bug in Task 2 affects all 4 devices at once

With serial: 2:
  Batch 1: Task 1→2→3 on [r1, r2]
  Batch 2: Task 1→2→3 on [r3, r4]  (only if Batch 1 succeeded)
  ↑ A bug in Task 2 affects 2 devices, not all 4
```

### Serial Values

```yaml
serial: 1          # Strictly one device at a time (password rotation, upgrades)
serial: 2          # Two at a time (rolling update with some parallelism)
serial: "25%"      # 25% of inventory at a time (scales with fleet size)
serial: [1, 5, 10] # Progressive: 1 first, then 5, then 10 (canary pattern)
```

The progressive serial pattern is the safest approach for deployments to large fleets:

```yaml
serial: [1, "10%", "25%"]
# Batch 1: 1 device (canary — verify the change works)
# Batch 2: 10% of remaining devices
# Batch 3: 25% of remaining devices
# Batch 4+: 25% each until done
# If any batch has failures exceeding max_fail_percentage, play stops
```

### max_fail_percentage

```yaml
serial: 2
max_fail_percentage: 0    # Stop immediately if ANY device fails
                          # Use for password rotation, destructive changes

max_fail_percentage: 25   # Stop if more than 25% fail in a batch
                          # Use for configuration deploys with expected partial success

max_fail_percentage: 100  # Never stop on failures (collect all results)
                          # Use for read-only audits and fact gathering
```

### Practical Combinations by Use Case

```yaml
# Password rotation — one at a time, stop on any failure
serial: 1
max_fail_percentage: 0

# Config deploy to fleet — progressive, tolerate some failures
serial: [1, "10%", "50%"]
max_fail_percentage: 20

# Fleet-wide audit — gather all results, never stop
serial: 0    # 0 = all hosts (default, explicit for clarity)
max_fail_percentage: 100
ignore_errors: true    # Ensure failed hosts don't stop remaining tasks

# Software upgrade — one at a time, never tolerate failures
serial: 1
max_fail_percentage: 0
```

---

## 29.6 — set_fact from Gathered Data

`set_fact` creates or updates host variables during play execution. The most important use in network automation is transforming raw show command output into structured variables that subsequent tasks can use cleanly.

### Extracting Data from Show Command Output

```yaml
- name: "Gather BGP summary"
  cisco.ios.ios_command:
    commands: [show ip bgp summary]
  register: bgp_raw
  changed_when: false

# Without set_fact — every task that needs BGP data does its own parsing:
- name: "Check neighbor count (ugly)"
  ansible.builtin.assert:
    that:
      - bgp_raw.stdout[0] | regex_findall('Established') | length >= 2

# With set_fact — parse once, use cleanly everywhere:
- name: "Parse BGP summary into structured variables"
  ansible.builtin.set_fact:
    bgp_neighbor_count: "{{ bgp_raw.stdout[0] | regex_findall('Established') | length }}"
    bgp_router_id: >-
      {{ bgp_raw.stdout[0]
         | regex_search('BGP router identifier (\S+)', '\\1')
         | first | default('unknown') }}
    bgp_local_as: >-
      {{ bgp_raw.stdout[0]
         | regex_search('local AS number (\d+)', '\\1')
         | first | default('0') | int }}

# Now every downstream task uses clean variable names:
- name: "Assert BGP health"
  ansible.builtin.assert:
    that:
      - bgp_neighbor_count | int >= 2
      - bgp_router_id != 'unknown'
    fail_msg: "BGP unhealthy: {{ bgp_neighbor_count }} neighbors, router-id={{ bgp_router_id }}"

- name: "Update Netbox with BGP data"
  netbox.netbox.netbox_device:
    data:
      name: "{{ inventory_hostname }}"
      custom_fields:
        bgp_as: "{{ bgp_local_as }}"
        bgp_neighbor_count: "{{ bgp_neighbor_count }}"
  delegate_to: localhost
```

### Building Cumulative Variables with set_fact

```yaml
# Pattern: accumulate results from a loop into a list
- name: "Check each BGP neighbor"
  ansible.builtin.set_fact:
    bgp_results: >-
      {{ bgp_results | default([]) + [{
        'neighbor': item.ip,
        'status': 'up' if item.ip in bgp_raw.stdout[0] else 'down'
      }] }}
  loop: "{{ bgp.neighbors | default([]) }}"
  loop_control:
    label: "{{ item.ip }}"

# bgp_results is now a list of dicts usable in reports and assertions
- name: "Assert all neighbors up"
  ansible.builtin.assert:
    that:
      - bgp_results | selectattr('status', 'equalto', 'down') | list | length == 0
    fail_msg: >
      Down neighbors: {{ bgp_results | selectattr('status', 'equalto', 'down')
                        | map(attribute='neighbor') | list | join(', ') }}
```

### cacheable: true — Persisting Facts Across Plays

```yaml
- name: "Set fact and cache it for later plays"
  ansible.builtin.set_fact:
    ios_current_version: "{{ parsed_version }}"
    cacheable: true    # Stores in Ansible fact cache (requires fact_caching in ansible.cfg)
  # A later play can read ios_current_version without re-running the gather task
  # Useful in multi-play pipelines where each play doesn't re-gather facts
```

---

## 29.7 — pre_tasks and post_tasks

`pre_tasks` and `post_tasks` are play-level sections that execute before and after the `tasks` and `roles` sections respectively.

```
Execution order within a play:
  1. pre_tasks
  2. roles
  3. tasks
  4. post_tasks
```

### pre_tasks for Setup and Gate Checks

```yaml
pre_tasks:
  - name: "Pre | Verify maintenance window is active"
    ansible.builtin.assert:
      that:
        - lookup('pipe', 'date +%H') | int >= 2
        - lookup('pipe', 'date +%H') | int <= 4
      fail_msg: "Not in maintenance window (02:00-04:00). Aborting."
    delegate_to: localhost
    run_once: true

  - name: "Pre | Take config backup before any changes"
    cisco.ios.ios_command:
      commands: [show running-config]
    register: pre_change_config

  - name: "Pre | Save backup"
    ansible.builtin.copy:
      content: "{{ pre_change_config.stdout[0] }}"
      dest: "backups/pre_change_{{ inventory_hostname }}_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.txt"
      mode: '0644'
    delegate_to: localhost
    # Backup taken before any task or role modifies the device
```

### post_tasks with tags: always for Guaranteed Cleanup

```yaml
post_tasks:
  - name: "Post | Send completion notification"
    ansible.builtin.uri:
      url: "{{ slack_webhook_url }}"
      method: POST
      body_format: json
      body:
        text: "Deployment complete: {{ ansible_play_hosts | length }} devices updated"
    delegate_to: localhost
    run_once: true
    tags: always    # Runs even if main tasks fail — notification always fires

  - name: "Post | Remove temp files from devices"
    cisco.ios.ios_command:
      commands: [delete /force flash:temp_*]
    changed_when: false
    failed_when: false    # Don't fail post_task cleanup
    tags: always
```

The `tags: always` annotation on post_tasks is the key pattern. Without it, `ansible-playbook --tags deploy` skips post_tasks that don't have the `deploy` tag. With `tags: always`, those tasks run regardless of which tags were specified on the command line.

---

## 29.8 — The Remaining Techniques (Brief Reference)

### include_role and import_role

The same static vs dynamic distinction from include_tasks/import_tasks applies to roles:

```yaml
# Static — role name is known at parse time
- ansible.builtin.import_role:
    name: network_validate
  # Role variables and handlers are available to the whole play
  # --tags works correctly with tasks inside the role

# Dynamic — role name or decision to include depends on runtime variable
- ansible.builtin.include_role:
    name: "{{ platform }}_deploy"    # platform set by set_fact earlier
  # Role is loaded at runtime — --tags does not work reliably inside
  # Use when the role to include varies by host or runtime condition
```

### strategy: free vs strategy: linear

```yaml
# Default: linear — all hosts complete task N before any host starts task N+1
- hosts: cisco_ios
  strategy: linear    # (default, doesn't need to be written)
  tasks:
    - Task 1: [r1, r2, r3, r4] all complete, then...
    - Task 2: [r1, r2, r3, r4] all complete, then...

# free — each host proceeds through tasks independently
- hosts: cisco_ios
  strategy: free
  tasks:
    - r1 might be on Task 3 while r4 is still on Task 1
    # Use for long-running tasks where hosts have variable completion times
    # Avoid for tasks with inter-host dependencies (BGP peers need coordination)
    # Use for: fact gathering, independent show commands, parallel file downloads
```

For network automation, `strategy: linear` is almost always correct. `strategy: free` is useful for fleet-wide read-only operations (audits, fact gathering) where devices are independent and the goal is maximum throughput.

### Callbacks (Brief Reference)

Callbacks are plugins that intercept Ansible events (play start, task result, play end) and do something with them — format output, write to a file, send to a monitoring system.

```ini
# ansible.cfg — enable built-in callbacks

[defaults]
# stdout_callback changes what you see in the terminal
stdout_callback = yaml        # Cleaner output than default 'default' callback
                              # Shows task results as readable YAML
# Other useful stdout callbacks:
# stdout_callback = json      # Machine-readable JSON — good for piping to jq
# stdout_callback = dense     # Compact — shows one line per task
# stdout_callback = debug     # Verbose — shows all variables on failure

# callback_whitelist enables additional callbacks alongside stdout
callback_whitelist = timer, profile_tasks, mail

[callback_timer]
# timer: shows total play duration at the end

[callback_profile_tasks]
# profile_tasks: shows time per task — find slow tasks
sort_order = descending       # Show slowest first
task_output_limit = 20        # Show top 20 slowest tasks
```

Enable the yaml callback and profile_tasks callback for the lab — it makes output significantly more readable and immediately shows which tasks take the most time:

```bash
# Test the yaml callback on an existing playbook
ANSIBLE_STDOUT_CALLBACK=yaml \
  ansible-playbook playbooks/validate/validate_network.yml \
  --limit wan-r1

# Test profile_tasks to find slow tasks
ANSIBLE_CALLBACKS_ENABLED=profile_tasks \
  ansible-playbook playbooks/backup/backup_all.yml
```

---



Advanced playbook techniques are complete — modular task inclusion, delegation and aggregation patterns, rolling execution with blast-radius control, structured fact building from device output, and guaranteed cleanup through pre/post tasks. The guide now covers every technique needed to write production-quality network automation playbooks. The final part brings everything together in the capstone.


