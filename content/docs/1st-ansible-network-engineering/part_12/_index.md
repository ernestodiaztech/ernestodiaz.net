---
draft: false
title: '12 - Ansible Vault'
description: "Part 12 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 12
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 12: Ansible Vault — Securing Secrets

> *Every password, API token, SNMP community string, and enable secret introduced in Parts 1–11 is sitting in plain text in `group_vars/` and `host_vars/` files. They're committed to Git. Anyone who can read the repository can read every credential. This part fixes that completely — using Ansible Vault to encrypt sensitive values so the files are safe to commit, safe to share, and safe to store, while remaining transparent to Ansible at runtime.*

---

## 12.1 — What Ansible Vault Is and Why It's Non-Negotiable

Ansible Vault is Ansible's built-in encryption system. It uses AES-256 encryption to protect sensitive data in YAML files — either entire files or individual variable values. Encrypted content can be committed to Git safely: without the vault password, the ciphertext is useless.

### The Problem Without Vault

```
inventory/
└── group_vars/
    └── all.yml              ← ansible_password: ansible123 ← plain text in Git
    └── cisco_ios.yml        ← ansible_become_password: ansible123 ← plain text in Git
    └── paloalto.yml         ← (API key will be plain text)
```

This creates several real risks:

- Anyone with read access to the Git repository has every device credential
- If the repository is accidentally made public, credentials are immediately compromised
- Secrets in Git history persist even after deletion — the credential is in every clone
- Log files, CI/CD pipeline output, and backup systems may capture the plain-text values

### What Vault Solves

```
inventory/
└── group_vars/
    └── all.yml
        ansible_password: !vault |          ← AES-256 encrypted
          $ANSIBLE_VAULT;1.2;AES256;lab
          38643933396661316365343939653265...
          (safe to commit — useless without the vault password)
```

Ansible decrypts vault values transparently at runtime when the vault password is provided. The rest of the playbook system — templates, modules, variable references — works exactly the same way. The encryption is invisible once the vault password is in place.

### ### 🔴 Danger

> Ansible Vault protects credentials in files and in Git. It does not protect credentials in playbook output, log files, or terminal history. If I run a playbook with `-vvv` and a vaulted password is used in a task, the decrypted value may appear in the verbose output. Always use `no_log: true` on tasks that handle credentials directly, and never enable `-vvv` verbosity in production CI/CD pipelines.

---

## 12.2 — Setting Up Vault Passwords and Vault IDs

### The Vault ID Pattern

A vault ID is a label that identifies which password was used to encrypt a value. This allows different passwords for different environments — lab credentials encrypted with the lab password, production credentials encrypted with the production password — so a developer with lab access can't decrypt production secrets.

The format when encrypting: `--vault-id <label>@<password-source>`
The password source can be:
- `prompt` — ask for the password interactively
- A file path — read the password from a file
- A script path — execute a script that outputs the password

### Creating the Vault Password Files

I create two vault password files — one for lab and one for production:

```bash
# Create the vault password directory
mkdir -p ~/projects/ansible-network/.vault

# Lab vault password file
# In production this would be a strong randomly generated password
echo "lab-vault-password-change-me" > ~/projects/ansible-network/.vault/lab.txt
chmod 600 ~/projects/ansible-network/.vault/lab.txt

# Production vault password file
# This would be retrieved from a secrets manager in real production
echo "prod-vault-password-NEVER-COMMIT" > ~/projects/ansible-network/.vault/prod.txt
chmod 600 ~/projects/ansible-network/.vault/prod.txt
```

### Adding Vault Files to `.gitignore`

```bash
# These files must NEVER be committed to Git
cat >> ~/projects/ansible-network/.gitignore << 'EOF'

# Ansible Vault password files
.vault/
*.vault_pass
.vault_pass
EOF

# Verify they're excluded
git check-ignore -v .vault/lab.txt
# .gitignore:XX:.vault/    .vault/lab.txt   ← Confirmed excluded
```

### Configuring `ansible.cfg` to Use Vault IDs

```bash
nano ~/projects/ansible-network/ansible.cfg
```

Update the `[defaults]` section:

