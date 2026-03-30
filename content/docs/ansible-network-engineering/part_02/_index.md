---
draft: true
title: 'Gitea'
description: "Part 2 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 2
---

{{< badge "Ansible" >}}
{{< badge content="Git" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

Deploying a self-hosted Git server, configuring SSH key auth, pushing the project, and establishing branch protection.

{{< objectives title="What I Will Be Completing In This Part" >}}
- Provision a dedicated VM for Gita and install Docker + Docker Compose
- Deploy Gitea with a PostgresSQL database backend using Docker Compose
- Complete the intial Gitea web configuration and create an admin account
- Generate an SSH key pair on `ansible-ctrl` and register it in Gitea
- Create the `network-automation-lab` repository in Gitea
- Push the existing local Git history from Part 1 to the Gitea remote
- Configure branch protection on `main` with required approvals
{{< /objectives >}}

I created a dedicated VM for Gitea because I didn't want to mix other services with it. The Git server is its own piece of infrastructure and it needs to be available even if the control node is being rebuilt.

Keeping Git and PostgresSQL off the Ansible control node means Ansible always has full access to CPU and memory when running large parallel playbook executions.

---

## <span class="section-num">01</span> VM Specifications

{{< codeblock lang="Specs" copy=false >}}
OS:        Ubuntu Server 22.04 LTS
Hostname:  gitea
CPU:       2 vCPU
RAM:       2 GB
Disk:      30 GB
Network:   1 NIC (bridged to management network)
{{< /codeblock >}}

As I did with the control node VM, I also assigned a static IP and added a DNS entry for this VM.

---

## <span class="section-num">02</span> System Setup

After the OS installation, I ran system updates and installed Docker Engine and Docker Compose.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Updates the package list and upgrades all installed packages.

Line 2:
: Prerequisites for adding Docker's GPG key and repository. `ca-certificates` and `curl` handle HTTPS, `gnupg` handles key verification.
{{< /line-explain >}}

---

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Created the `/etc/apt/keyrings` directory for storing GPG keys. The `-m 0755` flag sets permissions so all users can read the directory.

Lines 2-3:
: Downloaded Docker's official GPG key and converted it from ASCII-armored format to binary.

Line 4:
: Ensured the key file is world-readable so the APT process can verify package signatures.
{{< /line-explain >}}

---

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Refreshed the package list.

Line 2:
: `docker-ce` is the Docker daemon. `docker-ce-cli` is the command-line client. `containerd.io` is the container runtime. `docker-compose-plugin` provides `docker compose` as a Docker CLI subcommand.
{{< /line-explain >}}

Next I added my user to the `docker` group so I can run Docker commands with `sudo`.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo usermod -aG docker $USER
newgrp docker
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Added my user to the `docker` group. The `-a` flag appends and `-G` specifies the supplementary group.

Line 2:
: Activated the new group membership without requiring a logout/login.
{{< /line-explain >}}

Then I verified Docker was running.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
docker version
docker compose version
{{< /codeblock >}}

---

## <span class="section-num">03</span> Docker Compose File

I created a directory structure for Gitea's Docker Compose deployment and created the Compose file. Keeping all Docker service deployments in `/opt/` makes things predictable across all VMs.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo mkdir -p /opt/gitea
sudo chown $USER:$USER /opt/gitea
cd /opt/gitea
{{< /codeblock >}}

{{< codeblock file="/opt/gitea/docker-compose.yml" syntax="bash" lines="true" >}}
version: "3"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_db_password
    restart: unless-stopped
    volumes:
      - ./data/gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
    depends_on:
      - db

  db:
    image: postgres:16
    container_name: gitea-db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_db_password
      - POSTGRES_DB=gitea
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
{{< /codeblock >}}

{{< line-explain >}}
Line 5:
: Used the official Gitea image with the `latest` tag.

Lines 8-9:
: `USER_UID` and `USER_GID` tell Gitea to run its internal processes as UID/GID 1000, which matches the default user on Ubuntu. This ensures file permissions on the mounted volumes are correct.

Lines 10-14:
: Database configuration using Gitea's environment variable convention. This tells Gitea to connect to the `db` service on port 5432 with the specified credentials.

Line 15:
: `unless-stopped` means the container restarts automatically after a crash or reboot.

Lines 17-19:
: Mounted Gitea's data directory to `./data/gitea` on the host for persistence.

Lines 21-22:
: Port `3000` is Gitea's web UI. Port `2222` maps to the container's SSH port (22).

Lines 27-35:
: PortgresSQL 16 container. The credentials must match what I gave the Gitea container.
{{< /line-explain >}}

I created an `.env` file alongside the Compose file to hold the credentials separately.

{{< codeblock file="/opt/gitea/.env" syntax="bash" lines="true" >}}
POSTGRES_USER=gitea
POSTGRES_PASSWORD=gitea_password
POSTGRES_DB=gitea
{{< /codeblock >}}

Then I updated the Compose file to reference those variables.

{{< codeblock file="/opt/gitea/docker-compose.yml" syntax="bash" lines="true" >}}
- GITEA__database__USER=${POSTGRES_USER}
- GITEA__database__PASSWD=${POSTGRES_PASSWORD}
- GITEA__database__NAME=${POSTGRES_DB}

- POSTGRES_USER=${POSTGRES_USER}
- POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
- POSTGRES_DB=${POSTGRES_DB}
{{< /codeblock >}}

Docker Compose automatically reads a `.env` file in the same directory as the `docker-compose.yml` file.

---

## <span class="section-num">04</span> Deploying Gitea

After creating the compose and `.env` files, I brought the stack up.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
cd /opt/gitea
docker compose up -d
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
: The `-d` flag runs the containers in detached mode (background).
{{< /line-explain >}}

To check the status, I ran:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
docker compose ps
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy=false >}}
NAME        IMAGE               STATUS          PORTS
gitea       gitea/gitea:latest   Up 30 seconds   0.0.0.0:3000->3000/tcp, 0.0.0.0:2222->22/tcp
gitea-db    postgres:16          Up 31 seconds   5432/tcp
{{< /codeblock >}}

