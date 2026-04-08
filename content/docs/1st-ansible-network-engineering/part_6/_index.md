---
draft: false
title: '6 - Containerlab'
description: "Part 6 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 6
---

{{< badge "Ansible" >}}
{{< badge content="Containerlab" color="yellow" >}}

This is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. Each part will build upon the last. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Containerlab

Containerlab spins up actual vendor network operating systems as containers on my Ubuntu VM. I get a full enterprise-style topology with Cisco routers, NX-OS switches, and a Palo Alto firewall running in minutes, and I can destroy and rebuild it just as fast.

---

#### What Containerlab Is

Traditional network labs had three options: physical hardware , GNS3/EVE-NG, or vendor-specific simulators. Containerlab is a different approach entirely.

Containerlab is an open-source tool that uses Docker containers to run network operating systems as lightweight, fast-starting instances. Instead of emulating hardware at the CPU level like GNS3, Containerlab runs the actual vendor NOS binaries directly as containers. The result:

- **Fast startup** - a full 7-device topology can be up in under 5 minutes
- **Reproducible** - the topology is defined in a single YAML file that lives in Git
- **Destroyable** - `containerlab destroy` tears everything down in seconds, freeing all resources
- **Multi-vendor** - Cisco IOS-XE, NX-OS, Juniper, Palo Alto, Arista all on the same host
- **Ansible-ready** - devices get management IPs automatically, ready for Ansible to connect to
- **Resource-efficient** - with 16GB+ RAM on the Ubuntu VM, I can run enterprise-scale topologies

---

## Installing Docker on Ubuntu 22.04

Containerlab requires Docker. I install Docker Engine on my Ubuntu VM.

---

{{% steps %}}

#### Removing Old Docker Versions

Remove any old Docker installations that might conflict

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

---

#### Installing Docker Engine

Step 1: Install prerequisites

```bash
sudo apt update
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

Step 2: Add Docker's official GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Step 3: Add the Docker repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Step 4: Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

- `docker-ce` - the Docker Engine (the daemon that runs containers)
- `docker-ce-cli` - the `docker` command line tool
- `containerd.io` - the container runtime that Docker uses under the hood
- `docker-compose-plugin` - adds `docker compose` support (useful but not required for Containerlab)

---

#### Adding My User to the Docker Group

By default, only `root` can run Docker commands. I add my user to the `docker` group so I can run Docker (and Containerlab) without `sudo`:

```bash
sudo usermod -aG docker $USER
```

This change takes effect on the next login. I either log out and back in, or use:

```bash
newgrp docker
```

---

#### Verifying Docker

Check Docker is running

```bash
sudo systemctl status docker
```

Run the hello-world container to confirm everything works

```bash
docker run hello-world
```

Check Docker version

```bash
docker --version
# Docker version 26.x.x, build ...
```

---

#### Configuring Docker for Large Network Images

Network OS containers are large and require more resources than typical application containers. I configure Docker to handle this:

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "storage-driver": "overlay2",
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 65536,
            "Soft": 65536
        }
    }
}
```

- `log-driver` and `log-opts` - limits log file sizes so network OS containers don't fill up disk with logs
- `default-ulimits` - increases the open file descriptor limit, which some NOS containers require
- `storage-driver: overlay2` - the recommended storage driver for Ubuntu 22.04

Apply the changes:
```bash
sudo systemctl restart docker
sudo systemctl enable docker
```

{{% /steps %}}

---

## Installing Containerlab

With Docker running, Containerlab installs in one command:

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

This script detects my OS, downloads the correct package, and installs it. Verify:

```bash
containerlab version
```

Expected output:
```
                           _                   _       _
                 _        (_)                 | |     | |
 ____ ___  ____ | |_  ____ _ ____  ____  ____| | ____| | _
/ ___) _ \|  _ \|  _)/ _  | |  _ \/ _  )/ ___) |/ _  | || \
( (__| |_|| | | | |_( ( | | | | | ( (/ /| |   | ( ( | | |_) )
\____)___/|_| |_|\___)_||_|_|_| |_|\____)_|   |_|\_||_|____/

    version: 0.56.x
    commit: ...
    date: ...
    source: https://github.com/srl-labs/containerlab
```

---

## Understanding the Topology File

Everything about my lab (which devices exist, what images they use, how they connect) lives in a single YAML file called a topology file. This file goes in my project directory and is committed to Git just like any other configuration file.

