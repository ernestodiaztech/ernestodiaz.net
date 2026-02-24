---
draft: true
title: '15 - Error Handling'
weight: 15
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 15: Handlers & Error Handling

> *Part 13 covered handlers in the context of roles — the "save configuration once" pattern. This part goes further: handler patterns that didn't fit in Part 13, the three keywords that control whether Ansible treats a task outcome as success or failure, structured error handling with block/rescue/always, and the automatic rollback system that connects to the backup playbook from Part 8. Error handling is what separates automation that's safe to run in production from automation that's only safe in a lab.*

---

## 15.1 — Handlers Recap and New Patterns

Part 13 established the core handler concept: `notify:` triggers a handler, handlers run once at play end, `force_handlers: true` ensures they run even on failure. Here are the handler patterns that weren't covered there.

### Listening to Multiple Notifiers

A handler's `listen:` key lets multiple tasks notify it using a topic name rather than the handler's own name. This decouples task notifications from handler names — if I rename the handler, I don't need to update every `notify:` in every task:

```yaml
tasks:
  - name: "Config | Set hostname"
    cisco.ios.ios_hostname:
      config:
        hostname: "{{ device_hostname }}"
      state: merged
    notify: ios config changed    # ← Notifies the topic, not the handler name

  - name: "Config | Set NTP"
    cisco.ios.ios_ntp_global:
      config:
        servers:
          - server: "{{ ntp_servers[0] }}"
      state: merged
    notify: ios config changed    # ← Same topic

handlers:
  - name: "Save IOS running config to NVRAM"    # ← Handler name (internal)
    cisco.ios.ios_command:
      commands: [write memory]
    listen: ios config changed    # ← Topic name (what tasks notify)
```

The distinction between `name:` and `listen:` means I can have handler names that describe what the handler does (`Save IOS running config to NVRAM`) while tasks notify using a shorter topic (`ios config changed`). Multiple handlers can listen to the same topic:

```yaml
handlers:
  - name: "Save IOS running config to NVRAM"
    cisco.ios.ios_command:
      commands: [write memory]
    listen: ios config changed

  - name: "Log config change to syslog"
    cisco.ios.ios_command:
      commands:
        - "send log 6 Configuration change applied by Ansible"
    listen: ios config changed    # ← Both handlers trigger on the same notify
```

### Handler Ordering

Handlers run in the order they're **defined** in `handlers/main.yml`, not the order they're notified. This matters when one handler depends on another:

```yaml
handlers:
  # Order matters here — save must run before verify
  - name: "Save IOS configuration"
    cisco.ios.ios_command:
      commands: [write memory]
    listen: ios config changed

  - name: "Verify startup config matches running"
    cisco.ios.ios_command:
      commands: [show startup-config | include hostname]
    register: startup_verify
    listen: ios config changed    # ← Runs AFTER Save because it's defined after
```

### Triggering Handlers Mid-Play with `flush_handlers`

Covered briefly in Part 13. The practical network automation use case is a deploy-then-verify workflow where the verify step must run against the saved (not just running) configuration:

```yaml
tasks:
  - name: "Deploy | Push BGP configuration"
    cisco.ios.ios_bgp_global:
      config:
        as_number: "{{ bgp.as_number }}"
      state: merged
    notify: Save IOS configuration

  - name: "Flush | Ensure config is saved before verification"
    ansible.builtin.meta: flush_handlers
    # The "Save IOS configuration" handler runs NOW
    # not at the end of the play

  - name: "Verify | Confirm BGP config exists in startup-config"
    cisco.ios.ios_command:
      commands:
        - show startup-config | section router bgp
    register: bgp_in_startup
    # This only makes sense after write memory — flush_handlers ensures that
```

---

## 15.2 — Controlling Task Outcomes: `ignore_errors`, `failed_when`, `changed_when`

These three keywords give me precise control over how Ansible interprets a task's result. Without them, Ansible uses the module's own judgment about success and failure. With them, I override that judgment based on what the output actually means in my context.

### `ignore_errors` — Continue the Play Despite Failure

When a task fails, Ansible stops processing that host by default. `ignore_errors: true` lets the play continue even when the task fails:

```yaml
- name: "Check | Test if OSPF is running (may not be on all devices)"
  cisco.ios.ios_command:
    commands:
      - show ip ospf neighbor
  register: ospf_check
  ignore_errors: true    # ← Don't fail the play if OSPF isn't running
                         #   Some IOS devices in the group may not have OSPF configured

- name: "Report | Show OSPF status"
  ansible.builtin.debug:
    msg: "OSPF is {{ 'running' if ospf_check.failed == false else 'NOT running' }}"
```

`ignore_errors` is the right tool when a task might legitimately fail and that's acceptable — probing for features that may or may not be configured, running show commands that error on some platforms, or attempting optional configuration that's allowed to not apply.

### ### ⚠️ Warning

> `ignore_errors: true` is frequently overused as a way to silence noisy failures. If I find myself putting `ignore_errors: true` on a configuration-changing task because it "sometimes fails," that's a signal something is wrong with the task logic — not that I should ignore the failure. Reserve `ignore_errors` for genuinely optional operations where failure is an expected, handled outcome.

### `failed_when` — Define What Counts as Failure

Some modules always return `ok` even when the underlying operation failed. Some return failures for conditions that are actually fine. `failed_when` lets me define the failure condition explicitly based on the task's output:

```yaml
- name: "Check | Verify BGP session is established"
  cisco.ios.ios_command:
    commands:
      - show ip bgp neighbors {{ bgp.neighbors[0].ip }} | include BGP state
  register: bgp_state
  failed_when:
    - bgp_state.stdout[0] | length > 0           # Command returned output
    - "'Established' not in bgp_state.stdout[0]"  # But not the word Established
  # Fails if: output exists AND doesn't contain "Established"
  # Passes if: output contains "Established"
```

Multiple conditions in `failed_when` are ANDed — all must be true for the task to fail. For OR logic I use Jinja2:

```yaml
failed_when: >
  'Error' in bgp_state.stdout[0] or
  'Down' in bgp_state.stdout[0] or
  bgp_state.stdout[0] | length == 0
```

A practical NX-OS example — `nxos_command` returns rc=0 even when the command output contains error text:

```yaml
- name: "Config | Configure VPC peer-link"
  cisco.nxos.nxos_command:
    commands:
      - vpc peer-link
  register: vpc_result
  failed_when: "'ERROR' in vpc_result.stdout[0] or 'Invalid' in vpc_result.stdout[0]"
  # nxos_command returns ok even for invalid commands — failed_when catches them
```

### `changed_when` — Define What Counts as a Change

By default, `ios_command` and `nxos_command` (show commands) always report `ok` — they never report `changed` because they're read-only. But some modules that push config report `changed` even when the config was already correct. `changed_when` lets me override this:

```yaml
# Force a show command to never report changed (it's read-only — this is explicit)
- name: "Info | Get interface counters"
  cisco.ios.ios_command:
    commands:
      - show interfaces counters
  register: interface_counters
  changed_when: false    # ← This task can never cause a change — be explicit

# Force a config task to report changed only when the output contains specific text
- name: "Config | Apply ACL to interface"
  cisco.ios.ios_config:
    lines:
      - ip access-group MGMT_ACCESS in
    parents: interface GigabitEthernet1
  register: acl_result
  changed_when: acl_result.updates | length > 0
  # ios_config sets .updates to the list of lines it actually pushed
  # If .updates is empty, the config was already correct — no real change
```

The `changed_when: false` pattern is particularly useful for tasks that generate reports or collect facts — it prevents them from showing up as changes in the play recap and keeps `changed: N` counts meaningful.

### The Three Together — A Validation Task Pattern

```yaml
- name: "Validate | Check NTP sync status"
  cisco.ios.ios_command:
    commands:
      - show ntp status
  register: ntp_status
  changed_when: false                          # Read-only — never a change
  failed_when:
    - "'synchronized' not in ntp_status.stdout[0].lower()"
  # Fails if clock is not synchronized
  # This turns a show command into an assertion
```

This pattern — `changed_when: false` + `failed_when:` based on output content — is how I write validation tasks that behave like assertions. The task passes only if the device is in the expected state.

---