If either show 'Restarting' or 'Exited', I can check the logs:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
docker compose logs gitea
docker compose logs db
{{< /codeblock >}}

---

## <span class="section-num">05</span> Initial Configuration

I went to `http://gitea:3000` and was presented with the initial configuration page. Most of the database settings should already be populated.

**Settings I Changed:**

**Site Title:** Network Automation Lab

**SSH Server Domain:** 10.33.99.62

**SSH Server Port:** 2222

**Gitea Base URL:** http://gitea:3000/

---

Then clicked on 'Install Gitea'.

---

## <span class="section-num">06</span> SSH Key Authentication

To make it so I can run `git push` and `git pull` over SSH without prompting for a password I setup SSH key authentication.

On `ansible-ctrl` I genereated an ed25519 key pair.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
ssh-keygen -t ed25519 -C "ansible-ctrl > gitea" -f ~/.ssh/gitea
{{< /codeblock >}}

{{< line-explain >}}
-t ed25519:
: Ed25519 is the modern default for SSH keys.

-C:
: A comment embedded in the public key.

-f:
: Output file path.
{{< /line-explain >}}

I started `ssh-agent` and added the key so the passphrase only needs to be entered once per session

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
(.venv) $ eval "$(ssh-agent -s)"
Agent pid 12345
(.venv) $ ssh-add ~/.ssh/gitea
Enter passphrase for /home/nesto/.ssh/gitea:
Identity added: /home/nesto/.ssh/gitea
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Starts the ssh-agent daemon and sets the necessary environment variables in the current shell. The `eval` wrapper ensures the `SSH_AUTH_SOCK` and `SSH_AGENT_PID` variables are exported.

Line 3:
: Adds the Gitea private key to the agent.
{{< /line-explain >}}

I set a strong passphrase when prompted since a key without a passphrase is equivalent to a password written on a sticky note.

So I didn't have to start the ssh-agent manually on every login, I added the following to `~/.bashrc`:

{{< codeblock file="~/.bashrc" >}}
# Start ssh-agent if not already running
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/gitea 2>/dev/null
fi
{{< /codeblock >}}

This will start the agent and load the key on login. The passphrase prompt will appear once, and then all following Git and SSH operations will use the cached key.

---

Next, I configured SSH to use this specific key when conencting to Gitea:

{{< codeblock file="~/.bashrc" >}}
Host gitea
    HostName      gitea
    Port          2222
    User          git
    IdentityFile  ~/.ssh/gitea
{{< /codeblock >}}

{{< line-explain >}}
Host gitea:
: This is the alias that SSH (and Git) will match on.

HostName:
: The actual hostname or IP address to connect to.

Port 2222:
: Maps to the Docker port I configured in the Compose file.

User git:
: Gitea handles all SSH Git operations under the `git` user.

IdentityFile:
: Points SSH to the specific private key for this host.
{{< /line-explain >}}

I then set the proper permissions on `ansible-ctrl`.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/gitea
chmod 644 ~/.ssh/gitea.pub
{{< /codeblock >}}