---

#### Topology File Structure

```yaml {linenos=table}
name: <lab-name>

topology:
  defaults:
    ...

  nodes:
    <node-name>:
      kind: <node-kind>
      image: <docker-image>
      startup-config: <file>
      mgmt-ipv4: <ip>

  links:
    - endpoints:
        - "<node1>:<interface>"
        - "<node2>:<interface>"
```

- **Line 1** - Name of the lab.
- **Line 4** - Settings applied to all nodes unless overridden.
- **Line 7** - Dictionary of all devices in the topology.
- **Line 9** - Platform type.
- **Line 10** - Docker image to use for this node.
- **Line 11** - Optional: Push this config on first boot.
- **Line 12** - Management IP address on the mgmt network.
- **Line 14** - Lost of connections between nodes.

#### Node Kinds for My Platforms

| Platform | Containerlab `kind` |
|---|---|
| Cisco CSR1000v (IOS-XE) | `vr-cisco_csr1000v` |
| Cisco Nexus 9000v (NX-OS) | `vr-cisco_nxosv` |
| Palo Alto PAN-OS | `vr-pan_panos` |
| Linux host | `linux` |

---

## The Lab Topology Design

Before writing the topology file, I draw out what I'm building.

```
                        INTERNET / WAN CLOUD
                               │
                    ┌──────────┴──────────┐
                    │                     │
               ┌────┴────┐          ┌─────┴───┐
               │  WAN-R1 │          │  WAN-R2 │
               │CSR1000v │          │CSR1000v │
               │172.16.0.11│        │172.16.0.12│
               └────┬────┘          └─────┬───┘
                    │                     │
                    └──────────┬──────────┘
                               │
                        ┌──────┴──────┐
                        │    FW-01    │
                        │  PAN-OS     │
                        │172.16.0.10  │
                        └──────┬──────┘
                               │
               ┌───────────────┴───────────────┐
               │                               │
        ┌──────┴──────┐                 ┌──────┴──────┐
        │  SPINE-01   │                 │  SPINE-02   │
        │  NX-OS 9kv  │─────────────────│  NX-OS 9kv  │
        │172.16.0.21  │                 │172.16.0.22  │
        └──────┬──────┘                 └──────┬──────┘
               │    ╲                 ╱        │
               │     ╲               ╱         │
        ┌──────┴──────┐╲           ╱┌──────────┴──────┐
        │   LEAF-01   │ ╲─────────╱ │    LEAF-02      │
        │  Cat9Kv     │             │   Cat9Kv        │
        │172.16.0.23  │             │  172.16.0.24    │
        └──────┬──────┘             └────────┬────────┘
               │                             │
        ┌──────┴──────┐               ┌──────┴──────┐
        │   HOST-01   │               │   HOST-02   │
        │    Linux    │               │    Linux    │
        │172.16.0.31  │               │172.16.0.32  │
        └─────────────┘               └─────────────┘

Management Network: 172.16.0.0/24 (out-of-band, all devices reachable from Ubuntu VM)
```

#### Device Summary

| Device | Role | Platform | Mgmt IP |
|---|---|---|---|
| `wan-r1` | WAN Router 1 | Cisco CSR1000v (IOS-XE) | 172.16.0.11 |
| `wan-r2` | WAN Router 2 | Cisco CSR1000v (IOS-XE) | 172.16.0.12 |
| `fw-01` | WAN Edge Firewall | Palo Alto PAN-OS | 172.16.0.10 |
| `spine-01` | DC Spine Switch 1 | Cisco NX-OS N9Kv | 172.16.0.21 |
| `spine-02` | DC Spine Switch 2 | Cisco NX-OS N9Kv | 172.16.0.22 |
| `leaf-01` | DC Leaf Switch 1 | Cisco Cat9Kv (IOS-XE) | 172.16.0.23 |
| `leaf-02` | DC Leaf Switch 2 | Cisco Cat9Kv (IOS-XE) | 172.16.0.24 |
| `host-01` | Linux Server/Client 1 | Alpine Linux | 172.16.0.31 |
| `host-02` | Linux Server/Client 2 | Alpine Linux | 172.16.0.32 |

**Total: 9 nodes** (7 network devices + 2 Linux hosts). With 16GB+ RAM this runs comfortably.

---

## Writing the Topology File

I create the topology file in the project directory:

```bash
mkdir -p ~/projects/ansible-network/containerlab
nano ~/projects/ansible-network/containerlab/enterprise-lab.yml
```

