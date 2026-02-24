---
draft: true
title: '30 Backup & Restore'
weight: 30
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 30: Configuration Backup, Restore & Change Management

> *Configuration backup is the most basic form of network automation — and the most commonly neglected. Engineers who wouldn't run a production change without a plan for rollback still operate fleets where the only backup is whatever's in startup-config and hope that nothing has drifted. This part builds a backup system that runs on a schedule, commits every config to Git as a timestamped record, enables meaningful before/after diffs for every change, and adds a compliance layer that flags drift from required configuration standards. When something goes wrong — and it will — this system turns recovery from a search problem into a restore problem.*

---

## 30.1 — Backup Directory and Repository Structure

Before writing the playbook, establish the directory structure that backups land in and the Git repository that tracks them.

```bash
# Create the backup repository — separate from the playbook project
# Configs contain sensitive data (passwords, community strings in older configs)
# Keep them in a separate private repository
mkdir -p ~/network-backups/{ios,nxos,junos,panos}
cd ~/network-backups

# Initialize as a Git repository
git init
git branch -M main

# Create .gitignore — exclude nothing (we want every config committed)
cat > .gitignore << 'EOF'
# Temporary files only
*.tmp
*.swp
.DS_Store
EOF

# Create README
cat > README.md << 'EOF'
# Network Configuration Backups

Automated backups committed by Ansible on schedule and before/after changes.

## Directory Structure
```
ios/
  wan-r1/
    wan-r1_running_20250316_020001.cfg
    wan-r1_running_20250317_020001.cfg
  wan-r2/
    ...
nxos/
  spine-01/
    ...
junos/
  vjunos-01/
    ...
panos/
  panos-fw01/
    ...
```

## Commit Message Format
`backup: <hostname> <platform> <YYYY-MM-DD HH:MM:SS>`
`pre-change: <hostname> <change-ticket>`
`post-change: <hostname> <change-ticket>`

## Restoring from Backup
See: https://github.com/your-org/ansible-network/blob/main/playbooks/backup/restore.yml
EOF

git add .
git commit -m "init: backup repository structure"

# Add remote (replace with your actual remote URL)
git remote add origin git@github.com:your-org/network-backups.git
```

```bash
# Add backup repo path to the main project's ansible.cfg
cat >> ~/projects/ansible-network/ansible.cfg << 'EOF'

[backup]
backup_dir = /home/netops/network-backups
EOF
```

---

## 30.2 — The Universal Backup Playbook

Four separate plays in a single file — one per platform. Each play runs only against its own host group, uses the correct connection type, and saves the config in the right format. A final play handles the Git commit that captures all four platforms' backups in one atomic commit.