```ini
[defaults]
# ... existing settings ...

# Vault configuration — multiple vault IDs for lab and production
# Format: label@password_file
vault_identity_list = lab@.vault/lab.txt, prod@.vault/prod.txt

# Default vault ID to use when encrypting (without specifying --vault-id)
vault_encrypt_identity = lab
```

With this configuration:
- `ansible-playbook` automatically tries both vault passwords when decrypting
- `ansible-vault encrypt_string` uses the `lab` vault ID by default
- Production secrets encrypted with `prod@` can only be decrypted by someone with `.vault/prod.txt`

### ### ℹ️ Info

> In a real enterprise, vault passwords are never stored in files on the engineer's workstation. They're retrieved from a secrets manager — HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or CyberArk — using a small script that authenticates and returns the password. The `vault_identity_list` accepts executable scripts: `lab@~/.vault/get_lab_password.sh`. This keeps the actual passwords out of any file that could be accidentally committed or leaked. For this lab, file-based passwords are fine — just never commit the files.

---

## 12.3 — Encrypting Entire Files vs. Individual Values

There are two approaches to vaulting secrets. I'll cover both thoroughly so I can make the right choice for each situation.

### Approach 1: Encrypt Entire Files

I encrypt the complete `group_vars/` file. Every variable in the file becomes encrypted.

```bash
# Encrypt an entire file with the lab vault ID
ansible-vault encrypt --vault-id lab@.vault/lab.txt \
    inventory/group_vars/all.yml

# View the encrypted file (what gets committed to Git)
cat inventory/group_vars/all.yml
```

```
$ANSIBLE_VAULT;1.2;AES256;lab
66386565613837363866396365383563313864323565376434396432396538343038653439366132
3039613337336635373835363838393839643166363637310a323864346466323865623030623438
...
```

The entire file is now ciphertext. To read or edit it:

```bash
# View decrypted content without saving
ansible-vault view inventory/group_vars/all.yml

# Edit in-place (opens the decrypted file in the default editor)
ansible-vault edit inventory/group_vars/all.yml

# Decrypt to plain text permanently
ansible-vault decrypt inventory/group_vars/all.yml
```

**When to encrypt entire files:**

```
✅ Use whole-file encryption when:
  - The entire file is sensitive (e.g., a file containing only passwords)
  - You want the simplest possible implementation
  - The file doesn't need to be readable by people without vault access
  - You have a separate vars file just for secrets (recommended pattern)

❌ Avoid whole-file encryption when:
  - The file mixes sensitive and non-sensitive variables
  - Other engineers need to read the non-sensitive values without the password
  - You want git diff to show meaningful changes to non-sensitive values
  - The file is used as a reference document (encrypted files aren't readable)
```

### Approach 2: Encrypt Individual Variable Values with `encrypt_string`

I encrypt only the sensitive values inline, leaving non-sensitive variables readable. This is the best practice for `group_vars/` files that mix credentials with configuration data.

```bash
# Encrypt a single value
ansible-vault encrypt_string --vault-id lab@.vault/lab.txt \
    'ansible123' \
    --name 'ansible_password'
```

Output:
```yaml
ansible_password: !vault |
          $ANSIBLE_VAULT;1.2;AES256;lab
          38643933396661316365343939653265653661353563376334643065636135376162643662633333
          3530313039376331396231356261636264313137323230610a613462386565326163623461633861
          63376239326361303231616262336365343665373963646135326432626561646265623862353166
          3230306437626261650a386238663933313436636266326237636232623461383965366432333966
          3938
```

I copy this output and paste it directly into the `group_vars/` file, replacing the plain-text value.

**The Recommended Pattern — Separate Vault Files**

Rather than inline `encrypt_string` scattered throughout every file, the cleanest approach is to create a dedicated vault file alongside each `group_vars/` file. The vault file contains only the sensitive variables; the main file contains only non-sensitive variables and references the vault file through variable names.

```
inventory/group_vars/
├── all.yml           ← Non-sensitive vars + references vault vars by name
├── all_vault.yml     ← Encrypted — contains only the sensitive values
├── cisco_ios.yml     ← Non-sensitive connection settings
└── cisco_ios_vault.yml  ← Encrypted — contains only the become password
```