```yaml
---
name: enterprise-lab

mgmt:
  network: mgmt-net
  ipv4-subnet: 172.16.0.0/24

topology:

  defaults:
    env:
      USERNAME: ansible
      PASSWORD: ansible123

  nodes:

    # =========================================================
    # WAN ROUTERS — Cisco IOS-XE
    # =========================================================
    wan-r1:
      kind: vr-cisco_csr1000v
      image: vrnetlab/cisco_csr1000v:latest
      mgmt-ipv4: 172.16.0.11
      startup-config: configs/wan-r1-base.cfg

    wan-r2:
      kind: vr-cisco_csr1000v
      image: vrnetlab/cisco_csr1000v:latest
      mgmt-ipv4: 172.16.0.12
      startup-config: configs/wan-r2-base.cfg

    # =========================================================
    # FIREWALL — Palo Alto PAN-OS
    # =========================================================
    fw-01:
      kind: vr-pan_panos
      image: vrnetlab/pan_panos:latest
      mgmt-ipv4: 172.16.0.10
      startup-config: configs/fw-01-base.cfg

    # =========================================================
    # SPINE SWITCHES — Cisco NX-OS
    # =========================================================
    spine-01:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest
      mgmt-ipv4: 172.16.0.21
      startup-config: configs/spine-01-base.cfg

    spine-02:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest
      mgmt-ipv4: 172.16.0.22
      startup-config: configs/spine-02-base.cfg

    # =========================================================
    # LEAF SWITCHES — Cisco NX-OS
    # =========================================================
    leaf-01:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest
      mgmt-ipv4: 172.16.0.23
      startup-config: configs/leaf-01-base.cfg

    leaf-02:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest
      mgmt-ipv4: 172.16.0.24
      startup-config: configs/leaf-02-base.cfg

    # =========================================================
    # LINUX HOSTS — Alpine Linux
    # =========================================================
    host-01:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.0.31

    host-02:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.0.32

  links:
    # =========================================================
    # WAN ROUTERS → FIREWALL
    # WAN-R1 and WAN-R2 connect to FW-01's outside interfaces
    # =========================================================
    - endpoints: ["wan-r1:eth1", "fw-01:eth1"]    # WAN-R1 to FW outside-1
    - endpoints: ["wan-r2:eth2", "fw-01:eth2"]    # WAN-R2 to FW outside-2

    # WAN-R1 to WAN-R2 direct link (for BGP peering / WAN redundancy)
    - endpoints: ["wan-r1:eth2", "wan-r2:eth1"]

    # =========================================================
    # FIREWALL → SPINE LAYER
    # FW-01 inside interfaces connect to both spines (redundant)
    # =========================================================
    - endpoints: ["fw-01:eth3", "spine-01:eth1"]  # FW inside-1 to SPINE-01
    - endpoints: ["fw-01:eth4", "spine-02:eth1"]  # FW inside-2 to SPINE-02

    # =========================================================
    # SPINE → SPINE (inter-spine link for routing)
    # =========================================================
    - endpoints: ["spine-01:eth2", "spine-02:eth2"]

    # =========================================================
    # SPINE → LEAF (full mesh — each leaf connects to both spines)
    # =========================================================
    - endpoints: ["spine-01:eth3", "leaf-01:eth1"]   # SPINE-01 to LEAF-01
    - endpoints: ["spine-01:eth4", "leaf-02:eth1"]   # SPINE-01 to LEAF-02
    - endpoints: ["spine-02:eth3", "leaf-01:eth2"]   # SPINE-02 to LEAF-01
    - endpoints: ["spine-02:eth4", "leaf-02:eth2"]   # SPINE-02 to LEAF-02

    # =========================================================
    # LEAF → HOSTS (servers behind the leaf switches)
    # =========================================================
    - endpoints: ["leaf-01:eth3", "host-01:eth1"]    # LEAF-01 to HOST-01
    - endpoints: ["leaf-02:eth3", "host-02:eth1"]    # LEAF-02 to HOST-02
```

#### Understanding the `links:` Section

```yaml
- endpoints: ["wan-r1:eth1", "fw-01:eth1"]
```

This creates a virtual cable between `wan-r1`'s `eth1` interface and `fw-01`'s `eth1` interface. Containerlab creates a Linux veth (virtual ethernet) pair and attaches one end to each container.