```bash
mkdir -p ~/projects/ansible-network/playbooks/backup

cat > ~/projects/ansible-network/playbooks/backup/backup_all.yml << 'EOF'
---
# =============================================================
# backup_all.yml — Universal configuration backup
# Supports: Cisco IOS/IOS-XE, Cisco NX-OS, Juniper Junos, Palo Alto PAN-OS
#
# Usage:
#   Full backup all platforms:
#     ansible-playbook playbooks/backup/backup_all.yml
#
#   Single platform:
#     ansible-playbook playbooks/backup/backup_all.yml --tags ios
#     ansible-playbook playbooks/backup/backup_all.yml --tags nxos
#     ansible-playbook playbooks/backup/backup_all.yml --tags junos
#     ansible-playbook playbooks/backup/backup_all.yml --tags panos
#
#   Single device:
#     ansible-playbook playbooks/backup/backup_all.yml --limit wan-r1
#
#   Pre-change backup (adds change ticket to commit message):
#     ansible-playbook playbooks/backup/backup_all.yml \
#       -e "backup_type=pre-change change_ticket=CHG0001234"
#
#   Post-change backup:
#     ansible-playbook playbooks/backup/backup_all.yml \
#       -e "backup_type=post-change change_ticket=CHG0001234"
# =============================================================

# ════════════════════════════════════════════════════════════
# PLAY 1: Cisco IOS / IOS-XE
# ════════════════════════════════════════════════════════════
- name: "Backup | Cisco IOS/IOS-XE"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  tags: ios

  vars:
    backup_dir: "/home/{{ lookup('env', 'USER') }}/network-backups/ios"
    backup_type: "scheduled"          # Override: pre-change, post-change
    change_ticket: ""                 # Override: CHG0001234
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "IOS | Create device backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}/{{ inventory_hostname }}"
        state: directory
        mode: '0750'
      delegate_to: localhost

    - name: "IOS | Gather running configuration"
      cisco.ios.ios_command:
        commands:
          - show running-config
          - show version
          - show ip interface brief
          - show ip route summary
      register: ios_backup_data
      changed_when: false

    - name: "IOS | Write running config backup"
      ansible.builtin.copy:
        content: |
          ! ============================================================
          ! Backup metadata
          ! Host:      {{ inventory_hostname }}
          ! Platform:  Cisco IOS/IOS-XE
          ! Timestamp: {{ lookup('pipe', 'date') }}
          ! Type:      {{ backup_type }}
          ! Ticket:    {{ change_ticket | default('N/A') }}
          ! Ansible:   {{ lookup('pipe', ansible_playbook_python + ' -c "import ansible; print(ansible.__version__)"') }}
          ! ============================================================
          !
          {{ ios_backup_data.stdout[0] }}
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.cfg"
        mode: '0640'
      delegate_to: localhost

    - name: "IOS | Write show version snapshot"
      ansible.builtin.copy:
        content: "{{ ios_backup_data.stdout[1] }}"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_version_{{ timestamp }}.txt"
        mode: '0640'
      delegate_to: localhost

    - name: "IOS | Maintain latest symlink"
      ansible.builtin.file:
        src: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.cfg"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_LATEST.cfg"
        state: link
        force: true
      delegate_to: localhost
      # Symlink always points to the most recent backup
      # Restore playbook uses _LATEST.cfg as the default source

    - name: "IOS | Prune backups older than 90 days"
      ansible.builtin.find:
        paths: "{{ backup_dir }}/{{ inventory_hostname }}"
        patterns: "*.cfg"
        age: "90d"
        excludes: "*LATEST*"
      register: old_backups
      delegate_to: localhost

    - name: "IOS | Remove old backup files"
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
      loop_control:
        label: "{{ item.path | basename }}"
      delegate_to: localhost

    - name: "IOS | Record backup in run log"
      ansible.builtin.lineinfile:
        path: "{{ backup_dir }}/../backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | ios | {{ inventory_hostname }} | {{ backup_type }} | {{ change_ticket | default('scheduled') }} | OK"
        create: true
        mode: '0644'
      delegate_to: localhost

# ════════════════════════════════════════════════════════════
# PLAY 2: Cisco NX-OS
# ════════════════════════════════════════════════════════════
- name: "Backup | Cisco NX-OS"
  hosts: cisco_nxos
  gather_facts: false
  connection: network_cli
  tags: nxos

  vars:
    backup_dir: "/home/{{ lookup('env', 'USER') }}/network-backups/nxos"
    backup_type: "scheduled"
    change_ticket: ""
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "NX-OS | Create device backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}/{{ inventory_hostname }}"
        state: directory
        mode: '0750'
      delegate_to: localhost

    - name: "NX-OS | Gather running configuration"
      cisco.nxos.nxos_command:
        commands:
          - show running-config
          - show version
          - show interface status
          - show ip route summary
      register: nxos_backup_data
      changed_when: false

    - name: "NX-OS | Write running config backup"
      ansible.builtin.copy:
        content: |
          ! ============================================================
          ! Backup metadata
          ! Host:      {{ inventory_hostname }}
          ! Platform:  Cisco NX-OS
          ! Timestamp: {{ lookup('pipe', 'date') }}
          ! Type:      {{ backup_type }}
          ! Ticket:    {{ change_ticket | default('N/A') }}
          ! ============================================================
          !
          {{ nxos_backup_data.stdout[0] }}
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.cfg"
        mode: '0640'
      delegate_to: localhost

    - name: "NX-OS | Maintain latest symlink"
      ansible.builtin.file:
        src: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.cfg"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_LATEST.cfg"
        state: link
        force: true
      delegate_to: localhost

    - name: "NX-OS | Prune backups older than 90 days"
      ansible.builtin.find:
        paths: "{{ backup_dir }}/{{ inventory_hostname }}"
        patterns: "*.cfg"
        age: "90d"
        excludes: "*LATEST*"
      register: old_backups
      delegate_to: localhost

    - name: "NX-OS | Remove old backup files"
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
      loop_control:
        label: "{{ item.path | basename }}"
      delegate_to: localhost

    - name: "NX-OS | Record backup in run log"
      ansible.builtin.lineinfile:
        path: "{{ backup_dir }}/../backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | nxos | {{ inventory_hostname }} | {{ backup_type }} | {{ change_ticket | default('scheduled') }} | OK"
        create: true
        mode: '0644'
      delegate_to: localhost

# ════════════════════════════════════════════════════════════
# PLAY 3: Juniper Junos
# ════════════════════════════════════════════════════════════
- name: "Backup | Juniper Junos"
  hosts: junos_devices
  gather_facts: false
  connection: ansible.netcommon.netconf
  tags: junos

  vars:
    backup_dir: "/home/{{ lookup('env', 'USER') }}/network-backups/junos"
    backup_type: "scheduled"
    change_ticket: ""
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "Junos | Create device backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}/{{ inventory_hostname }}"
        state: directory
        mode: '0750'
      delegate_to: localhost

    - name: "Junos | Gather running configuration (set format)"
      junipernetworks.junos.junos_command:
        commands:
          - show configuration | display set
          - show version
          - show interfaces terse
        display: text
      register: junos_backup_data
      changed_when: false

    - name: "Junos | Gather running configuration (hierarchical format)"
      junipernetworks.junos.junos_command:
        commands:
          - show configuration
        display: text
      register: junos_backup_hier
      changed_when: false

    - name: "Junos | Write set-format backup"
      ansible.builtin.copy:
        content: |
          ## ============================================================
          ## Backup metadata
          ## Host:      {{ inventory_hostname }}
          ## Platform:  Juniper Junos
          ## Format:    set
          ## Timestamp: {{ lookup('pipe', 'date') }}
          ## Type:      {{ backup_type }}
          ## Ticket:    {{ change_ticket | default('N/A') }}
          ## ============================================================
          ##
          {{ junos_backup_data.stdout[0] }}
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_set_{{ timestamp }}.cfg"
        mode: '0640'
      delegate_to: localhost

    - name: "Junos | Write hierarchical backup"
      ansible.builtin.copy:
        content: |
          ## ============================================================
          ## Host:      {{ inventory_hostname }}
          ## Format:    hierarchical
          ## Timestamp: {{ lookup('pipe', 'date') }}
          ## ============================================================
          ##
          {{ junos_backup_hier.stdout[0] }}
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_hier_{{ timestamp }}.cfg"
        mode: '0640'
      delegate_to: localhost
      # Junos gets two formats: set-format (clean diffs) and hierarchical (human readable)

    - name: "Junos | Maintain latest symlinks"
      ansible.builtin.file:
        src: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_set_{{ timestamp }}.cfg"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_LATEST.cfg"
        state: link
        force: true
      delegate_to: localhost

    - name: "Junos | Prune backups older than 90 days"
      ansible.builtin.find:
        paths: "{{ backup_dir }}/{{ inventory_hostname }}"
        patterns: "*.cfg"
        age: "90d"
        excludes: "*LATEST*"
      register: old_backups
      delegate_to: localhost

    - name: "Junos | Remove old backup files"
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ old_backups.files }}"
      loop_control:
        label: "{{ item.path | basename }}"
      delegate_to: localhost

    - name: "Junos | Record backup in run log"
      ansible.builtin.lineinfile:
        path: "{{ backup_dir }}/../backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | junos | {{ inventory_hostname }} | {{ backup_type }} | {{ change_ticket | default('scheduled') }} | OK"
        create: true
        mode: '0644'
      delegate_to: localhost

# ════════════════════════════════════════════════════════════
# PLAY 4: Palo Alto PAN-OS
# ════════════════════════════════════════════════════════════
- name: "Backup | Palo Alto PAN-OS"
  hosts: panos_devices
  gather_facts: false
  connection: local
  tags: panos

  vars:
    backup_dir: "/home/{{ lookup('env', 'USER') }}/network-backups/panos"
    backup_type: "scheduled"
    change_ticket: ""
    timestamp: "{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}"

  tasks:
    - name: "PAN-OS | Create device backup directory"
      ansible.builtin.file:
        path: "{{ backup_dir }}/{{ inventory_hostname }}"
        state: directory
        mode: '0750'
      delegate_to: localhost

    - name: "PAN-OS | Export running configuration"
      paloaltonetworks.panos.panos_op:
        provider:
          ip_address: "{{ ansible_host }}"
          api_key: "{{ vault_panos_api_key }}"
        cmd: "show config running"
        cmd_is_xml: false
      register: panos_backup_data

    - name: "PAN-OS | Export candidate configuration"
      paloaltonetworks.panos.panos_op:
        provider:
          ip_address: "{{ ansible_host }}"
          api_key: "{{ vault_panos_api_key }}"
        cmd: "show config candidate"
        cmd_is_xml: false
      register: panos_candidate_data

    - name: "PAN-OS | Write running config backup"
      ansible.builtin.copy:
        content: |
          <!-- ============================================================
          Backup metadata
          Host:      {{ inventory_hostname }}
          Platform:  Palo Alto PAN-OS
          Timestamp: {{ lookup('pipe', 'date') }}
          Type:      {{ backup_type }}
          Ticket:    {{ change_ticket | default('N/A') }}
          ============================================================ -->
          {{ panos_backup_data.stdout }}
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.xml"
        mode: '0640'
      delegate_to: localhost

    - name: "PAN-OS | Maintain latest symlink"
      ansible.builtin.file:
        src: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_{{ timestamp }}.xml"
        dest: "{{ backup_dir }}/{{ inventory_hostname }}/{{ inventory_hostname }}_running_LATEST.xml"
        state: link
        force: true
      delegate_to: localhost

    - name: "PAN-OS | Record backup in run log"
      ansible.builtin.lineinfile:
        path: "{{ backup_dir }}/../backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | panos | {{ inventory_hostname }} | {{ backup_type }} | {{ change_ticket | default('scheduled') }} | OK"
        create: true
        mode: '0644'
      delegate_to: localhost

# ════════════════════════════════════════════════════════════
# PLAY 5: Git commit — runs on localhost after all platform plays
# ════════════════════════════════════════════════════════════
- name: "Backup | Commit all backups to Git"
  hosts: localhost
  gather_facts: false
  tags: [ios, nxos, junos, panos, git]
  # Tagged with all platform tags so it always runs after any platform backup

  vars:
    backup_repo: "/home/{{ lookup('env', 'USER') }}/network-backups"
    backup_type: "scheduled"
    change_ticket: ""
    git_remote: "origin"
    git_branch: "main"

  tasks:
    - name: "Git | Stage all new and changed backup files"
      ansible.builtin.command:
        cmd: git add --all
        chdir: "{{ backup_repo }}"
      register: git_add
      changed_when: git_add.rc == 0

    - name: "Git | Check if there are changes to commit"
      ansible.builtin.command:
        cmd: git status --porcelain
        chdir: "{{ backup_repo }}"
      register: git_status
      changed_when: false

    - name: "Git | Build commit message"
      ansible.builtin.set_fact:
        git_commit_message: >-
          {{ backup_type }}: fleet backup {{ lookup('pipe', 'date +%Y-%m-%d %H:%M:%S') }}
          {% if change_ticket %} | ticket={{ change_ticket }}{% endif %}
          | devices={{ ansible_play_hosts | default([]) | length }}
      run_once: true

    - name: "Git | Commit backup files"
      ansible.builtin.command:
        cmd: >
          git commit
          --author="Ansible Backup <ansible@{{ lookup('pipe', 'hostname') }}>"
          -m "{{ git_commit_message }}"
        chdir: "{{ backup_repo }}"
      register: git_commit
      changed_when: git_commit.rc == 0
      when: git_status.stdout | length > 0
      # Only commit when there are actual changes — prevents empty commits
      # on runs where no configs changed since last backup

    - name: "Git | Push to remote repository"
      ansible.builtin.command:
        cmd: git push {{ git_remote }} {{ git_branch }}
        chdir: "{{ backup_repo }}"
      register: git_push
      changed_when: git_push.rc == 0
      when: git_status.stdout | length > 0
      retries: 3
      delay: 10
      # Retry 3 times with 10s delay — handles transient network issues to remote

    - name: "Git | Display commit summary"
      ansible.builtin.debug:
        msg:
          - "Backup commit: {{ git_commit.stdout_lines | default(['No changes — configs unchanged since last backup']) | last }}"
          - "Push status:   {{ 'Pushed to ' + git_remote + '/' + git_branch if git_status.stdout | length > 0 else 'Skipped (no changes)' }}"
      when: git_commit is defined

    - name: "Git | Log git result"
      ansible.builtin.lineinfile:
        path: "{{ backup_repo }}/backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | git | commit={{ git_commit.stdout_lines[-1] | default('no-changes') }} | push={{ git_push.rc | default('skipped') }}"
        create: true
        mode: '0644'
EOF
```