The naming convention `<file>_vault.yml` is the community standard. Ansible automatically loads all `.yml` files in `group_vars/<group>/` (if using a directory) or the explicitly named files.

### ### 💡 Tip

> The split-file pattern has a major Git advantage: changes to non-sensitive variables in `all.yml` show up as readable diffs. Only changes to `all_vault.yml` are opaque. When a team member updates an NTP server in `all.yml`, I can review the change in a Pull Request. When a team member rotates a password in `all_vault.yml`, the diff shows encrypted ciphertext changed — I know a rotation happened without seeing the old or new password.

---

## 12.4 — Implementing Vault in the Lab: A Complete Walkthrough

Now I walk through the complete process of vaulting every sensitive credential introduced in Parts 1–11. I'll do this thoroughly for `group_vars/all.yml`, then note the pattern for all remaining files.

### Step 1: Identify All Sensitive Values

First, I catalog what needs to be vaulted across the project:

```bash
# Find plain-text passwords and tokens across the project
grep -r "ansible_password\|ansible_become_password\|snmp_community\|api_key\|token" \
    inventory/ --include="*.yml" -l
```

Files containing sensitive values from Parts 1–11:

```
inventory/group_vars/all.yml              → ansible_password, snmp_community_ro
inventory/group_vars/cisco_ios.yml        → ansible_become_password
inventory/group_vars/paloalto.yml         → (API key to be added)
inventory/host_vars/wan-r1.yml            → ansible_become_password (if overridden)
```

### Step 2: Create the Vault File for `group_vars/all`

```bash
# Create the vault file with only sensitive values
nano ~/projects/ansible-network/inventory/group_vars/all_vault.yml
```

```yaml
---
# =============================================================
# all_vault.yml — Encrypted sensitive values for all devices
# This file is encrypted with: ansible-vault encrypt all_vault.yml
# Edit with: ansible-vault edit all_vault.yml
# =============================================================

# SSH password for all devices (will be overridden per-platform)
vault_ansible_password: "ansible123"

# SNMP read-only community string
vault_snmp_community_ro: "lab-read-only"
```

### Step 3: Update `group_vars/all.yml` to Reference Vault Variables

```bash
nano ~/projects/ansible-network/inventory/group_vars/all.yml
```

```yaml
---
# =============================================================
# all.yml — Global variables for all devices
# Sensitive values are in all_vault.yml (encrypted)
# =============================================================

# --- Credentials (values defined in all_vault.yml) ---
ansible_user: ansible
ansible_password: "{{ vault_ansible_password }}"    # ← References vault variable

# --- SNMP ---
snmp_community_ro: "{{ vault_snmp_community_ro }}"  # ← References vault variable

# --- NTP Servers ---
ntp_servers:
  - 216.239.35.0
  - 216.239.35.4

# --- DNS Servers ---
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

# --- Syslog ---
syslog_server: 172.16.0.100

# --- Domain ---
domain_name: lab.local

# --- Timeouts ---
ansible_persistent_connect_timeout: 30
ansible_command_timeout: 30
```

The pattern: `all_vault.yml` defines `vault_ansible_password`. `all.yml` uses `{{ vault_ansible_password }}`. Ansible loads both files, resolves the reference, and the module receives the decrypted value.

**Why prefix vault variables with `vault_`?**
The `vault_` prefix is a community convention that makes it immediately clear which variables are sourced from a vault file. When I see `ansible_password: "{{ vault_ansible_password }}"`, I know to look in `all_vault.yml` for the actual value. Without this convention, tracking which variables are sensitive and where they're defined becomes confusing in large projects.

### Step 4: Encrypt the Vault File

```bash
cd ~/projects/ansible-network

# Encrypt the vault file with the lab vault ID
ansible-vault encrypt --vault-id lab@.vault/lab.txt \
    inventory/group_vars/all_vault.yml
```

Output:
```
Encryption successful
```

```bash
# Verify it's encrypted
cat inventory/group_vars/all_vault.yml
# $ANSIBLE_VAULT;1.2;AES256;lab
# 66386565613837363866396365383563...
```

