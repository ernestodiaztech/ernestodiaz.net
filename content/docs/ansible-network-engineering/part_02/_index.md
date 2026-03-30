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