---

## 30.3 — Pre-Change and Post-Change Workflow

The change management pattern wraps any deployment playbook in a pre-change backup, the change itself, and a post-change backup — all tagged with the same change ticket number. The Git history then tells a complete story: what the config looked like before, what changed, and what it looked like after.

```bash
cat > ~/projects/ansible-network/playbooks/backup/change_workflow.sh << 'EOF'
#!/bin/bash
# change_workflow.sh — Pre-change backup → deploy → post-change backup
#
# Usage:
#   ./change_workflow.sh <playbook> <limit> <change-ticket> [extra-args]
#
# Example:
#   ./change_workflow.sh playbooks/deploy/deploy_ios_ccna.yml wan-r1 CHG0001234
#   ./change_workflow.sh playbooks/deploy/deploy_nxos.yml "spine-01,spine-02" CHG0001235
#
# What it does:
#   1. Pre-change backup (committed to Git with ticket number)
#   2. Show --check --diff of what would change
#   3. Pause for engineer review
#   4. Execute the deployment playbook
#   5. Post-change backup (committed with same ticket number)
#   6. Show git diff between pre and post configs

set -euo pipefail

PLAYBOOK="${1:?Usage: $0 <playbook> <limit> <ticket> [extra-args]}"
LIMIT="${2:?Limit required — e.g. wan-r1 or 'spine-01,spine-02'}"
TICKET="${3:?Change ticket required — e.g. CHG0001234}"
EXTRA="${4:-}"

PROJECT="$HOME/projects/ansible-network"
BACKUP_REPO="$HOME/network-backups"

source "$HOME/ansible-venv/bin/activate"
cd "$PROJECT"

echo "════════════════════════════════════════════════════"
echo " Change Management Workflow"
echo " Playbook: $PLAYBOOK"
echo " Limit:    $LIMIT"
echo " Ticket:   $TICKET"
echo "════════════════════════════════════════════════════"

# ── Step 1: Pre-change backup ──────────────────────────────────────
echo ""
echo "── Step 1: Pre-change backup ────────────────────────"
ansible-playbook playbooks/backup/backup_all.yml \
  --limit "$LIMIT" \
  -e "backup_type=pre-change change_ticket=$TICKET"

# Capture the pre-change config paths for diff later
PRE_CHANGE_FILES=$(find "$BACKUP_REPO" -name "*pre*" -newer "$BACKUP_REPO/backup_run.log" 2>/dev/null || true)

# ── Step 2: Show what would change ────────────────────────────────
echo ""
echo "── Step 2: Dry run (--check --diff) ─────────────────"
ansible-playbook "$PLAYBOOK" \
  --limit "$LIMIT" \
  --check \
  --diff \
  $EXTRA

# ── Step 3: Engineer review gate ──────────────────────────────────
echo ""
echo "── Step 3: Review ───────────────────────────────────"
echo "Pre-change backup committed to Git."
echo "Review the diff above."
echo ""
read -rp "Press ENTER to proceed with $TICKET, or Ctrl+C to abort: "

# ── Step 4: Execute the change ────────────────────────────────────
echo ""
echo "── Step 4: Executing deployment ─────────────────────"
ansible-playbook "$PLAYBOOK" \
  --limit "$LIMIT" \
  $EXTRA

# ── Step 5: Post-change backup ────────────────────────────────────
echo ""
echo "── Step 5: Post-change backup ───────────────────────"
ansible-playbook playbooks/backup/backup_all.yml \
  --limit "$LIMIT" \
  -e "backup_type=post-change change_ticket=$TICKET"

# ── Step 6: Show git diff between pre and post ────────────────────
echo ""
echo "── Step 6: Configuration diff (pre vs post) ─────────"
cd "$BACKUP_REPO"

# Show git log for this ticket
echo "Git commits for $TICKET:"
git log --oneline --grep="$TICKET" | head -10

echo ""
echo "Configuration changes committed:"
# Show diff between the two most recent commits touching these devices
git diff HEAD~1 HEAD -- . | grep -E "^[+-][^+-]" | head -50 || true

echo ""
echo "════════════════════════════════════════════════════"
echo " Change $TICKET complete."
echo " Pre and post configs committed to Git."
echo " Full diff: cd ~/network-backups && git log --oneline --grep=$TICKET"
echo "════════════════════════════════════════════════════"
EOF
chmod +x ~/projects/ansible-network/playbooks/backup/change_workflow.sh
```