```bash
# Verify Ansible can read it (decrypts automatically using vault_identity_list)
ansible-inventory --host wan-r1 | grep ansible_password
# "ansible_password": "ansible123"   ← Decrypted correctly at runtime
```

### Step 5: Create and Encrypt the IOS Vault File

```bash
# Create
cat > inventory/group_vars/cisco_ios_vault.yml << 'EOF'
---
# cisco_ios_vault.yml — Encrypted IOS credentials
vault_ansible_become_password: "ansible123"
EOF

# Encrypt
ansible-vault encrypt --vault-id lab@.vault/lab.txt \
    inventory/group_vars/cisco_ios_vault.yml
```

Update `group_vars/cisco_ios.yml`:

```yaml
---
# cisco_ios.yml — IOS connection settings
# Sensitive values are in cisco_ios_vault.yml (encrypted)

ansible_network_os: cisco.ios.ios
ansible_connection: network_cli
ansible_become: true
ansible_become_method: enable
ansible_become_password: "{{ vault_ansible_become_password }}"  # ← From vault

ansible_terminal_length: 0
ios_resource_modules_supported: true
backup_dir: backups/cisco_ios
```

### Step 6: Verify the Vault Setup Works

```bash
# Test that a playbook runs correctly with vaulted credentials
ansible cisco_ios -m ansible.netcommon.net_ping

# If this returns SUCCESS, the vault decryption is working correctly
# wan-r1 | SUCCESS => {"changed": false, "ping": "pong"}
```

If I see `FAILED` with an authentication error, the vault password file path in `ansible.cfg` is wrong or the `.vault/` directory permissions need checking.

### Files That Need Vaulting — Complete List

Following this exact pattern for every file:

| File | Create vault file | Sensitive variables to move |
|---|---|---|
| `group_vars/all.yml` | `all_vault.yml` | `ansible_password`, `snmp_community_ro` |
| `group_vars/cisco_ios.yml` | `cisco_ios_vault.yml` | `ansible_become_password` |
| `group_vars/cisco_nxos.yml` | `cisco_nxos_vault.yml` | (none currently — no become needed) |
| `group_vars/paloalto.yml` | `paloalto_vault.yml` | API key/token (when added in Part 21) |
| `host_vars/*/` | Per-device vault file if needed | Device-specific enable passwords |

For each file: create the `_vault.yml` companion, move sensitive values into it with `vault_` prefix, update the main file to reference `{{ vault_* }}` variables, encrypt the vault file.

---

## 12.5 — Working with Encrypted Files Day-to-Day

### Viewing an Encrypted File

```bash
# View decrypted content without modifying the file
ansible-vault view inventory/group_vars/all_vault.yml
```

### Editing an Encrypted File

```bash
# Opens decrypted content in $EDITOR (nano/vim), re-encrypts on save
ansible-vault edit inventory/group_vars/all_vault.yml
```

This is the correct way to change a vaulted value. The file is decrypted in memory, opened in the editor, and re-encrypted when I save and quit. The plaintext never touches disk.

### Rekeying — Changing the Vault Password

```bash
# Change the vault password for a file (old password → new password)
ansible-vault rekey --vault-id lab@.vault/lab.txt \
    --new-vault-id lab@prompt \
    inventory/group_vars/all_vault.yml
# Prompts for the new password interactively
```

### Rotating Credentials

When a device password needs to be rotated:

```bash
# Step 1: Edit the vault file with the new password
ansible-vault edit inventory/group_vars/all_vault.yml
# Change vault_ansible_password to the new value, save

# Step 2: Push the new password to all devices
ansible-playbook playbooks/deploy/deploy_base_config.yml \
    --tags credentials

# Step 3: Commit the updated (still encrypted) vault file to Git
git add inventory/group_vars/all_vault.yml
git commit -m "chore(vault): rotate ansible SSH password — effective $(date +%Y-%m-%d)"
```

The commit message notes a rotation happened without revealing old or new values. The Git diff shows ciphertext changed — proof of rotation without leaking the credentials.

### ### 🏢 Real-World Scenario