{{< line-explain >}}
Lines 1-2:
: The SSH config and private key must be readable only by the owner.

Line 3:
: The public key can be readable by anyone.
{{< /line-explain >}}

I then copied the public key conent and added it to Gitea.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
cat ~/.ssh/gitea.pub
{{< /codeblock >}}

I made sure I copied the output, then went to Gitea's web UI. From there I went to **Settings** > **SSH/GPG Keys** > **Add Key**.

Then verified the SSH connection works:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
ssh -T git@gitea
{{< /codeblock>}}

{{< codeblock lang="Expected Output" copy=false >}}
Hi there, git! You've successfully authenticated with the key named ansible-ctrl, but Gitea does not provide shell access.
{{< /codeblock >}}

The 'does not provide shell access' message is normal since Git hosting platforms don't offer interactive shells.

---

## <span class="section-num">07</span> Creating the Repository

In the Gitea web UI, I clicked the + button in the top right and selected **New Repository**:

{{< codeblock lang="REPOSITORY SETTINGS" copy=false >}}
Repository Name:    network-automation-lab
Visibility:         Private
Description:        Ansible network automation
Initialize Repo:    (unchecked)
.gitignore:         None
License:            None
README:             None
Default Branch:     main
{{< /codeblock >}}

---

Back on `ansible-ctrl` I added the Gitea repo as a remote and pushed the existing history.

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
cd ~/network-automation-lab
git remote add origin git@gitea:nesto/network-automation-lab.git
git push -u origin main
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
: Added the Gitea repo as the `origin` remote.

Line 3:
: Pushed the `main` branch.
{{< /line-explain >}}

I confirmed the push by refreshing the Gitea web UI. The repository should show all the files committed.

---

## <span class="section-num">08</span> Branch Protection

I configured branch protection on `main` to enforce a pull request workflow. Even though I'm the only user right now, this builds the habit of never pushing directly to `main`.

In the Gitea web UI, I went to **Repository Settings** > **Branches** > **Add Branch Protection Rule** and configured it this way:

{{< codeblock lang="REPOSITORY SETTINGS" copy=false >}}
Branch Name Pattern:              main
Enable Branch Protection:          Checked
Disable Push:                      Checked
Enable Push Whitelist:             Unchecked
Require Approvals:                 Checked
Required Approvals:                1
Block Merge on Rejected Reviews:   Checked
Block Merge on Outdated Branch:    Checked
Block Merge on Official Review:    Unchecked
Enable Status Checks:              Unchecked (for now)
{{< /codeblock >}}

{{< lab-callout type="info" >}}
**Disable Push** prevents anyone (including the admin) from pushing directly to `main`. All changes must come through a pull request. This guarantees that every change to `main` is deliberate, reviewed, and traceable.

**Block Merge on Outdated Branch** means that if `main` has changed sign the PR branch was created, the PR branch much be rebased or merged with the latest `main` before the PR can be merged. This prevents situations where a PR was valid against an old version of `main` but conflicts with recent changes.
{{< /lab-callout >}}

---

The daily workflow for making changes will look like this:

1. Create a feature branch from main

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
git checkout -b feat/add-ntp-role
{{< /codeblock >}}

2. Make changes, stage, commit

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
git add -A
git commit -m "feat: add NTP configuration role"
{{< /codeblock >}}

3. Push the feature branch to Gitea

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
git push -u origin feat/add-ntp-role
{{< /codeblock >}}

4. Open a Pull Request in the Gitea UI
5. Review the diff, approve, merge
6. Delete the feature branch after merge

7. Return to main locally

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
git checkout main
git pull origin main
{{< /codeblock >}}

- Created a new branch and switched to it.
- Pushed the feature branch to Gitea (the PR workflow handles the merge).
- After the PR is merged, I switch back to `main` locally and pull the latest changes.

---

## <span class="section-num">09</span> Verification

I ran through a final checklist to confirm everything is working.

---

Gitea containers running:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
docker compose -f /opt/gitea/docker-compose.yml ps
{{< /codeblock >}}

Both `gitea` and `gitea-db` should show status of `Up`.

---

SSH connectivity from ansible-ctrl:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
ssh -T git@gitea
{{< /codeblock >}}

---

I verified branch protection is active by attempting a direct push to `main`:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
echo "test" >> README.md
git add README.md
git commit -m "test: verify branch protection"
git push origin main
{{< /codeblock >}}

---

Now I have a self-hosted Gitea instance running on a dedicated VM with PostgresSQL backend, SSH key authentication configured between `ansible-ctrl` and Gitea.