**Interface naming convention:**
- `eth0` - always the management interface (handled automatically by Containerlab, never wired in `links:`)
- `eth1`, `eth2`, etc. — data plane interfaces, wired in the `links:` section

#### Creating Base Startup Configs Directory

The `startup-config:` field in the topology file points to a config file that gets pushed to the device on first boot. I create the directory and minimal base configs to ensure SSH and an Ansible user are configured:

```bash
mkdir -p ~/projects/ansible-network/containerlab/configs
```

**`configs/wan-r1-base.cfg`** (and repeat for `wan-r2-base.cfg` with `wan-r2` as the hostname):

```
hostname wan-r1
!
ip domain-name lab.local
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
username ansible privilege 15 secret ansible123
!
line vty 0 4
 login local
 transport input ssh
!
ip ssh timeout 60
ip ssh authentication-retries 3
!
end
```

---

**`configs/leaf-01-base.cfg`** (repeat for `leaf-02-base.cfg` with `leaf-02` as the hostname).  

```
hostname leaf-01
!
ip domain-name lab.local
!
crypto key generate rsa modulus 2048
ip ssh version 2
!
username ansible privilege 15 secret ansible123
!
line vty 0 4
 login local
 transport input ssh
!
end
```

---

**`configs/spine-01-base.cfg`** (repeat for `spine-02-base.cfg`).  

```
hostname spine-01
!
feature ssh
feature nxapi
!
username ansible password ansible123 role network-admin
!
ssh key rsa 2048
!
```

---

## Starting, Stopping, and Destroying the Lab

All Containerlab commands are run from the directory containing the topology file, or I specify the path with `-t`.

---

{{% steps %}}

#### Deploying the Lab

First, move to the directory where the Containerlab YAML file is:

```bash
cd ~/projects/ansible-network/containerlab
```

Deploy the lab

```bash
sudo containerlab deploy -t enterprise-lab.yml
```

The first deploy takes longer because Docker pulls images that aren't cached yet. Subsequent deploys of the same topology are much faster.

Containerlab output during deploy:
```
INFO[0000] Containerlab v0.56.x started
INFO[0000] Parsing & checking topology file: enterprise-lab.yml
INFO[0000] Creating lab directory: /home/ansible/projects/ansible-network/containerlab/clab-enterprise-lab
INFO[0001] Creating docker network: Name=mgmt-net, IPv4Subnet=172.16.0.0/24
INFO[0002] Creating container: wan-r1
INFO[0003] Creating container: wan-r2
INFO[0004] Creating container: fw-01
INFO[0005] Creating container: spine-01
...
INFO[0120] Adding containerlab host entries to /etc/hosts
+----+------------------+--------------+---------------------+------+---------+
| #  |       Name       | Container ID |        Image        | Kind |  State  |
+----+------------------+--------------+---------------------+------+---------+
|  1 | clab-enterprise-lab-fw-01    | a1b2c3d4 | vrnetlab/paloalto_pa-vm:11.2.3    | paloalto_panos   | running |
|  2 | clab-enterprise-lab-wan-r1   | b2c3d4e5 | vrnetlab/cisco_csr1000v:17.03.08  | cisco_csr1000v   | running |
...
+----+------------------+--------------+---------------------+------+---------+
```

>[!Info]
> Containerlab automatically adds entries to `/etc/hosts` for every node in the topology. After deploying, I can SSH to `clab-enterprise-lab-wan-r1` or `wan-r1` by hostname instead of IP. The hostname format is always `clab-<lab-name>-<node-name>`. This is useful but the management IPs I defined in the topology file are what I'll use in the Ansible inventory.

---

#### Checking Lab Status

Check the status of all running labs

```bash
sudo containerlab inspect --all
```

Check status of a specific lab

```bash
sudo containerlab inspect -t enterprise-lab.yml
```

---

#### Stopping the Lab (Preserves State)

Stop all containers without destroying them (Configurations and state are preserved)

```bash
sudo containerlab deploy -t enterprise-lab.yml --reconfigure
```

**A cleaner approach to stop and start individual containers:**

Stop a specific container

```bash
docker stop clab-enterprise-lab-wan-r1
```

Start it again (preserves config)

```bash
docker start clab-enterprise-lab-wan-r1
```

Stop all lab containers at once

```bash
docker ps --filter "label=containerlab=enterprise-lab" -q | xargs docker stop
```

---

#### Destroying the Lab