> Credential rotation is one of the most operationally painful tasks in network management without automation. In an environment I know of, rotating the `ansible` user password on 300 network devices took two engineers half a day — manually SSHing to each device, changing the password, updating a spreadsheet, then updating each Ansible inventory file one by one. With Ansible Vault and a credential rotation playbook, the process became: update the vault file with the new password, run `ansible-playbook rotate_credentials.yml`, commit the encrypted vault file. Twelve minutes for 300 devices. The vault-first design is what made this possible — because the credentials were already centralized in one place.

---

## 12.6 — Using Multiple Vault IDs for Dev/Prod Separation

The multiple vault ID setup from Section 12.2 enables a critical security pattern: developers have the lab vault password but not the production vault password. Production secrets are encrypted with a different key that only the CI/CD system and senior engineers hold.

### The Environment Separation Pattern

```
inventory/
├── group_vars/
│   ├── all.yml                  ← Non-sensitive, readable by everyone
│   ├── all_vault.yml            ← Encrypted with lab@ vault ID
│   └── all_vault_prod.yml       ← Encrypted with prod@ vault ID
```

```yaml
# all.yml — references both lab and prod vault variables
ansible_password: "{{ vault_ansible_password }}"     # Lab devices
prod_api_token: "{{ vault_prod_api_token }}"         # Production API
```

```yaml
# all_vault.yml (encrypted with lab@ — developers can decrypt)
vault_ansible_password: "ansible123"   # Lab password
```

```yaml
# all_vault_prod.yml (encrypted with prod@ — developers CANNOT decrypt)
vault_prod_api_token: "actual-production-token-here"
```

### Encrypting with a Specific Vault ID

```bash
# Encrypt the lab vault file with the lab ID
ansible-vault encrypt --vault-id lab@.vault/lab.txt \
    inventory/group_vars/all_vault.yml

# Encrypt the production vault file with the prod ID
ansible-vault encrypt --vault-id prod@.vault/prod.txt \
    inventory/group_vars/all_vault_prod.yml
```

### Running Playbooks with Specific Vault IDs

```bash
# Lab run — only needs lab vault password
ansible-playbook playbooks/deploy/site.yml \
    --vault-id lab@.vault/lab.txt

# Production run — needs both passwords (or CI/CD injects them)
ansible-playbook playbooks/deploy/site.yml \
    --vault-id lab@.vault/lab.txt \
    --vault-id prod@.vault/prod.txt

# Or rely on vault_identity_list in ansible.cfg (automatically tries both)
ansible-playbook playbooks/deploy/site.yml
```

### Verifying Which Vault ID Encrypted a File

```bash
# The first line of the encrypted file shows the vault ID label
head -1 inventory/group_vars/all_vault.yml
# $ANSIBLE_VAULT;1.2;AES256;lab   ← "lab" is the vault ID label

head -1 inventory/group_vars/all_vault_prod.yml
# $ANSIBLE_VAULT;1.2;AES256;prod  ← "prod" is the vault ID label
```

### What Happens When the Wrong Password Is Used

```bash
# If prod vault file exists but only lab password is provided:
ansible-playbook site.yml --vault-id lab@.vault/lab.txt
# ERROR! Decryption failed (no vault secrets would work here)
# The playbook fails with a vault decryption error — not with an auth error.
# This is expected and correct — the wrong person can't decrypt prod secrets.
```

### ### ℹ️ Info

> In a CI/CD pipeline (covered in Part 32), vault passwords are never stored in files. They're injected as environment variables or retrieved from the CI system's secrets store at runtime. GitLab CI stores them as protected variables. GitHub Actions stores them as encrypted secrets. Jenkins stores them in the credential store. The CI system provides `--vault-id lab@<(echo $VAULT_LAB_PASSWORD)` — using process substitution to pass the password without writing it to a file. This keeps the vault password completely out of the filesystem and out of logs.

---

## 12.7 — Using `encrypt_string` for Inline Values

`encrypt_string` is the alternative to vault files — it encrypts a single value and produces output that can be pasted directly into any YAML file. I use this for one-off sensitive values that don't warrant a separate vault file.

### Encrypting a Single Value