### Using --diff Directly

For ad-hoc changes where the full workflow script is overkill:

```bash
# Show what would change without making any changes
ansible-playbook playbooks/deploy/deploy_ios_ccna.yml \
  --limit wan-r1 \
  --check \
  --diff

# Example --diff output for ios_config tasks:
# TASK [NTP | Configure NTP servers]
# --- before
# +++ after
# @@ -1,2 +1,3 @@
#  ntp server 216.239.35.0
#  ntp server 216.239.35.4
# +ntp server 216.239.35.8

# For ios_l3_interfaces resource modules, diff shows structured state:
# TASK [Interfaces | Configure IP addresses]
# --- before
# interface GigabitEthernet2
#   no ip address
# +++ after
# interface GigabitEthernet2
# +  ip address 10.20.30.1/30
```

---

## 30.4 — Restore Playbook

Restore is the highest-risk operation in this guide. Pushing the wrong config to the wrong device, or restoring a config that doesn't account for current cabling, can result in a device that cannot be reached remotely. The playbook below has three hard safety gates.

```bash
cat > ~/projects/ansible-network/playbooks/backup/restore.yml << 'EOF'
---
# =============================================================
# restore.yml — Push a saved configuration file to a device
#
# RISKS:
#   - Pushing an outdated config may remove currently-needed config
#   - Config from a different hostname will produce wrong output
#   - An interface/IP mismatch can break remote access
#   - Test in Containerlab FIRST
#
# GATES (all three must be passed):
#   1. --limit must be a single device (enforced by the playbook)
#   2. confirm_restore=true must be set via -e
#   3. backup_file must be explicitly specified via -e
#
# Usage:
#   ansible-playbook restore.yml \
#     --limit wan-r1 \
#     -e "confirm_restore=true" \
#     -e "backup_file=/home/netops/network-backups/ios/wan-r1/wan-r1_running_20250315_020001.cfg"
#
#   Restore from LATEST backup:
#   ansible-playbook restore.yml \
#     --limit wan-r1 \
#     -e "confirm_restore=true" \
#     -e "backup_file=/home/netops/network-backups/ios/wan-r1/wan-r1_running_LATEST.cfg"
# =============================================================

- name: "Restore | IOS configuration"
  hosts: cisco_ios
  gather_facts: false
  connection: network_cli
  become: true
  become_method: enable
  serial: 1                    # Never restore more than one device at a time

  vars:
    confirm_restore: false     # Gate 1: must pass -e "confirm_restore=true"
    backup_file: ""            # Gate 2: must pass -e "backup_file=/path/to/file"

  pre_tasks:
    - name: "Gate | Verify single device targeted"
      ansible.builtin.assert:
        that:
          - ansible_play_hosts | length == 1
        fail_msg: |
          RESTORE ABORTED: --limit must target exactly one device.
          Currently targeting: {{ ansible_play_hosts | join(', ') }}
          Re-run with: --limit {{ inventory_hostname }}

    - name: "Gate | Verify confirm_restore is set"
      ansible.builtin.assert:
        that:
          - confirm_restore | bool
        fail_msg: |
          RESTORE ABORTED: Explicit confirmation required.
          Re-run with: -e "confirm_restore=true"
          Read the playbook header before running.

    - name: "Gate | Verify backup file was specified and exists"
      ansible.builtin.stat:
        path: "{{ backup_file }}"
      register: backup_stat
      delegate_to: localhost
      failed_when: not backup_stat.stat.exists or backup_file == ""

    - name: "Gate | Display what will be restored and pause"
      ansible.builtin.pause:
        prompt: |
          ⚠  RESTORE CONFIRMATION

          Device:      {{ inventory_hostname }} ({{ ansible_host }})
          Backup file: {{ backup_file }}
          File size:   {{ backup_stat.stat.size | filesizeformat }}
          Modified:    {{ backup_stat.stat.mtime | int | strftime('%Y-%m-%d %H:%M:%S') }}

          This will replace the ENTIRE running configuration.
          Ensure console/OOB access is available before proceeding.

          Press ENTER to restore, or Ctrl+C then A to abort

    - name: "Pre-restore | Back up current config before overwriting"
      cisco.ios.ios_command:
        commands: [show running-config]
      register: pre_restore_backup
      changed_when: false

    - name: "Pre-restore | Save current config as emergency backup"
      ansible.builtin.copy:
        content: "{{ pre_restore_backup.stdout[0] }}"
        dest: "/home/{{ lookup('env', 'USER') }}/network-backups/ios/{{ inventory_hostname }}/{{ inventory_hostname }}_pre_restore_{{ lookup('pipe', 'date +%Y%m%d_%H%M%S') }}.cfg"
        mode: '0640'
      delegate_to: localhost

  tasks:
    - name: "Restore | Read backup file content"
      ansible.builtin.slurp:
        src: "{{ backup_file }}"
      register: backup_content
      delegate_to: localhost

    - name: "Restore | Push configuration to device"
      cisco.ios.ios_config:
        src: "{{ backup_content.content | b64decode }}"
        # ios_config with src: pushes a full config block
        # IOS merges it with running config — does not replace entirely
        # For a true replace: use 'replace: config' parameter (IOS-XE 16.x+)
      register: restore_result
      notify: Save IOS configuration

    - name: "Restore | Verify device is still reachable"
      cisco.ios.ios_facts:
        gather_subset: [default]
      register: post_restore_facts

    - name: "Restore | Confirm hostname matches expected"
      ansible.builtin.assert:
        that:
          - post_restore_facts.ansible_facts.ansible_net_hostname is defined
        fail_msg: "Device unreachable after restore — check console access"
        success_msg: "PASS: Device reachable after restore (hostname={{ post_restore_facts.ansible_facts.ansible_net_hostname }})"

  handlers:
    - name: Save IOS configuration
      cisco.ios.ios_command:
        commands: [write memory]
      listen: Save IOS configuration

  post_tasks:
    - name: "Post-restore | Log restore event"
      ansible.builtin.lineinfile:
        path: "/home/{{ lookup('env', 'USER') }}/network-backups/backup_run.log"
        line: "{{ lookup('pipe', 'date --iso-8601=seconds') }} | RESTORE | {{ inventory_hostname }} | file={{ backup_file | basename }} | user={{ lookup('env', 'USER') }}"
        create: true
        mode: '0644'
      delegate_to: localhost
      tags: always
EOF
```

