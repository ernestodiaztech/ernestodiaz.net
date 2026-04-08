---
draft: false
title: '27 - Cron'
description: "Part 27 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 27
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 27: Scheduling Automation with Cron

> *A playbook that only runs when someone manually executes it is half-automated. Configuration backups that depend on an engineer remembering to run them will be missed. Compliance checks that only happen before audits find problems too late. Cron turns one-time playbooks into recurring operational processes — and understanding both its capabilities and its limits is what determines when it's the right tool and when something more capable is needed.*

---

## 27.1 — Cron Fundamentals

Cron is Ubuntu's built-in job scheduler. It runs commands at specified times by reading entries from a crontab file. Each user has their own crontab, and there's a system-level crontab for root-owned jobs. For network automation on the control node, all jobs run under the user account that owns the Ansible project and virtualenv.

### The Five Fields

Every crontab entry has five time fields followed by the command:

```
┌─────────── minute        (0-59)
│ ┌───────── hour          (0-23)
│ │ ┌─────── day of month  (1-31)
│ │ │ ┌───── month         (1-12 or jan-dec)
│ │ │ │ ┌─── day of week   (0-7, both 0 and 7 = Sunday, or mon-sun)
│ │ │ │ │
* * * * *   command to run
```

Field values accept four forms:

```
*           Any value           — every minute, every hour, etc.
5           Exact value         — only when the field equals 5
1-5         Range               — values 1 through 5 inclusive
*/15        Step                — every 15 units (0, 15, 30, 45)
1,15,30     List                — exactly these values
```

Common schedule patterns in plain English:

```
0 2 * * *       Every day at 02:00
0 2 * * 0       Every Sunday at 02:00
0 2 * * 1-5     Weekdays at 02:00
*/15 * * * *    Every 15 minutes
0 */6 * * *     Every 6 hours (00:00, 06:00, 12:00, 18:00)
0 2 1 * *       First day of every month at 02:00
0 2 1 1 *       January 1st at 02:00 (annual)
```

Special strings can replace the five fields entirely:

```
@reboot         Run once at system startup
@hourly         Same as: 0 * * * *
@daily          Same as: 0 0 * * *
@weekly         Same as: 0 0 * * 0
@monthly        Same as: 0 0 1 * *
@annually       Same as: 0 0 1 1 *
```

### Managing the Crontab

```bash
# Edit current user's crontab (opens in $EDITOR)
crontab -e

# List current user's crontab
crontab -l

# Remove current user's crontab entirely (careful)
crontab -r

# Edit another user's crontab (as root)
sudo crontab -u netops -e

# Crontab files are stored at:
# /var/spool/cron/crontabs/<username>
```

### The Two Critical Cron Pitfalls

**Pitfall 1 — PATH is minimal in cron**

Cron runs with a stripped-down PATH: `/usr/bin:/bin`. Commands that work in a terminal may fail in cron because their binary isn't in the cron PATH.

```bash
# This works in a terminal (ansible in /home/user/ansible-venv/bin):
ansible-playbook playbooks/backup.yml

# This FAILS silently in cron — ansible not found:
0 2 * * *  ansible-playbook /home/netops/projects/ansible-network/playbooks/backup.yml

# Fix 1: use the full path to the binary
0 2 * * *  /home/netops/ansible-venv/bin/ansible-playbook ...

# Fix 2: set PATH at the top of the crontab file
PATH=/home/netops/ansible-venv/bin:/usr/local/bin:/usr/bin:/bin
0 2 * * *  ansible-playbook ...
```

**Pitfall 2 — The virtualenv is not activated**

Even using the full path to the `ansible-playbook` binary, Python libraries in the virtualenv may not be accessible unless the venv is properly activated. The safest approach is to use the full path to the venv's Python explicitly, or to activate the venv in a wrapper script before calling Ansible.

```bash
# Fragile: PATH to binary but libraries may not resolve
/home/netops/ansible-venv/bin/ansible-playbook backup.yml

# Reliable: explicitly source the venv activation
/bin/bash -c 'source /home/netops/ansible-venv/bin/activate && ansible-playbook backup.yml'

# Most reliable: use a wrapper script (see Section 27.2)
/home/netops/projects/ansible-network/scripts/run_playbook.sh backup.yml
```

