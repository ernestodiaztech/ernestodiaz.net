---
draft: false
title: '6 - Containerlab'
weight: 6
---

This project is me documenting my journey of learning Ansible that is focused on network engineering. It's not a "how-to guide" per-say, more of a diary. A lot of information on here is so I can come back to and reference later. I also learn best when teaching someone, and this is kind of me teaching.


## Part 6: Containerlab — Your Network Lab Environment

> *Every playbook from Part 7 onward runs against real network devices — not simulated ones, not mocked ones. Containerlab spins up actual vendor network operating systems as containers on my Ubuntu VM. I get a full enterprise-style topology with Cisco routers, NX-OS switches, and a Palo Alto firewall running in minutes, and I can destroy and rebuild it just as fast. This is the lab environment that makes everything else in this guide real.*

---

## 6.1 — What Containerlab Is and Why It's Ideal for Network Automation Labs

Traditional network labs had three options: physical hardware (expensive, inflexible), GNS3/EVE-NG (heavy, complex setup), or vendor-specific simulators (limited to one platform). Containerlab is a different approach entirely.

Containerlab is an open-source tool that uses Docker containers to run network operating systems as lightweight, fast-starting instances. Instead of emulating hardware at the CPU level like GNS3, Containerlab runs the actual vendor NOS binaries directly as containers. The result:

- **Fast startup** — a full 7-device topology can be up in under 5 minutes
- **Reproducible** — the topology is defined in a single YAML file that lives in Git
- **Destroyable** — `containerlab destroy` tears everything down in seconds, freeing all resources
- **Multi-vendor** — Cisco IOS-XE, NX-OS, Juniper, Palo Alto, Arista all on the same host
- **Ansible-ready** — devices get management IPs automatically, ready for Ansible to connect to
- **Resource-efficient** — with 16GB+ RAM on the Ubuntu VM, I can run enterprise-scale topologies

### ### ℹ️ Info

> Containerlab itself doesn't ship with vendor images. I need to provide the NOS container images separately. Most vendor images for Containerlab are built using **vrnetlab** — a tool that wraps VM-based router images (like CSR1000v `.qcow2` files) into Docker containers. The resulting images work with Containerlab's `vrnetlab` node kind. I already have my images — Part 6 covers how to use them, not how to acquire them.

### ### 🏢 Real-World Scenario

> Before Containerlab, setting up a multi-vendor test environment for a network automation project meant either booking time on physical lab gear (shared with other teams, always in the wrong configuration), or running EVE-NG on a beefy server (which took 30 minutes to start a 10-device topology). I've seen teams skip testing entirely because the lab was too painful to set up. Containerlab removes that excuse — the lab is a YAML file and a single command. If a test breaks the topology, I destroy it and redeploy in minutes.

---

## 6.2 — Installing Docker on Ubuntu 22.04