---

## 30.5 — Compliance Checking as a Validation Role Module

The compliance check integrates into the `network_validate` role from Part 22 as an additional check module. It compares running configuration against a list of required and prohibited lines defined in group_vars.

### Compliance Data Model

```bash
cat >> ~/projects/ansible-network/inventory/group_vars/cisco_ios.yml << 'EOF'

# ── Configuration compliance requirements ─────────────────────────
# Required: these lines MUST be present in running-config
# Prohibited: these lines MUST NOT be present in running-config
compliance:
  required:
    - pattern: "service password-encryption"
      description: "Password encryption must be enabled"
      severity: critical

    - pattern: "no ip http server"
      description: "HTTP server must be disabled"
      severity: critical

    - pattern: "no ip http secure-server"
      description: "HTTPS server must be disabled (use SSH only)"
      severity: high

    - pattern: "ip ssh version 2"
      description: "SSH must be version 2 only"
      severity: critical

    - pattern: "logging buffered"
      description: "Local logging buffer must be configured"
      severity: medium

    - pattern: "ntp server"
      description: "At least one NTP server must be configured"
      severity: high

    - pattern: "aaa new-model"
      description: "AAA must be enabled"
      severity: high
      conditional: "{{ aaa.enabled | default(false) }}"
      # Conditional compliance: only required when aaa.enabled is true

  prohibited:
    - pattern: "transport input telnet"
      description: "Telnet must not be permitted on VTY lines"
      severity: critical

    - pattern: "enable password "
      description: "Enable password (type 7) must not be used — use enable secret"
      severity: critical

    - pattern: "username .* password "
      regex: true
      description: "User passwords must use 'secret' not 'password' (type 7)"
      severity: critical

    - pattern: "snmp-server community public"
      description: "Default SNMP community string must not be used"
      severity: high

    - pattern: "snmp-server community private"
      description: "Default SNMP community string must not be used"
      severity: high

    - pattern: "ip finger"
      description: "Finger service must be disabled"
      severity: medium

    - pattern: "service tcp-small-servers"
      description: "TCP small servers must be disabled"
      severity: medium
EOF
```