---

## 27.2 — The Wrapper Script Pattern

The recommended pattern for all production cron jobs. A single wrapper script handles venv activation, working directory, logging, and a lock file to prevent overlapping runs. Each cron entry calls this script with the playbook name as the argument.

```bash
cat > ~/projects/ansible-network/scripts/run_playbook.sh << 'EOF'
#!/bin/bash
# run_playbook.sh — Wrapper for cron-scheduled Ansible playbook runs
#
# Usage: run_playbook.sh <playbook_path> [extra ansible-playbook args]
# Example: run_playbook.sh playbooks/backup/backup_all.yml
#          run_playbook.sh playbooks/validate/validate_network.yml --limit wan-r1
#
# Handles:
#   - Virtual environment activation
#   - Working directory
#   - Timestamped log files
#   - Lock file to prevent overlapping runs
#   - Exit code logging and alerting
#   - Log rotation (keeps last 30 runs per playbook)

set -euo pipefail

# ── Configuration ─────────────────────────────────────────────────
PROJECT_DIR="/home/$(whoami)/projects/ansible-network"
VENV_DIR="/home/$(whoami)/ansible-venv"
LOG_BASE_DIR="${PROJECT_DIR}/logs/cron"
LOCK_DIR="/tmp/ansible-locks"
VAULT_PASSWORD_FILE="${PROJECT_DIR}/.vault/lab.txt"
MAX_LOG_FILES=30    # Keep last 30 log files per playbook

# ── Argument handling ─────────────────────────────────────────────
if [ $# -lt 1 ]; then
    echo "Usage: $0 <playbook_path> [extra args]"
    exit 1
fi

PLAYBOOK="$1"
shift
EXTRA_ARGS="${@:-}"

# Derive a safe name for log files from the playbook path
PLAYBOOK_NAME=$(basename "${PLAYBOOK}" .yml)
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_DIR="${LOG_BASE_DIR}/${PLAYBOOK_NAME}"
LOG_FILE="${LOG_DIR}/${PLAYBOOK_NAME}_${TIMESTAMP}.log"
LOCK_FILE="${LOCK_DIR}/${PLAYBOOK_NAME}.lock"

# ── Setup ──────────────────────────────────────────────────────────
mkdir -p "${LOG_DIR}" "${LOCK_DIR}"

# ── Lock file: prevent overlapping runs ───────────────────────────
if [ -f "${LOCK_FILE}" ]; then
    LOCK_PID=$(cat "${LOCK_FILE}")
    if kill -0 "${LOCK_PID}" 2>/dev/null; then
        echo "[$(date)] SKIP: ${PLAYBOOK_NAME} already running (PID ${LOCK_PID})" \
            >> "${LOG_DIR}/skip.log"
        exit 0    # Exit cleanly — not an error, just a skip
    else
        # Stale lock file from a crashed run — remove it
        echo "[$(date)] Removing stale lock file for PID ${LOCK_PID}" \
            >> "${LOG_DIR}/skip.log"
        rm -f "${LOCK_FILE}"
    fi
fi

# Write current PID to lock file, remove on exit
echo $$ > "${LOCK_FILE}"
trap 'rm -f "${LOCK_FILE}"' EXIT

# ── Activate virtual environment ───────────────────────────────────
# shellcheck source=/dev/null
source "${VENV_DIR}/bin/activate"

# ── Run the playbook ───────────────────────────────────────────────
cd "${PROJECT_DIR}"

{
    echo "════════════════════════════════════════════════════"
    echo " Ansible Playbook Run"
    echo " Playbook:  ${PLAYBOOK}"
    echo " Started:   $(date)"
    echo " User:      $(whoami)"
    echo " Host:      $(hostname)"
    echo " Venv:      ${VENV_DIR}"
    echo " Extra args: ${EXTRA_ARGS:-none}"
    echo "════════════════════════════════════════════════════"
    echo ""
} >> "${LOG_FILE}" 2>&1

# Run ansible-playbook, capture exit code without set -e stopping us
set +e
ansible-playbook "${PLAYBOOK}" \
    --vault-password-file "${VAULT_PASSWORD_FILE}" \
    ${EXTRA_ARGS} \
    >> "${LOG_FILE}" 2>&1
EXIT_CODE=$?
set -e

# ── Log the result ─────────────────────────────────────────────────
{
    echo ""
    echo "════════════════════════════════════════════════════"
    echo " Finished: $(date)"
    echo " Exit code: ${EXIT_CODE}"
    if [ ${EXIT_CODE} -eq 0 ]; then
        echo " Status: SUCCESS"
    else
        echo " Status: FAILED — check log: ${LOG_FILE}"
    fi
    echo "════════════════════════════════════════════════════"
} >> "${LOG_FILE}" 2>&1

# ── Append summary to the run history log ─────────────────────────
echo "$(date --iso-8601=seconds) | ${PLAYBOOK_NAME} | $([ ${EXIT_CODE} -eq 0 ] && echo SUCCESS || echo FAILED) | ${LOG_FILE}" \
    >> "${LOG_BASE_DIR}/run_history.log"

# ── Alert on failure (customize as needed) ────────────────────────
if [ ${EXIT_CODE} -ne 0 ]; then
    # Option 1: Write to syslog (visible in journalctl)
    logger -t ansible-cron "FAILED: ${PLAYBOOK_NAME} — see ${LOG_FILE}"

    # Option 2: Send email (requires mail/sendmail configured)
    # echo "Ansible job FAILED: ${PLAYBOOK_NAME}" | \
    #     mail -s "Ansible Cron Failure: ${PLAYBOOK_NAME}" netops@lab.local

    # Option 3: Slack webhook (requires curl and webhook URL in env)
    # curl -s -X POST "${SLACK_WEBHOOK_URL}" \
    #     -d "{\"text\": \"⚠ Ansible cron FAILED: ${PLAYBOOK_NAME} on $(hostname)\"}"
fi

# ── Log rotation: keep last N log files per playbook ──────────────
# Count existing logs and remove oldest if over limit
LOG_COUNT=$(find "${LOG_DIR}" -name "*.log" -not -name "skip.log" | wc -l)
if [ "${LOG_COUNT}" -gt "${MAX_LOG_FILES}" ]; then
    EXCESS=$((LOG_COUNT - MAX_LOG_FILES))
    find "${LOG_DIR}" -name "*.log" -not -name "skip.log" \
        | sort | head -n "${EXCESS}" | xargs rm -f
fi

exit ${EXIT_CODE}
EOF

chmod +x ~/projects/ansible-network/scripts/run_playbook.sh
```