```bash
# Encrypt interactively (password prompt)
ansible-vault encrypt_string --vault-id lab@.vault/lab.txt \
    --stdin-name 'vault_snmp_community_ro'
# Type the value, then Ctrl+D

# Encrypt inline (value on command line — appears in shell history, use carefully)
ansible-vault encrypt_string --vault-id lab@.vault/lab.txt \
    'my-secret-value' \
    --name 'vault_my_secret'
```

Output:
```yaml
vault_my_secret: !vault |
          $ANSIBLE_VAULT;1.2;AES256;lab
          38643933396661316365343939653265653661353563376334643065636135376162643662633333
          3530313039376331396231356261636264313137323230610a613462386565326163623461633861
          63376239326361303231616262336365343665373963646135326432626561646265623862353166
          3230306437626261650a386238663933313436636266326237636232623461383965366432333966
          3938
```

I copy the entire block (from `vault_my_secret:` through the last line) and paste it directly into a `group_vars/` or `host_vars/` file.

### Verifying an `encrypt_string` Value

```bash
# Decrypt a specific variable value to verify it's correct
echo '$ANSIBLE_VAULT;1.2;AES256;lab
38643933396661316365343939653265...
3938' | ansible-vault decrypt --vault-id lab@.vault/lab.txt
```

Or use `ansible-inventory` to confirm the decrypted value appears correctly:

```bash
ansible-inventory --host wan-r1 | grep vault_my_secret
```

### When to Use `encrypt_string` vs Vault Files

| Situation | Use |
|---|---|
| One or two sensitive values in an otherwise non-sensitive file | `encrypt_string` inline |
| A whole file that's only credentials | Vault file (encrypt entire file) |
| Many sensitive values across a big vars file | Split `_vault.yml` + encrypt whole vault file |
| A sensitive value in `host_vars/` for a specific device | `encrypt_string` inline in that host_vars file |
| A value that will change frequently (API tokens, rotating passwords) | Vault file (easier to `edit` than to re-encrypt_string) |

---

## 12.8 — Best Practices Summary

### What to Vault

```
✅ ALWAYS VAULT:
  - SSH passwords (ansible_password, ansible_become_password)
  - Enable/privilege passwords
  - SNMP community strings (especially read-write)
  - API tokens and keys (Netbox, AWX, PAN-OS, cloud providers)
  - Database passwords
  - Certificate private keys and passphrases
  - Wireless PSKs and pre-shared keys
  - RADIUS/TACACS secrets

❌ NEVER VAULT (unnecessary — adds friction with no security benefit):
  - IP addresses
  - Hostnames
  - VLAN IDs
  - Interface names and descriptions
  - NTP server addresses (these are public servers anyway)
  - Non-sensitive configuration parameters
  - Anything that appears in public documentation or config examples
```

### The `no_log` Directive

Even with vault encryption, task output can reveal secrets. Add `no_log: true` to any task that handles credentials:

```yaml
- name: "Config | Create local user account"
  cisco.ios.ios_user:
    name: ansible
    password: "{{ vault_ansible_password }}"    # Vaulted
    privilege: 15
    state: present
  no_log: true    # ← Prevents the task arguments from appearing in output
                  #    Even with -vvv, the password won't be logged
```

### ### ⚠️ Warning

> `no_log: true` suppresses ALL output from the task — including error messages. If the task fails, I get `FAILED!` with no diagnostic information. During development, I sometimes temporarily remove `no_log: true` to debug, then put it back before committing. I never leave `no_log: false` (the explicit false form) on tasks that handle credentials in production playbooks.

### Vault in the `README.md`

I add a vault setup section to the project README so team members know how to get set up:

```markdown
## Vault Setup

This project uses Ansible Vault to protect sensitive credentials.

### Setup (first time)

1. Obtain the vault password from the team's password manager (entry: "ansible-network lab vault")
2. Create the vault directory: `mkdir -p .vault && chmod 700 .vault`
3. Write the password to the vault file: `echo "password_here" > .vault/lab.txt && chmod 600 .vault/lab.txt`
4. Verify setup: `ansible-inventory --host wan-r1 | grep ansible_password`

### Vault IDs

| ID | Used For | Who Has Access |
|---|---|---|
| `lab` | Lab credentials | All engineers |
| `prod` | Production credentials | Senior engineers + CI/CD |

### Rotating Credentials

See `playbooks/utils/rotate_credentials.yml` for the automated rotation playbook.
```