### Compliance Check Task File

```bash
mkdir -p ~/projects/ansible-network/roles/network_validate/tasks/checks

cat > ~/projects/ansible-network/roles/network_validate/tasks/checks/compliance.yml << 'EOF'
---
# checks/compliance.yml — Configuration compliance checking
# Added to the network_validate role alongside interface/bgp/ospf checks
# Reads running config once, checks all required and prohibited patterns

- name: "Compliance | IOS: gather running configuration"
  cisco.ios.ios_command:
    commands:
      - show running-config
  register: _compliance_running_config
  changed_when: false
  when: _validate_platform == 'ios'

- name: "Compliance | NX-OS: gather running configuration"
  cisco.nxos.nxos_command:
    commands:
      - show running-config
  register: _compliance_running_config
  changed_when: false
  when: _validate_platform == 'nxos'

- name: "Compliance | Junos: gather running configuration"
  junipernetworks.junos.junos_command:
    commands:
      - show configuration | display set
    display: text
  register: _compliance_running_config
  changed_when: false
  when: _validate_platform == 'junos'

# ── Check REQUIRED lines ───────────────────────────────────────────
- name: "Compliance | Check required configuration lines are present"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'compliance_required',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'pattern': item.pattern,
        'description': item.description,
        'severity': item.severity | default('medium'),
        'expected': 'present',
        'actual': 'present' if (
          item.regex | default(false) | bool and
          _compliance_running_config.stdout[0] | regex_search(item.pattern)
        ) or (
          not item.regex | default(false) | bool and
          item.pattern in _compliance_running_config.stdout[0]
        ) else 'MISSING',
        'result': 'PASS' if (
          (item.conditional | default(true)) and (
            (item.regex | default(false) | bool and
             _compliance_running_config.stdout[0] | regex_search(item.pattern))
            or
            (not item.regex | default(false) | bool and
             item.pattern in _compliance_running_config.stdout[0])
          )
        ) or not (item.conditional | default(true)) else 'FAIL',
        'detail': item.description + ' [' + (item.severity | default('medium') | upper) + ']'
      }] }}
  loop: "{{ compliance.required | default([]) }}"
  loop_control:
    label: "Required: {{ item.pattern }}"
  when: _compliance_running_config is defined

# ── Check PROHIBITED lines ─────────────────────────────────────────
- name: "Compliance | Check prohibited configuration lines are absent"
  ansible.builtin.set_fact:
    _validate_results: >-
      {{ _validate_results + [{
        'check': 'compliance_prohibited',
        'device': inventory_hostname,
        'platform': _validate_platform,
        'pattern': item.pattern,
        'description': item.description,
        'severity': item.severity | default('medium'),
        'expected': 'absent',
        'actual': 'PRESENT' if (
          item.regex | default(false) | bool and
          _compliance_running_config.stdout[0] | regex_search(item.pattern)
        ) or (
          not item.regex | default(false) | bool and
          item.pattern in _compliance_running_config.stdout[0]
        ) else 'absent',
        'result': 'FAIL' if (
          (item.regex | default(false) | bool and
           _compliance_running_config.stdout[0] | regex_search(item.pattern))
          or
          (not item.regex | default(false) | bool and
           item.pattern in _compliance_running_config.stdout[0])
        ) else 'PASS',
        'detail': item.description + ' [' + (item.severity | default('medium') | upper) + '] — PROHIBITED'
      }] }}
  loop: "{{ compliance.prohibited | default([]) }}"
  loop_control:
    label: "Prohibited: {{ item.pattern }}"
  when: _compliance_running_config is defined

# ── Compliance summary ─────────────────────────────────────────────
- name: "Compliance | Summarize compliance results"
  ansible.builtin.set_fact:
    _compliance_critical_failures: >-
      {{ _validate_results
         | selectattr('check', 'match', 'compliance_.*')
         | selectattr('result', 'equalto', 'FAIL')
         | selectattr('severity', 'equalto', 'critical')
         | list }}
    _compliance_high_failures: >-
      {{ _validate_results
         | selectattr('check', 'match', 'compliance_.*')
         | selectattr('result', 'equalto', 'FAIL')
         | selectattr('severity', 'equalto', 'high')
         | list }}

- name: "Compliance | Display compliance summary"
  ansible.builtin.debug:
    msg:
      - "Compliance results for {{ inventory_hostname }}:"
      - "  Critical failures: {{ _compliance_critical_failures | length }}"
      - "  High failures:     {{ _compliance_high_failures | length }}"
      - >-
        {% if _compliance_critical_failures | length > 0 %}
        CRITICAL violations:
        {% for f in _compliance_critical_failures %}
          ⛔ {{ f.description }}
        {% endfor %}
        {% endif %}
EOF
```