### The Simple Inline Alternative

For quick one-off cron entries where a full wrapper isn't warranted:

```bash
# Inline alternative: activate venv, redirect stdout+stderr to a timestamped log
# No lock file, no rotation — acceptable for low-frequency, non-critical jobs

0 2 * * * /bin/bash -c \
  'source ~/ansible-venv/bin/activate && \
   cd ~/projects/ansible-network && \
   ansible-playbook playbooks/backup/backup_all.yml \
     --vault-password-file .vault/lab.txt \
     >> logs/backup_$(date +\%Y\%m\%d).log 2>&1'
```

The `\%` is required in crontab — bare `%` characters are treated as newlines by cron and truncate the command. Always escape them as `\%` in inline cron entries.

---

## 27.3 — The Four Network Automation Cron Jobs

### Job 1 — Nightly Configuration Backup

```bash
# Purpose: Back up running configs from all devices every night
# Schedule: 02:00 daily — low-traffic window, before business hours
# On failure: generates syslog alert via wrapper, log preserved
# Retention: 30 log files per playbook (wrapper handles rotation)
#            Config backups themselves are managed in backup_all.yml

# Add to crontab (crontab -e):
0 2 * * *  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/backup/backup_all.yml
```

The backup playbook referenced here is the one built in Part 16. Backups go to `backups/` with timestamps; logs go to `logs/cron/backup_all/`.

**Verify it's working after the first run:**