Containerlab requires Docker. I install Docker Engine (not Docker Desktop — that's for macOS/Windows) on my Ubuntu VM.

### Removing Old Docker Versions

```bash
# Remove any old Docker installations that might conflict
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### Installing Docker Engine

```bash
# Step 1: Install prerequisites
sudo apt update
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Step 2: Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Step 3: Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 4: Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**What each package is:**
- `docker-ce` — the Docker Engine (the daemon that runs containers)
- `docker-ce-cli` — the `docker` command line tool
- `containerd.io` — the container runtime that Docker uses under the hood
- `docker-compose-plugin` — adds `docker compose` support (useful but not required for Containerlab)

### Adding My User to the Docker Group

By default, only `root` can run Docker commands. I add my user to the `docker` group so I can run Docker (and Containerlab) without `sudo`:

```bash
sudo usermod -aG docker $USER
```

This change takes effect on the next login. I either log out and back in, or use:

```bash
newgrp docker
```

### Verifying Docker

```bash
# Check Docker is running
sudo systemctl status docker

# Run the hello-world container to confirm everything works
docker run hello-world

# Check Docker version
docker --version
# Docker version 26.x.x, build ...
```

### ### ⚠️ Warning

> Adding my user to the `docker` group gives me the ability to run Docker without `sudo` — but this is effectively equivalent to root access. Any container I run can mount the host filesystem, modify system files, and escalate privileges. On a shared server, adding users to the `docker` group should be done carefully. On a personal lab VM that only I access, this is the standard and expected setup.

### Configuring Docker for Large Network Images

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

- `log-driver` and `log-opts` — limits log file sizes so network OS containers don't fill up disk with logs
- `default-ulimits` — increases the open file descriptor limit, which some NOS containers require
- `storage-driver: overlay2` — the recommended storage driver for Ubuntu 22.04

Apply the changes:
```bash
sudo systemctl restart docker
sudo systemctl enable docker    # Start Docker automatically on boot
```

---

## 6.3 — Installing Containerlab

With Docker running, Containerlab installs in one command:

```bash
# Official one-line installer
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

### ### 💡 Tip

> Containerlab releases updates frequently. I can upgrade to the latest version at any time by re-running the installer:
> ```bash
> bash -c "$(curl -sL https://get.containerlab.dev)"
> ```
> I check the Containerlab changelog at `containerlab.dev/rn/` before upgrading — topology file syntax occasionally changes between versions.

---

## 6.4 — Understanding the Topology File

Everything about my lab — which devices exist, what images they use, how they connect — lives in a single YAML file called a topology file. This file goes in my project directory and is committed to Git just like any other configuration file.

### Topology File Structure

```yaml
name: <lab-name>              # Name of the lab — used in container names

topology:
  defaults:                   # Settings applied to all nodes unless overridden
    ...

  nodes:                      # Dictionary of all devices in the topology
    <node-name>:
      kind: <node-kind>       # Platform type (vr-csr, vr-nxos, vr-pan, etc.)
      image: <docker-image>   # Docker image to use for this node
      startup-config: <file>  # Optional: push this config on first boot
      mgmt-ipv4: <ip>         # Management IP address on the mgmt network

  links:                      # List of connections between nodes
    - endpoints:
        - "<node1>:<interface>"
        - "<node2>:<interface>"
```

### Node Kinds for My Platforms

| Platform | Containerlab `kind` |
|---|---|
| Cisco CSR1000v (IOS-XE) | `vr-cisco_csr1000v` |
| Cisco Nexus 9000v (NX-OS) | `vr-cisco_nxosv` |
| Palo Alto PAN-OS | `vr-pan_panos` |
| Linux host | `linux` |

### ### ℹ️ Info

> The `vr-` prefix stands for **vrnetlab** — the tool used to wrap VM-based router images into Docker containers. When I see `kind: vr-cisco_csr1000v`, Containerlab knows to use the vrnetlab runtime, which handles booting the VM inside the container and bridging the network interfaces. The management interface is always handled automatically by Containerlab — I don't wire it up in the `links:` section.

---

## 6.5 — The Lab Topology Design

Before writing the topology file, I draw out what I'm building. This is the enterprise-style topology that all automation in this guide is written against.

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
                        │  PAN-OS    │
                        │172.16.0.10 │
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
        │  NX-OS 9kv  │             │   NX-OS 9kv     │
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

### Device Summary

| Device | Role | Platform | Mgmt IP |
|---|---|---|---|
| `wan-r1` | WAN Router 1 | Cisco CSR1000v (IOS-XE) | 172.16.0.11 |
| `wan-r2` | WAN Router 2 | Cisco CSR1000v (IOS-XE) | 172.16.0.12 |
| `fw-01` | WAN Edge Firewall | Palo Alto PAN-OS | 172.16.0.10 |
| `spine-01` | DC Spine Switch 1 | Cisco NX-OS 9kv | 172.16.0.21 |
| `spine-02` | DC Spine Switch 2 | Cisco NX-OS 9kv | 172.16.0.22 |
| `leaf-01` | DC Leaf Switch 1 | Cisco NX-OS 9kv | 172.16.0.23 |
| `leaf-02` | DC Leaf Switch 2 | Cisco NX-OS 9kv | 172.16.0.24 |
| `host-01` | Linux Server/Client 1 | Alpine Linux | 172.16.0.31 |
| `host-02` | Linux Server/Client 2 | Alpine Linux | 172.16.0.32 |

**Total: 9 nodes** (7 network devices + 2 Linux hosts). With 16GB+ RAM this runs comfortably.

---

## 6.6 — Writing the Topology File

I create the topology file in the project directory:

```bash
mkdir -p ~/projects/ansible-network/containerlab
nano ~/projects/ansible-network/containerlab/enterprise-lab.yml
```

```yaml
---
name: enterprise-lab

mgmt:
  network: mgmt-net           # Name of the Docker management network
  ipv4-subnet: 172.16.0.0/24  # Management subnet for all devices

topology:

  defaults:
    # Default credentials for all nodes (override per-node if needed)
    env:
      USERNAME: ansible
      PASSWORD: ansible123

  nodes:

    # =========================================================
    # WAN ROUTERS — Cisco IOS-XE (CSR1000v)
    # =========================================================
    wan-r1:
      kind: vr-cisco_csr1000v
      image: vrnetlab/cisco_csr1000v:latest   # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.11
      startup-config: configs/wan-r1-base.cfg

    wan-r2:
      kind: vr-cisco_csr1000v
      image: vrnetlab/cisco_csr1000v:latest   # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.12
      startup-config: configs/wan-r2-base.cfg

    # =========================================================
    # FIREWALL — Palo Alto PAN-OS
    # =========================================================
    fw-01:
      kind: vr-pan_panos
      image: vrnetlab/pan_panos:latest        # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.10
      startup-config: configs/fw-01-base.cfg

    # =========================================================
    # SPINE SWITCHES — Cisco NX-OS (Nexus 9000v)
    # =========================================================
    spine-01:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest      # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.21
      startup-config: configs/spine-01-base.cfg

    spine-02:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest      # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.22
      startup-config: configs/spine-02-base.cfg

    # =========================================================
    # LEAF SWITCHES — Cisco NX-OS (Nexus 9000v)
    # =========================================================
    leaf-01:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest      # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.23
      startup-config: configs/leaf-01-base.cfg

    leaf-02:
      kind: vr-cisco_nxosv
      image: vrnetlab/cisco_nxosv:latest      # ← Replace with your actual image tag
      mgmt-ipv4: 172.16.0.24
      startup-config: configs/leaf-02-base.cfg

    # =========================================================
    # LINUX HOSTS — Alpine Linux (lightweight)
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

### Understanding the `links:` Section

```yaml
- endpoints: ["wan-r1:eth1", "fw-01:eth1"]
```

This creates a virtual cable between `wan-r1`'s `eth1` interface and `fw-01`'s `eth1` interface. Containerlab creates a Linux veth (virtual ethernet) pair and attaches one end to each container.

**Interface naming convention:**
- `eth0` — always the management interface (handled automatically by Containerlab, never wired in `links:`)
- `eth1`, `eth2`, etc. — data plane interfaces, wired in the `links:` section

### ### ⚠️ Warning

> Never wire `eth0` in the `links:` section. `eth0` is reserved by Containerlab as the management interface and is automatically connected to the management network (`172.16.0.0/24`). Wiring it manually in `links:` will cause unpredictable behavior and may break management connectivity to the device.

### Creating Base Startup Configs Directory

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

**`configs/spine-01-base.cfg`** (repeat for spine-02, leaf-01, leaf-02 with correct hostnames):

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

### ### 💡 Tip

> Startup configs are optional — if I don't provide one, the device boots to factory defaults. For automation practice, I want SSH and an `ansible` user pre-configured so Ansible can connect immediately without manual setup. As I progress through later parts, I'll have Ansible manage the full configuration — the startup config is just the minimal bootstrap to get SSH working.

---

## 6.7 — Starting, Stopping, and Destroying the Lab

All Containerlab commands are run from the directory containing the topology file, or I specify the path with `-t`.

### Deploying the Lab

```bash
cd ~/projects/ansible-network/containerlab

# Deploy the lab
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
|  1 | clab-enterprise-lab-fw-01    | a1b2c3d4 | vrnetlab/pan_panos:latest    | vr-pan_panos        | running |
|  2 | clab-enterprise-lab-wan-r1   | b2c3d4e5 | vrnetlab/cisco_csr1000v:latest | vr-cisco_csr1000v | running |
...
+----+------------------+--------------+---------------------+------+---------+
```

### ### ℹ️ Info

> Containerlab automatically adds entries to `/etc/hosts` for every node in the topology. After deploying, I can SSH to `clab-enterprise-lab-wan-r1` or `wan-r1` by hostname instead of IP. The hostname format is always `clab-<lab-name>-<node-name>`. This is useful but the management IPs I defined in the topology file are what I'll use in the Ansible inventory.

### Checking Lab Status

```bash
# Check the status of all running labs
sudo containerlab inspect --all

# Check status of a specific lab
sudo containerlab inspect -t enterprise-lab.yml
```

### Stopping the Lab (Preserves State)

```bash
# Stop all containers without destroying them
# Configurations and state are preserved
sudo containerlab deploy -t enterprise-lab.yml --reconfigure
```

A cleaner approach — stop and start individual containers:

```bash
# Stop a specific container
docker stop clab-enterprise-lab-wan-r1

# Start it again (preserves config)
docker start clab-enterprise-lab-wan-r1

# Stop all lab containers at once
docker ps --filter "label=containerlab=enterprise-lab" -q | xargs docker stop
```

### Destroying the Lab

```bash
# Completely destroy the lab — removes all containers and the management network
sudo containerlab destroy -t enterprise-lab.yml

# Destroy and remove all lab files (including the clab-enterprise-lab/ directory)
sudo containerlab destroy -t enterprise-lab.yml --cleanup
```

### ### ⚠️ Warning

> `containerlab destroy` removes all containers and their state — any configuration changes made inside the devices that aren't saved in startup-config files or Git-committed Ansible playbooks are gone permanently. This is by design — the lab should be fully reproducible from the topology file and playbooks. If I want to preserve a device configuration, I run an Ansible backup playbook (covered in Part 30) before destroying.

### Redeploying from Scratch

```bash
# Destroy the old lab and immediately redeploy fresh
sudo containerlab destroy -t enterprise-lab.yml --cleanup
sudo containerlab deploy -t enterprise-lab.yml
```

This is my "nuclear reset" when a lab gets into a bad state. The entire topology comes back to a clean baseline in minutes.

---

## 6.8 — Accessing Devices via SSH from the Ubuntu VM

Once the lab is deployed, I can SSH directly to any device from my Ubuntu VM.

### SSH to Network Devices

```bash
# SSH using the management IP
ssh ansible@172.16.0.11        # wan-r1

# SSH using the Containerlab hostname (added to /etc/hosts)
ssh ansible@clab-enterprise-lab-wan-r1

# SSH to NX-OS
ssh ansible@172.16.0.21        # spine-01

# SSH to PAN-OS
ssh ansible@172.16.0.10        # fw-01
```

### ### 🪲 Gotcha — SSH Host Key Conflicts

> Every time I destroy and redeploy the lab, the devices get new SSH host keys — but the old keys are still in my `~/.ssh/known_hosts` file. SSH will refuse to connect and show:
> ```
> WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
> ```
> The fix:
> ```bash
> # Remove the old key for a specific IP
> ssh-keygen -R 172.16.0.11
>
> # Or remove all keys for the entire management subnet
> for ip in $(seq 10 35); do ssh-keygen -R 172.16.0.$ip; done
> ```
> This is also why `host_key_checking = False` is set in `ansible.cfg` for the lab — Ansible would fail on every fresh deploy otherwise.

### Verifying Device Connectivity

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

### ### 💡 Tip

> NX-OS and PAN-OS take significantly longer to boot than IOS-XE. After `containerlab deploy` returns, the management network is up but the NOS may still be booting. I wait 3–5 minutes and run the ping check before trying Ansible connections. For NX-OS specifically, I know it's ready when SSH responds — it can take 4–8 minutes on first boot.

### Checking Device Boot Status

```bash
# Watch a container's console output to see boot progress
docker logs -f clab-enterprise-lab-spine-01

# Check if SSH is responding (quicker than a full SSH session)
nc -zv 172.16.0.21 22
# Connection to 172.16.0.21 22 port [tcp/ssh] succeeded!
```

---

## 6.9 — Understanding Management IP Addressing in Containerlab

This is important to understand before building the Ansible inventory.

### The Management Network

When I deploy the lab, Containerlab creates a Docker bridge network called `mgmt-net` with the subnet `172.16.0.0/24`. Every container in the topology connects to this bridge via its `eth0` interface.

```
┌─────────────────────────────────────────────────────┐
│              Docker Bridge: mgmt-net                 │
│              Subnet: 172.16.0.0/24                   │
│              Gateway: 172.16.0.1                     │
│                                                      │
│  Ubuntu VM ─── 172.16.0.1 (Docker bridge gateway)  │
│                                                      │
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

The Ubuntu VM can reach all management IPs directly — the Docker bridge acts as a router on the Ubuntu VM at `172.16.0.1`. This is the out-of-band management network.

### Data Plane vs Management Plane

The `links:` section in the topology file creates data plane connections — the interfaces that carry production traffic between devices. These are the `eth1`, `eth2`, etc. interfaces that I'll configure with IP addresses and routing protocols via Ansible.

```
Management plane (eth0):  172.16.0.0/24  — Ansible connects here
Data plane (eth1+):       I configure these via playbooks (e.g., 10.0.0.0/30 point-to-point links)
```

This separation mirrors real enterprise networks where out-of-band management (OOBM) is kept completely separate from production traffic.

### ### 🏢 Real-World Scenario

> In enterprise networks, out-of-band management is taken very seriously. Production routers and switches have a dedicated management interface (often `Gi0/0` on IOS or `mgmt0` on NX-OS) connected to a separate management network, sometimes on a dedicated VLAN or even a completely separate physical network. Ansible always connects to devices through this management interface, never through a production data plane interface. Containerlab's management network models this exactly — `eth0` is the management interface, data plane interfaces are `eth1` and above.

---

## 6.10 — Mapping Containerlab Devices to the Ansible Inventory

This is the bridge between Containerlab and everything that comes next. I create the Ansible inventory that references my Containerlab lab devices. This inventory file is used for all playbooks in Parts 7–21.

### Creating the Inventory File

```bash
nano ~/projects/ansible-network/inventory/hosts.yml
```

```yaml
---
# =============================================================
# Ansible Inventory — Enterprise Lab (Containerlab)
# Management Network: 172.16.0.0/24
# =============================================================

all:
  children:

    # =========================================================
    # CISCO IOS-XE — WAN Routers
    # =========================================================
    cisco_ios:
      hosts:
        wan-r1:
          ansible_host: 172.16.0.11
        wan-r2:
          ansible_host: 172.16.0.12

    # =========================================================
    # CISCO NX-OS — Spine and Leaf Switches
    # =========================================================
    cisco_nxos:
      children:
        spine_switches:
          hosts:
            spine-01:
              ansible_host: 172.16.0.21
            spine-02:
              ansible_host: 172.16.0.22
        leaf_switches:
          hosts:
            leaf-01:
              ansible_host: 172.16.0.23
            leaf-02:
              ansible_host: 172.16.0.24

    # =========================================================
    # PALO ALTO — Firewall
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
    # LOGICAL GROUPINGS (cross-platform)
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

### Creating Group Variables

Group variables define connection settings that apply to all devices in a group. This is where `ansible_network_os`, `ansible_user`, and `ansible_password` live.

```bash
# Create group_vars files for each platform
```

**`inventory/group_vars/all.yml`** — settings for every device:

```yaml
---
# Global settings applied to all devices
ansible_user: ansible
ansible_password: ansible123    # Will be vaulted in Part 12
```

**`inventory/group_vars/cisco_ios.yml`**:

```yaml
---
ansible_network_os: cisco.ios.ios
ansible_connection: network_cli
ansible_become: true
ansible_become_method: enable
ansible_become_password: ansible123    # Enable password — will be vaulted in Part 12
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
ansible_httpapi_validate_certs: false    # Self-signed cert in lab
```

**`inventory/group_vars/linux_hosts.yml`**:

```yaml
---
ansible_connection: ssh
ansible_python_interpreter: /usr/bin/python3
```

### ### ℹ️ Info

> Notice that `paloalto` uses `ansible_connection: ansible.netcommon.httpapi` instead of `network_cli`. PAN-OS automation goes through the XML API over HTTPS, not SSH CLI. This is a fundamental difference from the Cisco devices and explains why PAN-OS playbooks look different — they use API-based modules rather than CLI parsing modules.

### Verifying the Inventory

```bash
cd ~/projects/ansible-network

# Show the full inventory structure as JSON
ansible-inventory -i inventory/hosts.yml --list

# Show the inventory as a tree graph
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

### Testing Ansible Connectivity to the Lab

With the inventory in place and the lab running, I run a quick connectivity test:

```bash
# Test connection to all IOS-XE devices
ansible cisco_ios -m ansible.netcommon.net_ping -i inventory/hosts.yml

# Test connection to NX-OS devices
ansible cisco_nxos -m ansible.netcommon.net_ping -i inventory/hosts.yml

# Test connection to Linux hosts
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

### ### 💡 Tip

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

## 6.11 — Containerlab Day-to-Day Workflow

This is the routine I follow at the start and end of every lab session:

### Starting a Lab Session

```bash
# 1. Navigate to the project
cd ~/projects/ansible-network

# 2. Activate the virtual environment
source ~/venvs/ansible-network/bin/activate

# 3. Start a tmux session (from Part 1)
tmux new -s ansible-lab   # or attach: tmux attach -t ansible-lab

# 4. Deploy the lab (if not already running)
cd containerlab
sudo containerlab deploy -t enterprise-lab.yml

# 5. Wait for devices to finish booting (~5 minutes for NX-OS)
# Watch NX-OS boot progress:
docker logs -f clab-enterprise-lab-spine-01

# 6. Test connectivity
cd ~/projects/ansible-network
./scripts/test-connectivity.sh

# 7. Start working
```

### Ending a Lab Session

```bash
# Option A: Leave it running (devices preserve state, uses RAM)
# Just detach from tmux: Ctrl+B, d

# Option B: Stop containers (preserves state, frees RAM)
docker ps --filter "label=containerlab" -q | xargs docker stop

# Option C: Destroy the lab completely (frees all resources)
cd ~/projects/ansible-network/containerlab
sudo containerlab destroy -t enterprise-lab.yml
```

### ### 💡 Tip

> With 16GB of RAM on the Proxmox VM, I can comfortably leave the lab running between sessions. NX-OS containers each use about 2–4GB of RAM. The full 9-node topology uses roughly 12–16GB total. If Proxmox is running other VMs, destroying the lab between sessions and redeploying is the better practice — it keeps resources available for the host.

---

## 6.12 — Common Gotchas in This Section

### ### 🪲 Gotcha — `containerlab deploy` fails with "permission denied" on Docker socket

```bash
# Error:
# Cannot connect to the Docker daemon. Permission denied.

# Fix: confirm my user is in the docker group
groups $USER    # Should show "docker" in the list

# If not, add it and re-login
sudo usermod -aG docker $USER
newgrp docker
```

### ### 🪲 Gotcha — Devices show as running but SSH times out

NX-OS and PAN-OS take several minutes to fully boot after the container starts. The container shows as "running" in Docker as soon as the process starts — not when the NOS is fully booted and accepting SSH.

```bash
# Check if SSH is actually available yet
nc -zv 172.16.0.21 22    # spine-01

# Watch the boot log for "login:" prompt
docker logs -f clab-enterprise-lab-spine-01 | grep -i "login\|ready\|cisco"
```

I wait until I see the login prompt in the logs before running Ansible against NX-OS devices.

### ### 🪲 Gotcha — Management IP conflicts with existing Docker networks

If I already have Docker networks using the `172.16.0.0/24` range, Containerlab will fail to create the management network:

```bash
# List existing Docker networks
docker network ls

# Check if 172.16.0.0/24 is already in use
docker network inspect bridge | grep -A 5 "Subnet"

# If there's a conflict, change the mgmt subnet in enterprise-lab.yml
# mgmt:
#   ipv4-subnet: 172.16.100.0/24    # ← change to an unused range
```

### ### 🪲 Gotcha — `containerlab destroy` leaves orphaned Docker networks

Sometimes after a failed deploy or crash, Docker networks aren't cleaned up:

```bash
# List Docker networks
docker network ls | grep clab

# Remove orphaned networks manually
docker network rm clab-enterprise-lab-mgmt-net

# Nuclear option — remove all unused Docker networks
docker network prune
```

### ### 🪲 Gotcha — Topology file changes don't apply after redeploy

Containerlab caches some state in the `clab-<labname>/` directory. If I change the topology file and redeploy without destroying first, the old config may persist:

```bash
# Always destroy before redeploying after topology changes
sudo containerlab destroy -t enterprise-lab.yml --cleanup
sudo containerlab deploy -t enterprise-lab.yml
```

---

## 6.13 — Committing the Lab Files to Git

Everything created in this part goes into version control:

```bash
cd ~/projects/ansible-network

# Stage all new files
git add containerlab/
git add inventory/
git add scripts/

# Check what's being committed
git status

# Commit
git commit -m "feat(lab): add Containerlab enterprise topology and Ansible inventory

- Add 9-node enterprise topology: 2x IOS-XE WAN routers, 1x PAN-OS firewall,
  2x NX-OS spine switches, 2x NX-OS leaf switches, 2x Linux hosts
- Add management network 172.16.0.0/24 with static IPs per device
- Add Ansible inventory with platform groups and logical groupings
- Add group_vars for cisco_ios, cisco_nxos, paloalto, linux_hosts
- Add connectivity test script"
```

---

The lab is running. I have a full enterprise-style topology — WAN routers, a firewall, spine/leaf switches, and Linux hosts — all reachable from Ansible via the 172.16.0.0/24 management network. Every playbook from here on runs against real devices.