### Enabling Compliance in the Validation Role

Add compliance as a toggleable check in the validation role's `defaults/main.yml` and `tasks/main.yml`:

```bash
# Add to roles/network_validate/defaults/main.yml
cat >> ~/projects/ansible-network/roles/network_validate/defaults/main.yml << 'EOF'
validate_compliance: true     # Check configuration compliance
validate_compliance_fail_on_critical: true   # Fail play on critical violations
EOF

# Add to roles/network_validate/tasks/main.yml (after the existing check includes)
cat >> ~/projects/ansible-network/roles/network_validate/tasks/main.yml << 'EOF'

- name: "Validate | Run compliance checks"
  ansible.builtin.include_tasks: "checks/compliance.yml"
  when: validate_compliance | bool
EOF
```

### Running Compliance as a Standalone Check

```bash
# Run compliance only (skip interface/bgp/ospf checks)
ansible-playbook playbooks/validate/validate_network.yml \
  -e "validate_interfaces=false validate_bgp=false validate_ospf=false \
      validate_vlans=false validate_routing=false validate_compliance=true"

# Run full validation including compliance
ansible-playbook playbooks/validate/validate_network.yml

# Check compliance against a single device
ansible-playbook playbooks/validate/validate_network.yml \
  --limit wan-r1 \
  -e "validate_compliance=true"
```