```bash
# Check the run history
cat ~/projects/ansible-network/logs/cron/run_history.log

# Check the most recent backup log
ls -lt ~/projects/ansible-network/logs/cron/backup_all/ | head -5

# Verify backup files were created
ls -lt ~/projects/ansible-network/backups/ | head -20
```

### Job 2 — Daily Compliance and Validation Check

```bash
# Purpose: Run the validation role (Part 22) against all devices
# Schedule: 06:00 daily — after backup, before business hours
# On failure: alert via syslog, check logs for which device/check failed
# Output: reports/validation/ JSON and text reports

0 6 * * *  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/validate/validate_network.yml
```

**Schedule variation — more frequent during maintenance windows:**

```bash
# During active lab work: validate every 4 hours
0 */4 * * *  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
                 playbooks/validate/validate_network.yml

# Revert to daily after lab work settles
0 6 * * *    /home/netops/projects/ansible-network/scripts/run_playbook.sh \
                 playbooks/validate/validate_network.yml
```

### Job 3 — Weekly Package Update Check

```bash
# Purpose: Run the maintenance report (Part 23) — check for outdated packages and CVEs
# Schedule: Monday 07:00 — start of week, before engineers begin work
# On failure: the check itself is read-only, failures mean connectivity/env issues
# Output: reports/maintenance/ text report

0 7 * * 1  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/maintenance/check_updates.yml
```

**Separate pip-audit daily scan (security-only, fast):**

```bash
# Purpose: Daily CVE scan — catches newly published CVEs between weekly reports
# Schedule: 03:00 daily — lightweight, fast
# No playbook needed — direct pip-audit call

0 3 * * *  /bin/bash -c \
  'source ~/ansible-venv/bin/activate && \
   pip-audit --format=json \
     --output ~/projects/ansible-network/reports/maintenance/audit_$(date +\%Y\%m\%d).json \
     >> ~/projects/ansible-network/logs/cron/pip_audit.log 2>&1 && \
   if grep -q '"vulns"' ~/projects/ansible-network/reports/maintenance/audit_$(date +\%Y\%m\%d).json 2>/dev/null; then \
       logger -t pip-audit "CVE found — check audit report for $(date +\%Y\%m\%d)"; \
   fi'
```

### Job 4 — Periodic Fact Gathering and Reporting

```bash
# Purpose: Collect device facts fleet-wide and update Netbox
# Schedule: Sunday 01:00 weekly — low-traffic window, out of maintenance schedule
# On failure: Netbox may be stale — non-critical but worth alerting

0 1 * * 0  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/netbox/sync_to_netbox.yml
```

**Optional: add user audit to the weekly Sunday run:**

```bash
# Run fact gathering and user audit back to back on Sundays
0 1 * * 0  /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/netbox/sync_to_netbox.yml

30 1 * * 0 /home/netops/projects/ansible-network/scripts/run_playbook.sh \
               playbooks/security/audit_users.yml
# 30 minutes after sync_to_netbox — staggered to avoid concurrent device connections
```

### Full Crontab

```bash
# Open crontab for editing
crontab -e

# Paste the complete crontab:

# ── Environment ───────────────────────────────────────────────────
# Set PATH so cron can find system utilities
PATH=/usr/local/bin:/usr/bin:/bin

# ── Ansible control node — network automation schedule ────────────
# All jobs use run_playbook.sh wrapper for logging and lock files
# Logs: ~/projects/ansible-network/logs/cron/
# Reports: ~/projects/ansible-network/reports/

# Nightly config backup — 02:00 every day
0 2 * * *  /home/netops/projects/ansible-network/scripts/run_playbook.sh playbooks/backup/backup_all.yml

# Daily compliance check — 06:00 every day
0 6 * * *  /home/netops/projects/ansible-network/scripts/run_playbook.sh playbooks/validate/validate_network.yml

# Daily CVE scan — 03:00 every day
0 3 * * *  /bin/bash -c 'source ~/ansible-venv/bin/activate && pip-audit --format=json --output ~/projects/ansible-network/reports/maintenance/audit_$(date +\%Y\%m\%d).json >> ~/projects/ansible-network/logs/cron/pip_audit.log 2>&1'

# Weekly package update report — Monday 07:00
0 7 * * 1  /home/netops/projects/ansible-network/scripts/run_playbook.sh playbooks/maintenance/check_updates.yml

# Weekly fact gathering and Netbox sync — Sunday 01:00
0 1 * * 0  /home/netops/projects/ansible-network/scripts/run_playbook.sh playbooks/netbox/sync_to_netbox.yml

# Weekly user audit — Sunday 01:30 (30 min after Netbox sync)
30 1 * * 0 /home/netops/projects/ansible-network/scripts/run_playbook.sh playbooks/security/audit_users.yml
```