---

## 12.9 — AWX/AAP Credential Management Preview

When AWX is set up in Part 28, file-based vault passwords are replaced by AWX's built-in credential management system. This is worth understanding now because it changes the vault workflow for CI/CD playbook execution.

### How AWX Handles Vault

AWX has a credential type called "Vault" — it stores the vault password securely in AWX's encrypted database and injects it automatically when running jobs. The workflow changes to:

```
File-based vault (Parts 1–27):
  Engineer runs playbook → ansible reads .vault/lab.txt → decrypts vault files

AWX-based vault (Parts 28+):
  AWX job runs → AWX injects ANSIBLE_VAULT_PASSWORD → decrypts vault files
  (no .vault/ directory needed on the AWX server)
```

### Creating a Vault Credential in AWX

In the AWX web interface (preview — full details in Part 28):

1. **Credentials** → **Add**
2. **Credential Type:** Vault
3. **Name:** `Lab Vault Password`
4. **Vault Password:** (paste the lab vault password)
5. **Vault Identifier:** `lab` ← must match the vault ID label
6. **Save**

Then in a Job Template, I attach the credential. When the job runs, AWX injects the vault password as an environment variable — `ansible-playbook` receives it without it ever being written to a file on the AWX server.

### Implication for the `.vault/` Directory

When AWX runs playbooks, the `.vault/` directory doesn't exist on the AWX server. The `vault_identity_list` setting in `ansible.cfg` points to `.vault/lab.txt` which won't be there. The solution:

```ini
# ansible.cfg — conditional vault config for AWX compatibility
[defaults]
# When running locally: reads from file
# When running in AWX: AWX injects the password via environment variable
# AWX sets ANSIBLE_VAULT_PASSWORD_FILE automatically when a vault credential is attached
# So this setting is used locally only:
vault_identity_list = lab@.vault/lab.txt, prod@.vault/prod.txt
```

AWX overrides `vault_identity_list` with its own injection mechanism when a vault credential is attached to the job template. The `ansible.cfg` setting is only used for local execution. This means the same `ansible.cfg` works for both local runs (using `.vault/` files) and AWX runs (using AWX credential injection) without modification.

### ### 💡 Tip

> This dual-mode behavior — local file for development, AWX credential for CI/CD — is the standard pattern in enterprises running Ansible Automation Platform. Engineers work locally against the lab with `.vault/lab.txt`. The CI/CD pipeline (AWX) uses its credential store. The vault files in Git work the same in both cases because the encryption is identical — only the mechanism for providing the password changes.

---

## 12.10 — Common Gotchas in This Section

### ### 🪲 Gotcha — "Decryption failed" when the vault ID label doesn't match

```bash
# Encrypted with vault ID "lab"
ansible-vault encrypt --vault-id lab@.vault/lab.txt all_vault.yml

# Decryption fails if I use a different label
ansible-vault view --vault-id dev@.vault/lab.txt all_vault.yml
# ERROR: Decryption failed (no vault secrets would work here) for all_vault.yml
```

The vault ID label in the encrypted file (`lab`) must match the label used for decryption (`lab`). The password itself doesn't have to match the label — the label is just a lookup key. The actual matching is done by label first, then Ansible tries the password associated with that label.

```bash
# Fix: use the correct label
ansible-vault view --vault-id lab@.vault/lab.txt all_vault.yml
```

### ### 🪲 Gotcha — `ansible-vault edit` leaves a temporary plaintext file on disk

When I run `ansible-vault edit`, Ansible decrypts the file to a temporary location, opens it in the editor, and re-encrypts after the editor closes. On some systems, the temp file is written to `/tmp/` and can persist briefly after the editor closes.

```bash
# Use a RAM-based temp directory if available
export TMPDIR=/dev/shm    # Linux RAM filesystem — files exist only in memory
ansible-vault edit inventory/group_vars/all_vault.yml
```