Completely destroy the lab, this removes all containers and the management network

```bash
sudo containerlab destroy -t enterprise-lab.yml
```

Destroy and remove all lab files (including the clab-enterprise-lab/ directory)

```bash
sudo containerlab destroy -t enterprise-lab.yml --cleanup
```

>[!Caution]
> `containerlab destroy` removes all containers and their state. Any configuration changes made inside the devices that aren't saved in startup-config files or Git-committed Ansible playbooks are gone permanently. This is by design since the lab should be fully reproducible from the topology file and playbooks.

---

#### Redeploying from Scratch

Destroy the old lab and immediately redeploy fresh

```bash
sudo containerlab destroy -t enterprise-lab.yml --cleanup
sudo containerlab deploy -t enterprise-lab.yml
```

The entire topology comes back to a clean baseline in minutes.

{{% /steps %}}

---

## Accessing Devices via SSH

Once the lab is deployed, I can SSH directly to any device from my Ubuntu VM.

---

#### SSH to Network Devices

SSH using the management IP

```bash
ssh ansible@172.16.0.11
```

SSH using the Containerlab hostname (added to /etc/hosts)

```bash
ssh ansible@clab-enterprise-lab-wan-r1
```

SSH to NX-OS

```bash
ssh ansible@172.16.0.21
```

SSH to PAN-OS

```bash
ssh ansible@172.16.0.10
```

{{< callout type="info" >}}
Every time I destroy and redeploy the lab, the devices get new SSH host keys but the old keys are still in my `~/.ssh/known_hosts` file. SSH will refuse to connect and show:
```
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```
The fix:

Remove the old key for a specific IP

```bash
ssh-keygen -R 172.16.0.11
```

Or remove all keys for the entire management subnet

```bash
for ip in $(seq 10 35); do ssh-keygen -R 172.16.0.$ip; done
```
This is also why `host_key_checking = False` is set in `ansible.cfg` for the lab since Ansible would fail on every fresh deploy otherwise.
{{< /callout >}}

---

#### Verifying Device Connectivity

Before running any Ansible playbooks, I verify basic connectivity to all devices:

```bash
# Ping all management IPs
for ip in 10 11 12 21 22 23 24 31 32; do
    ping -c 1 -W 1 172.16.0.$ip > /dev/null 2>&1 && \
    echo "172.16.0.$ip — UP" || \
    echo "172.16.0.$ip — DOWN"
done
```

Expected output when all devices are ready:
```
172.16.0.10 — UP   (fw-01)
172.16.0.11 — UP   (wan-r1)
172.16.0.12 — UP   (wan-r2)
172.16.0.21 — UP   (spine-01)
172.16.0.22 — UP   (spine-02)
172.16.0.23 — UP   (leaf-01)
172.16.0.24 — UP   (leaf-02)
172.16.0.31 — UP   (host-01)
172.16.0.32 — UP   (host-02)
```

#### Checking Device Boot Status

Watch a container's console output to see boot progress

```bash
docker logs -f clab-enterprise-lab-spine-01
```

Check if SSH is responding (quicker than a full SSH session)

```bash
nc -zv 172.16.0.21 22
# Connection to 172.16.0.21 22 port [tcp/ssh] succeeded!
```

---

## Management IP Addressing

This is important to understand before building the Ansible inventory.

---

#### The Management Network

When I deploy the lab, Containerlab creates a Docker bridge network called `mgmt-net` with the subnet `172.16.0.0/24`. Every container in the topology connects to this bridge via its `eth0` interface.

```
┌─────────────────────────────────────────────────────┐
│              Docker Bridge: mgmt-net                │
│              Subnet: 172.16.0.0/24                  │
│              Gateway: 172.16.0.1                    │
│                                                     │
│  Ubuntu VM ─── 172.16.0.1 (Docker bridge gateway)   │
│                                                     │
│  wan-r1    ─── 172.16.0.11 (eth0 management)        │
│  wan-r2    ─── 172.16.0.12 (eth0 management)        │
│  fw-01     ─── 172.16.0.10 (eth0 management)        │
│  spine-01  ─── 172.16.0.21 (eth0 management)        │
│  spine-02  ─── 172.16.0.22 (eth0 management)        │
│  leaf-01   ─── 172.16.0.23 (eth0 management)        │
│  leaf-02   ─── 172.16.0.24 (eth0 management)        │
│  host-01   ─── 172.16.0.31 (eth0 management)        │
│  host-02   ─── 172.16.0.32 (eth0 management)        │
└─────────────────────────────────────────────────────┘
```