---

## 27.4 — Log Management

### Log Directory Structure

```
logs/
├── cron/
│   ├── run_history.log              ← one-line summary of every run ever
│   ├── pip_audit.log                ← pip-audit raw output
│   ├── backup_all/
│   │   ├── backup_all_20250316_020001.log
│   │   ├── backup_all_20250317_020002.log
│   │   └── skip.log                 ← entries when backup was already running
│   ├── validate_network/
│   │   ├── validate_network_20250316_060001.log
│   │   └── ...
│   ├── check_updates/
│   │   └── ...
│   └── sync_to_netbox/
│       └── ...
└── maintenance.log                  ← manual maintenance entries
```

### Viewing Logs

```bash
# See all recent cron runs across all playbooks
tail -20 ~/projects/ansible-network/logs/cron/run_history.log

# Example run_history.log output:
# 2025-03-16T02:00:01+00:00 | backup_all | SUCCESS | logs/cron/backup_all/backup_all_20250316_020001.log
# 2025-03-16T03:00:02+00:00 | pip_audit | SUCCESS | logs/cron/pip_audit.log
# 2025-03-16T06:00:01+00:00 | validate_network | FAILED | logs/cron/validate_network/validate_network_20250316_060001.log

# See all FAILED runs
grep FAILED ~/projects/ansible-network/logs/cron/run_history.log

# View a specific failed run
cat "$(grep FAILED ~/projects/ansible-network/logs/cron/run_history.log | tail -1 | awk -F'|' '{print $4}' | tr -d ' ')"

# Follow the current run live (when you know a job is running)
tail -f ~/projects/ansible-network/logs/cron/backup_all/$(ls -t ~/projects/ansible-network/logs/cron/backup_all/*.log | head -1)

# Check syslog for cron execution records and failure alerts
grep ansible-cron /var/log/syslog | tail -20
# Or with journalctl:
journalctl -t ansible-cron --since "24 hours ago"
```

### System-Level Log Rotation with logrotate

The wrapper script handles per-playbook log rotation (keeping last 30 files). For the `run_history.log` and other persistent logs, add a `logrotate` config:

```bash
sudo cat > /etc/logrotate.d/ansible-cron << 'EOF'
/home/netops/projects/ansible-network/logs/cron/run_history.log
/home/netops/projects/ansible-network/logs/cron/pip_audit.log
/home/netops/projects/ansible-network/logs/maintenance.log
{
    weekly
    rotate 52       # Keep 52 weeks (one year)
    compress
    delaycompress
    missingok
    notifempty
    create 0640 netops netops
}
EOF
```

---

## 27.5 — The Limits of Cron and When to Move to AWX

Cron is the right tool for a single-engineer lab environment running a known set of recurring playbooks. It becomes the wrong tool as the environment and team grow. Understanding exactly where cron breaks down makes the decision to move to AWX straightforward rather than premature.

### What Cron Lacks

**No job visibility.** The only way to know if a cron job ran successfully is to read a log file. There's no dashboard showing recent runs, no at-a-glance status, no way for a second engineer to see what ran while they were away without digging through files in `logs/cron/`.

**No RBAC.** Any user who can edit the crontab can run any playbook. There's no concept of "this team can run backup playbooks but not user management playbooks." Access control is all-or-nothing at the OS user level.

**No audit trail.** The run history log in the wrapper script is basic — a single line per run. There's no record of which user triggered a job, what inventory was used, what variables were passed, or what the exact play output was for a given historical run.