### Viewing Compliance in the Validation Report

The compliance results flow directly into the existing validation report from Part 22 because they use the same `_validate_results` accumulator. The text report automatically shows compliance failures alongside interface and protocol checks:

```
[PASS] interface_state    GigabitEthernet1 is up
[PASS] bgp_neighbor       BGP neighbor 10.0.0.2 present
[FAIL] compliance_required Password encryption must be enabled [CRITICAL]
[FAIL] compliance_prohibited Telnet must not be permitted [CRITICAL]
[PASS] compliance_required SSH must be version 2 only [CRITICAL]
[FAIL] compliance_prohibited Default SNMP community string [HIGH]
```

---

## 30.6 — Querying the Backup Git History

With configs committed to Git on every backup run, the history becomes a change audit trail:

```bash
cd ~/network-backups

# Show all backups for a specific device
git log --oneline -- ios/wan-r1/

# Show what changed on wan-r1 between two dates
git log --oneline --after="2025-03-01" --before="2025-03-16" -- ios/wan-r1/

# Show the actual config diff between two commits
git diff abc1234..def5678 -- ios/wan-r1/wan-r1_running_LATEST.cfg

# Find when a specific string was added to wan-r1's config
git log -S "router bgp 65001" -- ios/wan-r1/

# Show all pre-change and post-change commits for a ticket
git log --oneline --grep="CHG0001234"

# Compare pre-change and post-change configs for a ticket
PRE=$(git log --oneline --grep="pre-change.*CHG0001234" | head -1 | awk '{print $1}')
POST=$(git log --oneline --grep="post-change.*CHG0001234" | head -1 | awk '{print $1}')
git diff ${PRE}..${POST} -- ios/wan-r1/
```

---


Configuration backup, restore, change management, and compliance checking are complete — a universal backup playbook covering four platforms with Git-backed history, a change workflow that wraps every deployment in pre/post backups, a gated restore playbook, and compliance checking integrated into the existing validation role. The guide now covers the full operational lifecycle of network automation.