The Ubuntu VM can reach all management IPs directly because the Docker bridge acts as a router on the Ubuntu VM at `172.16.0.1`. This is the out-of-band management network.

---

#### Data Plane vs Management Plane

The `links:` section in the topology file creates data plane connections, the interfaces that carry production traffic between devices. These are the `eth1`, `eth2`, etc. interfaces that I'll configure with IP addresses and routing protocols via Ansible.

```
Management plane (eth0):  172.16.0.0/24  — Ansible connects here
Data plane (eth1+):       I configure these via playbooks (e.g., 10.0.0.0/30 point-to-point links)
```

This separation mirrors real enterprise networks where out-of-band management (OOBM) is kept completely separate from production traffic.

---

## Mapping Containerlab Devices to the Ansible Inventory

This is the bridge between Containerlab and everything that comes next. I create the Ansible inventory that references my Containerlab lab devices.

---

#### Creating the Inventory File

```bash
nano ~/projects/ansible-network/inventory/hosts.yml
```

```yaml
---
# =============================================================
# Ansible Inventory
# Management Network: 172.16.0.0/24
# =============================================================

all:
  children:

    # =========================================================
    # CISCO IOS-XE - WAN Routers + Leaf Switches
    # CSR1000v (wan-r1, wan-r2) and Cat9Kv (leaf-01, leaf-02)
    # =========================================================
    cisco_ios:
      children:
        wan_routers:
          hosts:
            wan-r1:
              ansible_host: 172.16.0.11
            wan-r2:
              ansible_host: 172.16.0.12
        leaf_switches:
          hosts:
            leaf-01:
              ansible_host: 172.16.0.23
            leaf-02:
              ansible_host: 172.16.0.24

    # =========================================================
    # CISCO NX-OS - Spine Switches only
    # =========================================================
    cisco_nxos:
      children:
        spine_switches:
          hosts:
            spine-01:
              ansible_host: 172.16.0.21
            spine-02:
              ansible_host: 172.16.0.22

    # =========================================================
    # PALO ALTO - Firewall
    # =========================================================
    paloalto:
      hosts:
        fw-01:
          ansible_host: 172.16.0.10

    # =========================================================
    # LINUX HOSTS
    # =========================================================
    linux_hosts:
      hosts:
        host-01:
          ansible_host: 172.16.0.31
        host-02:
          ansible_host: 172.16.0.32

    # =========================================================
    # LOGICAL GROUPINGS
    # =========================================================
    wan:
      hosts:
        wan-r1:
        wan-r2:

    datacenter:
      children:
        spine_switches:
        leaf_switches:

    network_devices:
      children:
        cisco_ios:
        cisco_nxos:
        paloalto:
```

---

#### Creating Group Variables

Group variables define connection settings that apply to all devices in a group. This is where `ansible_network_os`, `ansible_user`, and `ansible_password` live.

```bash
# Create group_vars files for each platform
```

**`inventory/group_vars/all.yml`** - settings for every device:

```yaml
---
# Global settings applied to all devices
ansible_user: ansible
ansible_password: ansible123
```

**`inventory/group_vars/cisco_ios.yml`**:

```yaml
---
ansible_network_os: cisco.ios.ios
ansible_connection: network_cli
ansible_become: true
ansible_become_method: enable
ansible_become_password: ansible123
```

**`inventory/group_vars/cisco_nxos.yml`**:

```yaml
---
ansible_network_os: cisco.nxos.nxos
ansible_connection: network_cli
```

**`inventory/group_vars/paloalto.yml`**:

```yaml
---
ansible_network_os: paloaltonetworks.panos.panos
ansible_connection: ansible.netcommon.httpapi
ansible_httpapi_use_ssl: true
ansible_httpapi_validate_certs: false
```

**`inventory/group_vars/linux_hosts.yml`**:

```yaml
---
ansible_connection: ssh
ansible_python_interpreter: /usr/bin/python3
```

---

#### Verifying the Inventory

```bash
cd ~/projects/ansible-network
```

Show the full inventory structure as JSON

```bash
ansible-inventory -i inventory/hosts.yml --list
```

Show the inventory as a tree graph

```bash
ansible-inventory -i inventory/hosts.yml --graph
```