## 15.3 — Structured Error Handling: `block`, `rescue`, `always`

`block/rescue/always` is Ansible's structured exception handling — equivalent to `try/except/finally` in Python. It's the most powerful error handling tool in Ansible and the right approach for any operation where failure requires a specific response.

### The Structure

```yaml
tasks:
  - block:
      # ── Try ─────────────────────────────────────────────────
      # Tasks that might fail go here
      # If any task in the block fails, execution jumps to rescue:

    rescue:
      # ── Except ──────────────────────────────────────────────
      # Tasks that run ONLY when the block fails
      # Used for: cleanup, rollback, alerting, logging the failure

    always:
      # ── Finally ─────────────────────────────────────────────
      # Tasks that ALWAYS run, whether the block succeeded or failed
      # Used for: guaranteed cleanup, status reporting
```

### A Realistic Network Automation Failure Scenario

Here's the scenario: I'm deploying a new BGP configuration to `wan-r1`. The deploy task pushes the config, but BGP fails to establish — maybe a neighbor IP is wrong, maybe the remote AS is incorrect. Without structured error handling, the play fails, the broken config stays on the device, and the engineer has to manually SSH in to fix it. With `block/rescue/always`, the failure is caught, the original config is automatically restored, and the engineer gets a clear failure message.

```bash
nano ~/projects/ansible-network/playbooks/deploy/deploy_bgp_safe.yml
```