This is a paranoia-level concern for most lab environments, but worth knowing for production systems where plaintext credential exposure is genuinely catastrophic.

### ### 🪲 Gotcha — `encrypt_string` value breaks YAML if not properly indented

When pasting `encrypt_string` output into a YAML file, the indentation must be consistent:

```yaml
# ❌ Wrong — misaligned continuation lines
ansible_password: !vault |
$ANSIBLE_VAULT;1.2;AES256;lab   ← Should be indented
38643933...

# ✅ Correct — all continuation lines indented 10 spaces (standard)
ansible_password: !vault |
          $ANSIBLE_VAULT;1.2;AES256;lab
          38643933...
```

The `ansible-vault encrypt_string` output includes correct indentation. The problem only arises when I manually paste and the editor strips leading spaces. I always use `yamllint` after pasting vault strings to confirm the YAML is valid:

```bash
yamllint inventory/group_vars/all_vault.yml
# If it's encrypted, yamllint can't parse it — decrypt first, lint, re-encrypt
ansible-vault decrypt --vault-id lab@.vault/lab.txt inventory/group_vars/all_vault.yml
yamllint inventory/group_vars/all_vault.yml
ansible-vault encrypt --vault-id lab@.vault/lab.txt inventory/group_vars/all_vault.yml
```

### ### 🪲 Gotcha — Committing `.vault/` or plain-text vault files to Git

```bash
# Verify that vault files are excluded BEFORE the first commit
git status
# Make sure .vault/ and any *_vault.yml files don't appear as staged

# Double-check the gitignore is working
git check-ignore -v .vault/lab.txt
# .gitignore:XX:.vault/    .vault/lab.txt

# If an unencrypted vault file was accidentally staged:
git reset HEAD inventory/group_vars/all_vault.yml
# Then encrypt it before staging again
ansible-vault encrypt --vault-id lab@.vault/lab.txt \
    inventory/group_vars/all_vault.yml
git add inventory/group_vars/all_vault.yml
```

### ### 🔴 Danger

> If a plain-text credential was committed to Git at any point — even if deleted in a subsequent commit — it is permanently in the Git history. The correct remediation is `git filter-repo` to rewrite history (which invalidates all clones), plus immediate credential rotation. A `git rm` does not solve the problem — the file is still in the history and can be recovered with `git checkout <old-commit> -- filename`. Treat any credential that touched a Git commit as permanently compromised, rotate it, and fix the process that allowed it.

---

## 12.11 — Committing the Vault Setup to Git

```bash
cd ~/projects/ansible-network

# Confirm .vault/ is gitignored
git check-ignore -v .vault/
git check-ignore -v .vault/lab.txt

# Add the encrypted vault files and updated group_vars files
git add inventory/group_vars/all.yml
git add inventory/group_vars/all_vault.yml          # ← Encrypted — safe to commit
git add inventory/group_vars/cisco_ios.yml
git add inventory/group_vars/cisco_ios_vault.yml    # ← Encrypted — safe to commit
git add ansible.cfg                                  # ← Updated with vault_identity_list
git add .gitignore                                   # ← Updated with .vault/ exclusion
git add README.md                                    # ← Updated with vault setup section

# Verify nothing sensitive is being committed
git diff --staged | grep -i "password\|secret\|token\|key" | grep -v "vault\|vault_"
# This should return nothing — any plaintext secrets would appear here

git commit -m "feat(vault): implement Ansible Vault for all sensitive credentials

- Create all_vault.yml and cisco_ios_vault.yml with vaulted credentials
- Update group_vars to reference vault_ prefixed variables
- Configure vault_identity_list in ansible.cfg (lab + prod vault IDs)
- Update .gitignore to exclude .vault/ password files
- Add vault setup instructions to README.md
- All vault files encrypted with AES-256 using lab vault ID"
```

---


Credentials are now protected. The project can be shared, the repository can be read by the whole team, and Git history is safe — all sensitive values are AES-256 encrypted. Part 13 takes everything built so far and organizes it into Ansible roles: reusable, self-contained units of automation that eliminate copy-paste between playbooks.