Expected `--graph` output:
```
@all:
  |--@cisco_ios:
  |  |--wan-r1
  |  |--wan-r2
  |--@cisco_nxos:
  |  |--@spine_switches:
  |  |  |--spine-01
  |  |  |--spine-02
  |  |--@leaf_switches:
  |  |  |--leaf-01
  |  |  |--leaf-02
  |--@paloalto:
  |  |--fw-01
  |--@linux_hosts:
  |  |--host-01
  |  |--host-02
  |--@wan:
  |  |--wan-r1
  |  |--wan-r2
  |--@datacenter:
  |  |--spine-01
  |  |--spine-02
  |  |--leaf-01
  |  |--leaf-02
  |--@network_devices:
  |  |--wan-r1
  |  |--wan-r2
  |  |--fw-01
  |  |--spine-01
  |  |--spine-02
  |  |--leaf-01
  |  |--leaf-02
  |--@ungrouped:
```

---

#### Testing Ansible Connectivity to the Lab

With the inventory in place and the lab running, I run a quick connectivity test:

1. Test connection to all IOS-XE devices

```bash
ansible cisco_ios -m ansible.netcommon.net_ping -i inventory/hosts.yml
```

2. Test connection to NX-OS devices

```bash
ansible cisco_nxos -m ansible.netcommon.net_ping -i inventory/hosts.yml
```

3. Test connection to Linux hosts

```bash
ansible linux_hosts -m ansible.builtin.ping -i inventory/hosts.yml
```

A successful IOS-XE response looks like:
```yaml
wan-r1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
wan-r2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

>[!Tip]
> I save this connectivity test as a simple shell script `~/projects/ansible-network/scripts/test-connectivity.sh`:
> ```bash
> #!/bin/bash
> echo "Testing IOS-XE..."
> ansible cisco_ios -m ansible.netcommon.net_ping
> echo "Testing NX-OS..."
> ansible cisco_nxos -m ansible.netcommon.net_ping
> echo "Testing Linux hosts..."
> ansible linux_hosts -m ansible.builtin.ping
> ```
> I run this after every lab deploy to confirm everything is reachable before starting automation work. It saves me from debugging a playbook failure that was actually just a device that hadn't finished booting.

---

## Containerlab Day-to-Day Workflow

This is the routine I follow at the start and end of every lab session:

---

{{% steps %}}

#### Starting a Lab Session

1. Navigate to the project

```bash
cd ~/projects/ansible-network
```

2. Activate the virtual environment

```bash
source ~/venvs/ansible-network/bin/activate
```

3. Start a tmux session (from Part 1)

```bash
tmux new -s ansible-lab
```

4. Deploy the lab (if not already running)

```bash
cd containerlab
sudo containerlab deploy -t enterprise-lab.yml
```

5. Wait for devices to finish booting (~5 minutes for NX-OS)

```bash
docker logs -f clab-enterprise-lab-spine-01
```

6. Test connectivity

```bash
cd ~/projects/ansible-network
./scripts/test-connectivity.sh
```

7. Start working

---

#### Ending a Lab Session

Option A: Leave it running (devices preserve state, uses RAM)

Just detach from tmux: Ctrl+B, d

---

Option B: Stop containers (preserves state, frees RAM)

```bash
docker ps --filter "label=containerlab" -q | xargs docker stop
```

---

Option C: Destroy the lab completely (frees all resources)

```bash
cd ~/projects/ansible-network/containerlab
sudo containerlab destroy -t enterprise-lab.yml
```

{{% /steps %}}

---

## Committing the Lab Files to Git

Everything created in this part goes into version control:

```bash
cd ~/projects/ansible-network
```


Stage all new files

```bash
git add containerlab/
git add inventory/
git add scripts/
```

Check what's being committed

```bash
git status
```

Commit

```bash
git commit -m "feat(lab): add Containerlab enterprise topology and Ansible inventory

- Add 9-node enterprise topology: 2x IOS-XE WAN routers, 1x PAN-OS firewall,
  2x NX-OS spine switches, 2x NX-OS leaf switches, 2x Linux hosts
- Add management network 172.16.0.0/24 with static IPs per device
- Add Ansible inventory with platform groups and logical groupings
- Add group_vars for cisco_ios, cisco_nxos, paloalto, linux_hosts
- Add connectivity test script"
```

---

The lab is running. I have a full enterprise-style topology, WAN routers, a firewall, spine/leaf switches, and Linux hosts.