```yaml
---
# =============================================================
# deploy_bgp_safe.yml
# BGP deployment with automatic rollback on failure.
# Connects to the backup system from Part 8.
#
# Usage:
#   ansible-playbook playbooks/deploy/deploy_bgp_safe.yml
#   ansible-playbook playbooks/deploy/deploy_bgp_safe.yml -l wan-r1
# =============================================================

- name: "Deploy | BGP configuration with automatic rollback"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
    backup_path: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_pre_bgp_{{ timestamp }}.cfg"
    bgp_establish_timeout: 60    # Seconds to wait for BGP to come up

  tasks:

    # ── Step 1: Always gather facts first ──────────────────────────
    - name: "Pre-flight | Gather IOS facts"
      cisco.ios.ios_facts:
        gather_subset: default
      tags: always

    # ── Step 2: The protected block ────────────────────────────────
    - name: "BGP Deploy | Protected deployment with rollback"
      block:

        # ── Block: Pre-change backup ────────────────────────────
        - name: "BGP Deploy | Create backup directory"
          ansible.builtin.file:
            path: "backups/cisco_ios/{{ inventory_hostname }}"
            state: directory
            mode: '0755'
          delegate_to: localhost

        - name: "BGP Deploy | Capture pre-change running config"
          cisco.ios.ios_command:
            commands: [show running-config]
          register: pre_change_config

        - name: "BGP Deploy | Save pre-change backup"
          ansible.builtin.copy:
            content: "{{ pre_change_config.stdout[0] }}"
            dest: "{{ backup_path }}"
            mode: '0644'
          delegate_to: localhost

        - name: "BGP Deploy | Confirm backup was written"
          ansible.builtin.stat:
            path: "{{ backup_path }}"
          register: backup_stat
          delegate_to: localhost
          failed_when: not backup_stat.stat.exists
          changed_when: false
          # Abort if backup didn't write — don't proceed without a safety net

        # ── Block: Push BGP configuration ──────────────────────
        - name: "BGP Deploy | Configure BGP process"
          cisco.ios.ios_bgp_global:
            config:
              as_number: "{{ bgp.as_number }}"
              bgp:
                router_id:
                  address: "{{ bgp.router_id }}"
                log_neighbor_changes: true
            state: merged
          notify: Save IOS configuration

        - name: "BGP Deploy | Configure BGP neighbors"
          cisco.ios.ios_bgp_global:
            config:
              as_number: "{{ bgp.as_number }}"
              neighbor:
                - neighbor_address: "{{ item.ip }}"
                  remote_as: "{{ item.remote_as }}"
                  description: "{{ item.description }}"
            state: merged
          loop: "{{ bgp.neighbors }}"
          loop_control:
            label: "BGP neighbor {{ item.ip }} (AS {{ item.remote_as }})"
          notify: Save IOS configuration

        - name: "BGP Deploy | Save configuration before verification"
          ansible.builtin.meta: flush_handlers
          # Write memory NOW so BGP config persists before we test it

        # ── Block: Post-deploy verification ─────────────────────
        - name: "BGP Deploy | Wait for BGP sessions to establish"
          cisco.ios.ios_command:
            commands:
              - show ip bgp summary
            wait_for:
              - result[0] contains Established    # Wait until 'Established' appears
            retries: 12                           # Try up to 12 times
            interval: 5                           # Every 5 seconds (60 seconds total)
          register: bgp_verify
          changed_when: false

        - name: "BGP Deploy | Assert all configured neighbors are established"
          ansible.builtin.assert:
            that:
              - bgp_verify.stdout[0] | regex_findall('Established') | length >= bgp.neighbors | length
            fail_msg: >
              BGP deployment FAILED on {{ inventory_hostname }}.
              Expected {{ bgp.neighbors | length }} established session(s).
              Current BGP summary:
              {{ bgp_verify.stdout_lines[0] | join('\n') }}
            success_msg: >
              BGP deployment SUCCESS on {{ inventory_hostname }}.
              All {{ bgp.neighbors | length }} BGP session(s) established.

      # ── Rescue: Automatic rollback on any block failure ─────────
      rescue:
        - name: "ROLLBACK | BGP deploy failed — beginning automatic rollback"
          ansible.builtin.debug:
            msg:
              - "======================================================"
              - "ROLLBACK TRIGGERED on {{ inventory_hostname }}"
              - "Failure: {{ ansible_failed_task.name }}"
              - "Error:   {{ ansible_failed_result.msg | default('See above') }}"
              - "Backup:  {{ backup_path }}"
              - "======================================================"

        - name: "ROLLBACK | Verify backup file exists before restoring"
          ansible.builtin.stat:
            path: "{{ backup_path }}"
          register: rollback_stat
          delegate_to: localhost

        - name: "ROLLBACK | Abort — backup file not found"
          ansible.builtin.fail:
            msg: >
              CRITICAL: Cannot rollback {{ inventory_hostname }} —
              backup file not found at {{ backup_path }}.
              Manual intervention required.
          when: not rollback_stat.stat.exists

        - name: "ROLLBACK | Push pre-change configuration"
          cisco.ios.ios_config:
            src: "{{ backup_path }}"
            replace: config      # Replace entire config (not merge)
          register: rollback_result

        - name: "ROLLBACK | Save restored configuration"
          cisco.ios.ios_command:
            commands: [write memory]

        - name: "ROLLBACK | Verify device responds after rollback"
          cisco.ios.ios_command:
            commands: [show version]
          register: post_rollback_check
          changed_when: false

        - name: "ROLLBACK | Report rollback outcome"
          ansible.builtin.debug:
            msg:
              - "======================================================"
              - "ROLLBACK COMPLETE on {{ inventory_hostname }}"
              - "Device is responsive: {{ post_rollback_check is succeeded }}"
              - "Config restored from: {{ backup_path }}"
              - "MANUAL ACTION REQUIRED: Investigate the root cause."
              - "======================================================"

        - name: "ROLLBACK | Re-raise failure after rollback"
          ansible.builtin.fail:
            msg: >
              BGP deployment failed on {{ inventory_hostname }} and was rolled back.
              Original error: {{ ansible_failed_task.name }}
          # Re-raising the failure ensures the play recap shows FAILED
          # Without this, rescue tasks completing successfully would show the play as ok

      # ── Always: Guaranteed post-run actions ─────────────────────
      always:
        - name: "Cleanup | Report final device state"
          cisco.ios.ios_command:
            commands:
              - show ip bgp summary
          register: final_bgp_state
          ignore_errors: true    # Device may be mid-rollback — don't fail here
          changed_when: false

        - name: "Cleanup | Display final BGP state"
          ansible.builtin.debug:
            msg: "{{ final_bgp_state.stdout_lines[0] | default(['BGP output unavailable']) }}"

        - name: "Cleanup | Record playbook run in local log"
          ansible.builtin.lineinfile:
            path: "backups/cisco_ios/deploy_log.txt"
            line: >
              {{ timestamp }} | {{ inventory_hostname }} | BGP deploy |
              {{ 'SUCCESS' if ansible_failed_task is not defined else 'FAILED+ROLLED_BACK' }}
            create: true
            mode: '0644'
          delegate_to: localhost
          ignore_errors: true    # Log failure is not critical

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration
```