**No on-demand triggering with controls.** Manually running a cron-managed playbook requires SSH access to the control node and knowledge of the correct command. There's no web UI or API to trigger a job, pass extra variables, or limit to a subset of devices without modifying files.

**No notifications beyond syslog/email.** The wrapper script can send syslog entries and emails, but integrating Slack, PagerDuty, ServiceNow, or any other notification system requires custom scripting for each playbook.

**No parallel job management.** The lock file in the wrapper prevents the same playbook from running twice concurrently, but there's no coordination between different playbooks. Two jobs can run simultaneously and contend for device connections.

### What AWX Adds

AWX is the open-source upstream of Red Hat Ansible Automation Platform. It provides a web UI and REST API on top of Ansible — the playbooks are identical, but how they're managed, triggered, and monitored changes entirely.

```
Cron → AWX: What changes

Visibility:
  Cron:  Read log files to see what ran
  AWX:   Dashboard shows all jobs, status, duration, output — searchable

RBAC:
  Cron:  OS user permission — all or nothing
  AWX:   Organizations, teams, roles — "network-ops can run backups,
         only network-admins can run user management playbooks"

Job history:
  Cron:  Log files in ~/logs/cron/ — manual retention
  AWX:   Full job history with stdout, variables, inventory used,
         credential used, triggered by — retained and searchable

Triggers:
  Cron:  Time-based only
  AWX:   Schedules + manual launch + REST API + webhook triggers
         (GitHub push → AWX job → deploy new config)

Credentials:
  Cron:  Vault password file on disk, readable by cron user
  AWX:   Credential store — vault passwords, SSH keys, API tokens
         stored encrypted, never visible to job runners

Inventory:
  Cron:  Static files or Netbox plugin, same for all runs
  AWX:   Multiple inventory sources, dynamic, synchronized on schedule
         Different teams can have different inventory views

Notifications:
  Cron:  Custom scripts per playbook
  AWX:   Built-in notification integrations: email, Slack, Teams,
         PagerDuty, webhook — configured once, applied to any job

Survey forms:
  Cron:  Extra vars on command line — requires SSH access
  AWX:   Web form prompts users for variables before launch
         "Which site to deploy to?" without touching the CLI
```

### The Decision Point

```
Stay on cron when:
  ✓ Single engineer or small team (2-3 people)
  ✓ All team members have SSH access to the control node
  ✓ Playbooks are stable and not frequently triggered manually
  ✓ No compliance or audit requirement for job history
  ✓ Infrastructure is lab or small-scale production

Move to AWX when:
  ✗ Multiple engineers need to trigger playbooks without SSH access
  ✗ Different team members should have different playbook permissions
  ✗ Audit/compliance requires a record of who ran what and when
  ✗ Playbooks need to be triggered by external events (CI/CD, ITSM)
  ✗ Stakeholders outside the network team need visibility into automation runs
  ✗ Managing more than ~20 devices across multiple sites
```

### Installing AWX (Preview)

AWX runs on Kubernetes. For a lab environment, the AWX Operator on k3s is the simplest path:

```bash
# This is a preview — not a complete install guide
# AWX documentation: https://github.com/ansible/awx-operator

# Prerequisites: k3s or another Kubernetes distribution
# Install k3s (single-node Kubernetes)
curl -sfL https://get.k3s.io | sh -

# Install AWX Operator
kubectl apply -f \
  https://raw.githubusercontent.com/ansible/awx-operator/main/deploy/awx-operator.yaml

# After AWX is running, the existing playbooks work without modification
# AWX uses the same playbooks, same collections, same inventory
# What changes: how jobs are launched and monitored, not the playbooks themselves
```

The key point: migrating from cron to AWX does not require rewriting any playbooks. The automation logic stays identical — only the scheduling and management layer changes.

---


Scheduling is complete — recurring operational playbooks run automatically, logs are collected and rotated, failures alert via syslog, and the boundary between cron and AWX is clearly defined. The guide now has all 27 parts: from control node setup through multi-vendor device automation, validation, security, source-of-truth integration, and scheduled operations. The capstone in Part 28 runs the entire lab from a clean state using everything built across the guide.