### Walking Through the Execution Flow

**Happy path (BGP establishes correctly):**
```
block tasks run → BGP deploys → sessions establish → assert passes
                                                              ↓
                                                    always tasks run
                                                    (final state report + log)
```

**Failure path (BGP doesn't establish):**
```
block tasks start → backup created → BGP deployed → assert FAILS
                                                              ↓
                                                    rescue tasks run
                                                    (rollback report + config restore)
                                                              ↓
                                                    always tasks run
                                                    (final state report + log)
                                                              ↓
                                                    play recap shows FAILED
```

### The Special Variables Available in `rescue:`

When execution jumps to `rescue:`, two special variables are available that describe what failed:

```yaml
rescue:
  - name: "Log the failure details"
    ansible.builtin.debug:
      msg:
        - "Failed task:   {{ ansible_failed_task.name }}"
        - "Failed task:   {{ ansible_failed_task.action }}"
        - "Error message: {{ ansible_failed_result.msg | default('no message') }}"
        - "Return code:   {{ ansible_failed_result.rc | default('N/A') }}"
        - "stdout:        {{ ansible_failed_result.stdout | default('') }}"
```

| Variable | Contains |
|---|---|
| `ansible_failed_task.name` | The name: of the task that failed |
| `ansible_failed_task.action` | The module name that failed |
| `ansible_failed_result.msg` | The error message from the module |
| `ansible_failed_result.rc` | The return code (if applicable) |
| `ansible_failed_result.stdout` | The stdout output at failure |

These variables only exist within `rescue:` — they're not available in `always:` or in subsequent tasks.

### ### ℹ️ Info

> The `always:` section runs whether the block succeeded OR whether the rescue tasks succeeded. If both the block AND the rescue fail (for example, the backup file doesn't exist and the rollback itself fails), `always:` still runs. This makes `always:` the right place for logging and cleanup that must happen regardless of any outcome — it's the unconditional guarantee.

---

## 15.4 — Connecting to the Part 8 Backup System

The rollback in section 15.3 references a backup file at `backup_path`. That backup is created in the same play, immediately before the change. But in real operations, I want to use the most recent scheduled backup — not just a backup from the same play run.

### The Complete Backup-to-Rollback System

The backup playbook from Part 8 creates timestamped files:
```
backups/cisco_ios/wan-r1/wan-r1_20240315_140000.cfg
backups/cisco_ios/wan-r1/wan-r1_20240315_020000.cfg  ← Previous scheduled backup
```

When I need to rollback to the last known-good state (not just pre-change), I find the most recent backup file and restore from it:

```bash
nano ~/projects/ansible-network/playbooks/rollback/rollback_to_last_backup.yml
```

```yaml
---
# =============================================================
# rollback_to_last_backup.yml
# Restore device configuration from the most recent backup.
# Use when: a change window went wrong and pre-change backup
# was not captured in the deployment playbook.
#
# Usage:
#   Single device:  ansible-playbook rollback_to_last_backup.yml -l wan-r1
#   Confirm only:   ansible-playbook rollback_to_last_backup.yml -l wan-r1 --check
# =============================================================

- name: "Rollback | Restore from most recent scheduled backup"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable

  tasks:

    - name: "Rollback | Find most recent backup file for {{ inventory_hostname }}"
      ansible.builtin.find:
        paths: "backups/cisco_ios/{{ inventory_hostname }}"
        patterns: "*.cfg"
        age: "-7d"          # Files modified within the last 7 days
        age_stamp: mtime
      register: backup_files
      delegate_to: localhost

    - name: "Rollback | Abort — no backup files found"
      ansible.builtin.fail:
        msg: >
          No backup files found for {{ inventory_hostname }} in
          backups/cisco_ios/{{ inventory_hostname }}/.
          Run the backup playbook first: ansible-playbook playbooks/backup/backup_all.yml
      when: backup_files.files | length == 0

    - name: "Rollback | Identify most recent backup"
      ansible.builtin.set_fact:
        latest_backup: >-
          {{ backup_files.files | sort(attribute='mtime') | last }}

    - name: "Rollback | Display backup that will be restored"
      ansible.builtin.debug:
        msg:
          - "Device:       {{ inventory_hostname }}"
          - "Backup file:  {{ latest_backup.path }}"
          - "Backup date:  {{ '%Y-%m-%d %H:%M:%S' | strftime(latest_backup.mtime) }}"
          - "File size:    {{ latest_backup.size }} bytes"

    - name: "Rollback | Pause for confirmation before restoring"
      ansible.builtin.pause:
        prompt: >
          About to restore {{ inventory_hostname }} from
          {{ latest_backup.path }} ({{ '%Y-%m-%d %H:%M:%S' | strftime(latest_backup.mtime) }}).
          Press ENTER to continue or Ctrl+C then A to abort.
      when: not ansible_check_mode

    - name: "Rollback | Capture current config before restore"
      cisco.ios.ios_command:
        commands: [show running-config]
      register: current_config

    - name: "Rollback | Save current config as safety snapshot"
      ansible.builtin.copy:
        content: "{{ current_config.stdout[0] }}"
        dest: >-
          backups/cisco_ios/{{ inventory_hostname }}/
          {{ inventory_hostname }}_pre_rollback_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.cfg
        mode: '0644'
      delegate_to: localhost

    - name: "Rollback | Push backup configuration to device"
      cisco.ios.ios_config:
        src: "{{ latest_backup.path }}"
        replace: config
      register: rollback_push

    - name: "Rollback | Save restored configuration"
      cisco.ios.ios_command:
        commands: [write memory]
      when: rollback_push.changed

    - name: "Rollback | Verify device is responsive after restore"
      cisco.ios.ios_command:
        commands: [show version]
      register: post_rollback_verify
      changed_when: false

    - name: "Rollback | Report outcome"
      ansible.builtin.debug:
        msg:
          - "Rollback complete for {{ inventory_hostname }}"
          - "Restored from: {{ latest_backup.path }}"
          - "Device responsive: {{ post_rollback_verify is succeeded }}"
```

### The Backup Discipline This Requires

For this system to work reliably, backups must be current. The recommended schedule:

```bash
# Cron job on the Ubuntu VM — backup all devices every night at 2am
# Edit with: crontab -e
0 2 * * * cd /home/ansible/projects/ansible-network && \
    source ~/venvs/ansible-network/bin/activate && \
    ansible-playbook playbooks/backup/backup_all.yml \
    >> /var/log/ansible-backup.log 2>&1
```

And before any planned change window, always run the backup manually:

```bash
# Manual pre-change backup
ansible-playbook playbooks/backup/backup_all.yml --limit cisco_ios
```

This ensures the rollback system always has a recent known-good configuration to restore from.

---

## 15.5 — `any_errors_fatal` — Controlling Blast Radius

By default, when a task fails on one host, Ansible removes that host from the play and continues with the others. In a 20-device play, one device's failure doesn't abort the other 19.

### The Default Behavior and When It's Wrong

```
Default (any_errors_fatal: false):
wan-r1  → Task 3 fails → removed from play → Tasks 4, 5, 6 skip for wan-r1
wan-r2  → Task 3 ok   → continues → Tasks 4, 5, 6 run for wan-r2
spine-01 → Task 3 ok  → continues → Tasks 4, 5, 6 run for spine-01
...
```

This is usually the right behavior — one broken device shouldn't prevent all the others from getting their configuration. But there are scenarios where continuing after a failure creates a worse situation than stopping entirely.

### The Scenario Where `any_errors_fatal` Matters

Imagine a playbook that deploys a new routing policy across all WAN routers. The sequence is:

1. Deploy new BGP policy to all routers
2. Verify BGP sessions are still established
3. Remove the old static routes that the new BGP policy replaces

If step 2 fails on `wan-r1` (BGP didn't come up with the new policy) but succeeds on `wan-r2`, and the play continues:
- `wan-r2` proceeds to step 3 and removes the old static routes
- `wan-r1` has a broken BGP policy but still has the static routes (didn't reach step 3)
- Network is now in an asymmetric state — traffic takes different paths depending on direction
- This is often worse than if neither router had been changed

With `any_errors_fatal: true`:

```yaml
- name: "Deploy | Coordinated routing policy change"
  hosts: wan
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  any_errors_fatal: true    # ← If ANY host fails, stop the ENTIRE play immediately
  force_handlers: true

  tasks:
    - name: "Deploy | Push new BGP policy"
      cisco.ios.ios_bgp_global:
        ...

    - name: "Verify | BGP sessions established on all routers"
      cisco.ios.ios_command:
        commands: [show ip bgp summary]
        wait_for:
          - result[0] contains Established
        retries: 12
        interval: 5
      changed_when: false
      # If ANY router fails this check, the entire play stops here.
      # Step 3 (removing static routes) never runs for any router.
      # The network stays in its previous state everywhere.

    - name: "Deploy | Remove superseded static routes"
      cisco.ios.ios_static_routes:
        ...
        state: deleted
      # This only runs if every router passed the verification step
```

### The Blast Radius Decision

```
any_errors_fatal: false (default)
  Use when: Tasks are independent per device.
  Effect: One failure affects only that device.
  Risk: Other devices may reach states that assume all devices succeeded.
  Example: Pushing base config (hostname, NTP) — each device is independent

any_errors_fatal: true
  Use when: Tasks across devices are interdependent.
  Effect: One failure stops the entire play.
  Risk: All devices get the same partial change if the play stops mid-way.
  Example: Coordinated routing changes, VPC pair configurations,
           anything where asymmetric state is worse than no change
```

### `max_fail_percentage` — A Middle Ground

Between "stop if anyone fails" and "continue regardless", `max_fail_percentage` stops the play only if more than a threshold percentage of hosts fail:

```yaml
- name: "Deploy | Rolling NTP update"
  hosts: cisco_ios
  gather_facts: false
  max_fail_percentage: 20    # Stop play if more than 20% of hosts fail
                              # With 10 IOS devices: stop if 3+ fail
  tasks:
    - name: "Config | Update NTP configuration"
      cisco.ios.ios_ntp_global:
        ...
```

This allows for the reality that some devices may be temporarily unreachable (rebooting, in a maintenance state) without failing the entire play, while still stopping if a large fraction of devices are failing — which suggests a systematic problem rather than a one-off issue.

---

## 15.6 — Putting It Together: A Production-Ready Error-Handling Wrapper

This pattern wraps any risky task sequence in the full error handling stack. I use this as a template for any playbook that makes irreversible or high-impact changes:

```yaml
# The production error-handling wrapper template
# Replace <CHANGE_DESCRIPTION> and the tasks inside block: with actual content

- name: "Deploy | <CHANGE_DESCRIPTION>"
  hosts: <target_group>
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  force_handlers: true          # Save config even if play fails
  any_errors_fatal: false       # Evaluate per-change — set true for coordinated changes

  vars:
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"
    backup_path: "backups/cisco_ios/{{ inventory_hostname }}/{{ inventory_hostname }}_pre_change_{{ timestamp }}.cfg"

  tasks:

    - name: "Pre-flight | Gather facts"
      cisco.ios.ios_facts:
        gather_subset: default
      tags: always

    - block:

        # ── 1. Backup ──────────────────────────────────────────────
        - name: "Backup | Capture pre-change configuration"
          cisco.ios.ios_command:
            commands: [show running-config]
          register: pre_config

        - name: "Backup | Save to control node"
          ansible.builtin.copy:
            content: "{{ pre_config.stdout[0] }}"
            dest: "{{ backup_path }}"
            mode: '0644'
          delegate_to: localhost

        # ── 2. Change ──────────────────────────────────────────────
        # ... change tasks go here ...
        # ... each task uses notify: Save IOS configuration ...

        # ── 3. Verify ──────────────────────────────────────────────
        # ... verification tasks with failed_when and changed_when: false ...

      rescue:
        - name: "RESCUE | Log failure details"
          ansible.builtin.debug:
            msg:
              - "FAILED: {{ ansible_failed_task.name }}"
              - "Error:  {{ ansible_failed_result.msg | default('unknown') }}"

        - name: "RESCUE | Restore pre-change configuration"
          cisco.ios.ios_config:
            src: "{{ backup_path }}"
            replace: config
          when: backup_path is defined

        - name: "RESCUE | Save restored configuration"
          cisco.ios.ios_command:
            commands: [write memory]

        - name: "RESCUE | Re-raise to mark play as failed"
          ansible.builtin.fail:
            msg: "Change failed on {{ inventory_hostname }} — rollback complete. See above."

      always:
        - name: "Always | Report final state"
          cisco.ios.ios_command:
            commands: [show version | include Version]
          register: final_state
          ignore_errors: true
          changed_when: false

        - name: "Always | Display final state"
          ansible.builtin.debug:
            msg: "{{ inventory_hostname }}: {{ final_state.stdout[0] | default('device not responding') }}"

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration
```

---

## 15.7 — Common Gotchas

### ### 🪲 Gotcha — `rescue:` masks the original failure unless re-raised

If `rescue:` tasks all succeed, the play recap shows the play as **OK** — not as FAILED. This means a rollback that worked correctly shows as a success, which is misleading: the change didn't apply, the rollback happened, but the summary says everything is fine.

Always end `rescue:` with `ansible.builtin.fail` to re-raise the failure:

```yaml
rescue:
  - name: "Rollback | Restore config"
    cisco.ios.ios_config:
      src: "{{ backup_path }}"
      replace: config

  - name: "Rollback | Re-raise failure"
    ansible.builtin.fail:
      msg: "Deployment failed — rollback complete. Manual review required."
# ↑ Without this, the play recap shows OK even though the change was rolled back
```

### ### 🪲 Gotcha — `changed_when: false` on a task that uses `notify:` suppresses the handler

If I set `changed_when: false` on a task that also has `notify:`, the handler is never triggered — because `changed_when: false` means the task always reports `ok`, and handlers only trigger on `changed`:

```yaml
# ❌ Handler never fires — task always reports ok due to changed_when: false
- name: "Config | Push BGP config"
  cisco.ios.ios_config:
    lines: [...]
  notify: Save IOS configuration
  changed_when: false    # ← This prevents the handler from ever triggering

# ✅ Use changed_when only on read-only or idempotent-by-definition tasks
- name: "Check | Show BGP summary"
  cisco.ios.ios_command:
    commands: [show ip bgp summary]
  register: bgp_check
  changed_when: false    # ← Correct — this IS a read-only task
```

### ### 🪲 Gotcha — `any_errors_fatal` applies to the play, not the block

`any_errors_fatal: true` at the play level stops the entire play across all hosts if any host fails any task. But a `block/rescue` within that play catches the failure at the block level — the rescue runs and the failure is handled, so the play may not see it as a fatal error.

If I have `any_errors_fatal: true` AND a `block/rescue`, I need to re-raise the failure in `rescue:` (with `ansible.builtin.fail`) for `any_errors_fatal` to stop the other hosts. Without the re-raise, the rescue completes successfully and the other hosts continue.

### ### 🪲 Gotcha — `ignore_errors: true` affects the play recap but not `rescue:`

`ignore_errors: true` makes a failed task appear as `...ignoring` in output and lets the play continue, but it does NOT trigger `rescue:`. Only unhandled failures (tasks without `ignore_errors`) jump to `rescue:`.

```yaml
block:
  - name: "This failure DOES trigger rescue:"
    cisco.ios.ios_command:
      commands: [invalid_command]
    # No ignore_errors — failure jumps to rescue:

  - name: "This failure does NOT trigger rescue:"
    cisco.ios.ios_command:
      commands: [another_invalid_command]
    ignore_errors: true    # ← Error is swallowed, rescue: never sees it
```

---


Error handling is now a complete system: backup before changes, block/rescue/always for structured failure response, automatic rollback connected to the backup files, and blast-radius control with any_errors_fatal. Part 16 moves into network-specific automation — using resource modules for VLAN, interface, and routing configuration at depth.


