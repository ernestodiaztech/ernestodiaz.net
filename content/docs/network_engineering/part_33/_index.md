---
draft: true
title: '33 - Proxmox VM Architecture'
description: "Part 33 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="VS Code" color="blue" >}}
{{< badge content="Linux" color="red" >}}

## Part 33: Proxmox VM Architecture & Network Design

> *Every tool I've automated in Parts 1–32 has lived on a single Ubuntu control node — which is fine for learning, but it's nothing like how real infrastructure is deployed. Real environments have separation of concerns: the monitoring system doesn't share a host with the thing it's monitoring, the logging system doesn't go down when the automation server is busy running a 500-device playbook, and the source of truth doesn't compete for RAM with Containerlab. Part 33 is the architectural foundation for everything that follows. I'm not installing anything yet — I'm designing the environment properly before touching a single command, the same way I'd design a network before pulling cable.*

---

## 33.1 — Why Separate VMs Instead of One Big Server

Before designing anything, it's worth being explicit about why the stack is split across multiple VMs rather than running everything on the control node with Docker Compose.

### The separation-of-concerns argument

```
Single-node approach (what I had before):
  ┌─────────────────────────────────────┐
  │         ubuntu-control              │
  │  Ansible + Containerlab + Netbox +  │
  │  Zabbix + Prometheus + Graylog +    │
  │  Git + everything else              │
  └─────────────────────────────────────┘

  Problems:
  - One package conflict breaks everything
  - Containerlab lab restart takes down monitoring
  - Can't snapshot just the Netbox VM before an upgrade
  - Resource contention between Containerlab (CPU-heavy)
    and OpenSearch (RAM-heavy) causes both to perform badly
  - No blast radius control — a misconfigured playbook
    targeting localhost can break the tool running it

Multi-VM approach (what I'm building):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ network  │  │  netbox  │  │automation│
  │   lab    │  │          │  │          │
  └──────────┘  └──────────┘  └──────────┘
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │monitoring│  │observ-   │  │ logging  │
  │          │  │ability   │  │          │
  └──────────┘  └──────────┘  └──────────┘

  Benefits:
  - Snapshot individual VMs before upgrades
  - Resource isolation — Containerlab gets its RAM,
    OpenSearch gets its RAM, no competition
  - Failure isolation — Graylog going down doesn't
    affect Ansible running a playbook
  - Mirrors enterprise architecture — each VM maps to
    a real-world server or container cluster role
  - Ansible manages the VMs themselves (IaC for infra)
```

### How this mirrors enterprise architecture

```
Home lab VM          Enterprise equivalent
─────────────────────────────────────────────
network-lab          Dedicated network simulation/staging environment
netbox               IPAM/DCIM server (Netbox or ServiceNow CMDB)
automation           Ansible Tower / AWX server + CI/CD runner
monitoring           Zabbix server or SolarWinds NMS
observability        Prometheus/Grafana stack or Datadog
logging              SIEM / Splunk / Graylog cluster
```

Understanding this mapping matters for career development — when an employer describes their stack, I can immediately place each component against something I've built and operated.

---

## 33.2 — Proxmox Network Design

### Recommended bridge architecture

For a lab that mirrors enterprise network segmentation, three bridges is the right answer — not one (too flat) and not five (unnecessarily complex for a single host).

```
Proxmox Host
│
├── vmbr0 — INFRASTRUCTURE MANAGEMENT
│   Purpose:  VM management, Ansible control plane, SSH, web UIs
│   Subnet:   172.16.0.0/24
│   Gateway:  172.16.0.1 (Proxmox host / home router)
│   Connects: All VMs have an interface here (their primary NIC)
│             Existing Containerlab mgmt-net also on this range
│   Real-world equivalent: Out-of-band management network
│
├── vmbr1 — LAB DATA PLANE
│   Purpose:  Containerlab virtual links, inter-device traffic,
│             simulated WAN/LAN traffic between network devices
│   Subnet:   10.0.0.0/8 (broad — Containerlab subnets carve from this)
│   Gateway:  none (L2 only — Containerlab handles routing internally)
│   Connects: network-lab VM only
│   Real-world equivalent: Production network / data plane
│
└── vmbr2 — SERVICES / TELEMETRY
    Purpose:  Metrics scraping, syslog, SNMP traps, log shipping
              between VMs — keeps telemetry traffic off the mgmt bridge
    Subnet:   192.168.100.0/24
    Gateway:  192.168.100.1 (Proxmox host interface on this bridge)
    Connects: All VMs have a secondary interface here
    Real-world equivalent: Dedicated monitoring/telemetry network
                           (common in enterprise — keeps SNMP/syslog
                            off the management plane)
```

### Why three bridges and not one

```
vmbr0 (management) — Ansible SSH, web UIs, AWX, Git push/pull.
  If this is mixed with telemetry, a Prometheus scrape storm or
  syslog flood can saturate the bridge and break Ansible connectivity.

vmbr1 (lab data) — Containerlab creates its own internal Docker
  networks for data plane links. This bridge provides the uplink
  from the network-lab VM to the Proxmox host for any traffic that
  needs to escape the lab (simulated internet, etc.). Keeping it
  separate prevents lab traffic from reaching monitoring VMs directly.

vmbr2 (services/telemetry) — SNMP polling, Prometheus scrapes,
  syslog UDP streams, and log shipping are all high-volume and
  low-latency-sensitive. A dedicated bridge ensures this traffic
  has its own path and doesn't interfere with SSH sessions on vmbr0.
```

### Setting up the bridges on Proxmox

```bash
# Edit /etc/network/interfaces on the Proxmox host
# (or use the Proxmox web UI: Host → Network → Create → Linux Bridge)

# View current network config
cat /etc/network/interfaces

# Add vmbr1 and vmbr2 if not present
# Proxmox web UI: Datacenter → pve → Network → Create → Linux Bridge

# vmbr1 — Lab data plane (no IP on host — L2 passthrough only)
# Bridge ports: leave blank (no physical uplink — internal only)
# IPv4/CIDR: leave blank

# vmbr2 — Services/telemetry (host gets .1)
# Bridge ports: leave blank
# IPv4/CIDR: 192.168.100.1/24

# Apply without reboot
ifreload -a

# Verify all three bridges are up
ip link show type bridge
# Expected:
# vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
# vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
# vmbr2: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```

---

## 33.3 — VM Specifications

Hardware available: 64GB+ RAM, 8+ cores, SSD storage. The allocations below leave headroom for Containerlab running 7 network device VMs simultaneously alongside the full stack.

### VM allocation table

| VM Name | Role | vCPU | RAM | Disk | vmbr0 IP | vmbr2 IP |
|---|---|---|---|---|---|---|
| `network-lab` | Containerlab + vrnetlab | 8 | 24GB | 120GB | 172.16.0.10 | 192.168.100.10 |
| `netbox` | Netbox + PostgreSQL + Redis | 2 | 4GB | 40GB | 172.16.0.20 | 192.168.100.20 |
| `automation` | Ansible + AWX + Gitea | 4 | 8GB | 60GB | 172.16.0.30 | 192.168.100.30 |
| `monitoring` | Zabbix + SNMP | 2 | 4GB | 40GB | 172.16.0.40 | 192.168.100.40 |
| `observability` | Prometheus + Grafana + Loki | 4 | 8GB | 80GB | 172.16.0.50 | 192.168.100.50 |
| `logging` | Graylog + OpenSearch | 4 | 12GB | 100GB | 172.16.0.60 | 192.168.100.60 |
| **Total** | | **24 vCPU** | **60GB** | **440GB** | | |

### Sizing rationale

```
network-lab (8 vCPU / 24GB):
  Containerlab with 7 network device VMs is the most resource-
  intensive workload. CSR1000v needs ~1.5GB each, N9Kv ~2GB,
  Cat9Kv ~1.5GB, PAN-OS ~4GB. Total lab RAM ≈ 16GB minimum.
  24GB gives headroom for the host OS and Docker overhead.
  8 vCPU because vrnetlab boots VMs inside containers — each
  device gets a QEMU process that benefits from dedicated cores.

logging (4 vCPU / 12GB):
  OpenSearch is the most RAM-hungry service in the stack.
  The JVM heap for a single-node lab instance needs at least
  4GB, and OpenSearch recommends half of available RAM for heap.
  12GB gives 6GB to the JVM and 6GB for the OS and Graylog.
  Do not go below 8GB for this VM — OpenSearch will OOM.

observability (4 vCPU / 8GB):
  Prometheus is CPU-efficient but RAM grows with the number of
  metrics and retention period. Loki is similarly RAM-sensitive
  when ingesting high-volume log streams. 8GB is comfortable
  for a lab with ~50 scrape targets and 30-day retention.

automation (4 vCPU / 8GB):
  AWX runs several containers (web, task, redis, postgresql).
  The AWX task container spawns Ansible runner processes for
  each concurrent job — 4 vCPU prevents job queue starvation.

netbox / monitoring (2 vCPU / 4GB each):
  Netbox with PostgreSQL and Redis is lightweight at lab scale.
  Zabbix server with a small device count is similarly modest.
  These can be bumped to 4GB/4 vCPU later if needed.
```

---

## 33.4 — Full IP Address Plan

### Infrastructure management network (vmbr0 — 172.16.0.0/24)

```
172.16.0.1    Proxmox host (vmbr0 gateway / home router)
172.16.0.10   network-lab VM
172.16.0.20   netbox VM
172.16.0.30   automation VM (Ansible control node + AWX + Gitea)
172.16.0.40   monitoring VM (Zabbix)
172.16.0.50   observability VM (Prometheus / Grafana / Loki)
172.16.0.60   logging VM (Graylog / OpenSearch)

── Containerlab devices (existing — Part 6) ──────────────────
172.16.0.10   Proxmox host (also the Docker bridge gateway for
              Containerlab mgmt-net — same host, same IP)
172.16.0.11   wan-r1     (CSR1000v)
172.16.0.12   wan-r2     (CSR1000v)
172.16.0.21   dist-01    (Cat9Kv)
172.16.0.31   spine-01   (N9Kv)
172.16.0.33   leaf-01    (Cat9Kv)
172.16.0.34   leaf-02    (Cat9Kv)
172.16.0.51   panos-fw01 (PAN-OS)

── Reserved ──────────────────────────────────────────────────
172.16.0.100–199  DHCP range (physical hosts, laptops)
172.16.0.200–250  Future VMs / services
172.16.0.250      Proxmox web UI (same as host — port 8006)
```

### Services / telemetry network (vmbr2 — 192.168.100.0/24)

```
192.168.100.1   Proxmox host (vmbr2 gateway)
192.168.100.10  network-lab VM (secondary NIC)
192.168.100.20  netbox VM
192.168.100.30  automation VM
192.168.100.40  monitoring VM
192.168.100.50  observability VM
192.168.100.60  logging VM

Purpose of this range:
  Prometheus scrapes metrics FROM: .10, .20, .30, .40, .60
  Zabbix polls devices via SNMP TO: .10 (network-lab reaches devices)
  Graylog receives syslog FROM: .10 (network-lab forwards device syslog)
  Loki receives logs FROM: Promtail agents on all VMs
  All telemetry traffic stays on this bridge — vmbr0 stays clean
```

### Why Containerlab devices use 172.16.0.x and not 192.168.100.x

Containerlab devices (wan-r1, spine-01, etc.) live inside Docker networks on the network-lab VM. They are reachable on 172.16.0.x because Containerlab's `mgmt-net` bridge uses that subnet and the Proxmox host routes between vmbr0 and the Docker bridge. The devices themselves don't have vmbr2 interfaces — all monitoring traffic (SNMP, syslog) flows through the network-lab VM, which does have a 192.168.100.x address and acts as the relay point.

```
Monitoring flow:
  monitoring VM (192.168.100.40)
    → polls network-lab VM (192.168.100.10)
      → Docker bridge routes to wan-r1 (172.16.0.11)
        → SNMP response returns via same path

Syslog flow:
  wan-r1 (172.16.0.11) sends syslog to 172.16.0.60
    (logging VM's vmbr0 IP — devices know this address)
  OR: devices send to 192.168.100.60
    (if the network-lab VM's routing is set up to forward)
  Both work — 172.16.0.60 is simpler since devices already
  have 172.16.0.0/24 routes via Containerlab mgmt-net
```

---

## 33.5 — Operating System and Cloud-Init Template

All VMs use **Ubuntu 24.04 LTS** — consistent with the existing control node, same package ecosystem, same Ansible connection assumptions.

### Creating the cloud-init template (run once on Proxmox host)

```bash
# On the Proxmox host — SSH in or use the shell console

# Download Ubuntu 24.04 cloud image
cd /var/lib/vz/template/iso/
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create a base VM (ID 9000 — template convention)
qm create 9000 \
  --name ubuntu-2404-template \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26 \
  --scsihw virtio-scsi-pci

# Import the cloud image as the boot disk
qm importdisk 9000 \
  /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img \
  local-lvm

# Attach disk, set boot order, add cloud-init drive
qm set 9000 \
  --scsi0 local-lvm:vm-9000-disk-0,discard=on \
  --ide2 local-lvm:cloudinit \
  --boot c \
  --bootdisk scsi0 \
  --serial0 socket \
  --vga serial0

# Configure cloud-init defaults
qm set 9000 \
  --ciuser ansible \
  --cipassword "$(openssl passwd -6 'AnsibleService#2025')" \
  --sshkeys ~/.ssh/authorized_keys \
  --ipconfig0 ip=dhcp \
  --nameserver 8.8.8.8

# Convert to template (can no longer start this VM directly)
qm template 9000

# Verify template exists
qm list | grep 9000
# 9000 ubuntu-2404-template  template  2048
```

### Cloud-init template — what it provides

```
Every VM cloned from this template gets:
  - Ubuntu 24.04 LTS minimal install
  - User: ansible (matches existing control node user)
  - SSH key auth from Proxmox host key (Ansible can connect immediately)
  - Python3 pre-installed (cloud images include it)
  - cloud-init runs on first boot: sets hostname, expands disk,
    configures network from the per-VM cloud-init settings

What Ansible adds in Part 34:
  - Static IP configuration (replaces DHCP)
  - Additional packages (per-role installs)
  - Second NIC on vmbr2 (192.168.100.x)
  - Hardened SSH config
  - Firewall rules (ufw)
```

---

## 33.6 — VM Creation (Key Commands)

Part 34 handles VM provisioning entirely via Ansible. These commands are the manual equivalent — shown so I understand what Ansible is doing under the hood, and as a fallback if I need to create a VM by hand.

```bash
# Clone the template to create a new VM
# Usage: qm clone <template-id> <new-vm-id> --name <name> --full

qm clone 9000 101 --name network-lab --full
qm clone 9000 102 --name netbox --full
qm clone 9000 103 --name automation --full
qm clone 9000 104 --name monitoring --full
qm clone 9000 105 --name observability --full
qm clone 9000 106 --name logging --full

# Resize disks to match the spec table
qm resize 101 scsi0 120G
qm resize 102 scsi0 40G
qm resize 103 scsi0 60G
qm resize 104 scsi0 40G
qm resize 105 scsi0 80G
qm resize 106 scsi0 100G

# Set CPU and RAM per VM
qm set 101 --cores 8 --memory 24576
qm set 102 --cores 2 --memory 4096
qm set 103 --cores 4 --memory 8192
qm set 104 --cores 2 --memory 4096
qm set 105 --cores 4 --memory 8192
qm set 106 --cores 4 --memory 12288

# Add second NIC on vmbr2 (services/telemetry bridge) to all VMs
for VMID in 101 102 103 104 105 106; do
  qm set $VMID --net1 virtio,bridge=vmbr2
done

# network-lab also needs vmbr1 (lab data plane)
qm set 101 --net2 virtio,bridge=vmbr1

# Set static IPs via cloud-init
qm set 101 --ipconfig0 ip=172.16.0.10/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.10/24 \
           --hostname network-lab

qm set 102 --ipconfig0 ip=172.16.0.20/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.20/24 \
           --hostname netbox

qm set 103 --ipconfig0 ip=172.16.0.30/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.30/24 \
           --hostname automation

qm set 104 --ipconfig0 ip=172.16.0.40/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.40/24 \
           --hostname monitoring

qm set 105 --ipconfig0 ip=172.16.0.50/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.50/24 \
           --hostname observability

qm set 106 --ipconfig0 ip=172.16.0.60/24,gw=172.16.0.1 \
           --ipconfig1 ip=192.168.100.60/24 \
           --hostname logging

# Start all VMs
for VMID in 101 102 103 104 105 106; do
  qm start $VMID
  echo "Started VM $VMID"
done

# Verify all VMs are running
qm list
```

---

## 33.7 — Ansible Inventory Extension

The existing `inventory/hosts.yml` covers network devices. It now needs an infrastructure group for the new VMs. This is added manually here — Part 34 covers the Proxmox dynamic inventory plugin that can auto-populate this from the Proxmox API.

```bash
# Add infrastructure hosts to existing inventory
cat >> ~/projects/ansible-network/inventory/hosts.yml << 'EOF'

    # =========================================================
    # INFRASTRUCTURE VMs (Proxmox guests)
    # =========================================================
    infrastructure:
      children:

        lab_hosts:
          hosts:
            network-lab:
              ansible_host: 172.16.0.10

        netbox_hosts:
          hosts:
            netbox:
              ansible_host: 172.16.0.20

        automation_hosts:
          hosts:
            automation:
              ansible_host: 172.16.0.30

        monitoring_hosts:
          hosts:
            monitoring:
              ansible_host: 172.16.0.40

        observability_hosts:
          hosts:
            observability:
              ansible_host: 172.16.0.50

        logging_hosts:
          hosts:
            logging:
              ansible_host: 172.16.0.60

    # All Linux VMs — for base hardening playbook
    linux_vms:
      children:
        lab_hosts:
        netbox_hosts:
        automation_hosts:
        monitoring_hosts:
        observability_hosts:
        logging_hosts:
EOF
```

### Group variables for infrastructure VMs

```bash
cat > ~/projects/ansible-network/inventory/group_vars/infrastructure.yml << 'EOF'
---
# All infrastructure VMs run Ubuntu 24.04 and use standard SSH
ansible_connection: ssh
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
ansible_python_interpreter: /usr/bin/python3

# Telemetry network interface (vmbr2) — consistent naming across all VMs
# Ubuntu uses predictable interface names: ens18 = NIC1 (vmbr0), ens19 = NIC2 (vmbr2)
telemetry_interface: ens19
telemetry_network: 192.168.100.0/24

# NTP (same servers as network devices)
ntp_servers:
  - 216.239.35.0
  - 216.239.35.4

# Syslog destination — all VMs forward to Graylog
syslog_server: 192.168.100.60
syslog_port: 514

# Proxmox host details (used by provisioning playbooks in Part 34)
proxmox_host: 172.16.0.1
proxmox_api_user: root@pam
proxmox_api_token_id: ansible
# proxmox_api_token_secret: "{{ vault_proxmox_token }}"
EOF
```

### ### ℹ️ Info: Dynamic inventory preview

> Part 34 introduces the `community.general.proxmox` inventory plugin, which replaces the static infrastructure entries above with auto-discovered VMs pulled from the Proxmox API. The plugin can filter by tags, pools, or node — so adding a new VM in Proxmox automatically makes it available to Ansible without editing `hosts.yml`. The static entries above are useful now and serve as a fallback.

---

## 33.8 — Service Port Reference

Every service in the stack exposes one or more ports. Having this map before installation prevents port conflicts and makes firewall rules straightforward in Part 34.

```
VM: network-lab (172.16.0.10 / 192.168.100.10)
  22    SSH
  No externally exposed services — Containerlab runs internally

VM: netbox (172.16.0.20 / 192.168.100.20)
  22    SSH
  80    Netbox HTTP (nginx → gunicorn)
  443   Netbox HTTPS
  5432  PostgreSQL (localhost only)
  6379  Redis (localhost only)

VM: automation (172.16.0.30 / 192.168.100.30)
  22    SSH
  80    AWX HTTP  → redirect to 443
  443   AWX HTTPS (web UI + API)
  3000  Gitea web UI
  22xxx Gitea SSH (2222 — avoids conflict with system SSH on 22)
  5432  AWX PostgreSQL (localhost only)

VM: monitoring (172.16.0.40 / 192.168.100.40)
  22    SSH
  80    Zabbix web UI (nginx → PHP-FPM)
  443   Zabbix HTTPS
  10051 Zabbix server (active agent connections)
  162   SNMP trap receiver (UDP)

VM: observability (172.16.0.50 / 192.168.100.50)
  22    SSH
  3100  Loki HTTP API + log ingestion
  9090  Prometheus web UI + API
  9093  AlertManager web UI + API
  3000  Grafana web UI
  9100  Node Exporter (metrics about this VM itself)

VM: logging (172.16.0.60 / 192.168.100.60)
  22    SSH
  514   Graylog syslog input (UDP + TCP)
  1514  Graylog GELF input (UDP + TCP)
  2514  Graylog raw/plaintext input
  9000  Graylog web UI + REST API
  9200  OpenSearch HTTP API
  9300  OpenSearch transport (cluster — single-node, internal only)
  9600  OpenSearch performance analyser
```

### Port conflict check

```bash
# Run this on each VM after provisioning (Part 34) to verify no conflicts
ss -tlnp | awk '{print $4}' | grep -oP ':\K[0-9]+' | sort -n | uniq -d
# Any output here is a duplicate port — investigate before proceeding
```

---

## 33.9 — Architecture Diagram (Full Stack)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PROXMOX HOST                                 │
│                    172.16.0.1 / 192.168.100.1                       │
│                                                                       │
│  vmbr0 (172.16.0.0/24)    vmbr1 (lab)    vmbr2 (192.168.100.0/24) │
│  ─────────────────────    ───────────    ──────────────────────────  │
│                                                                       │
│  ┌─────────────────┐                    ┌─────────────────────────┐ │
│  │  network-lab    │◄──── vmbr1 ───────►│  (internal Containerlab │ │
│  │  172.16.0.10    │                    │   virtual links)        │ │
│  │  192.168.100.10 │                    └─────────────────────────┘ │
│  │                 │                                                  │
│  │  Containerlab   │  Docker bridge: 172.16.0.11–172.16.0.51        │
│  │  wan-r1         │  (Containerlab devices reachable on vmbr0)     │
│  │  wan-r2         │                                                  │
│  │  dist-01        │                                                  │
│  │  spine-01       │                                                  │
│  │  leaf-01/02     │                                                  │
│  │  panos-fw01     │                                                  │
│  └────────┬────────┘                                                 │
│           │ vmbr0 (SSH, web UIs, Ansible)                            │
│  ─────────┼──────────────────────────────────────────────────────── │
│           │                                                           │
│  ┌────────┴──────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │   netbox      │  │  automation  │  │       monitoring         │ │
│  │ 172.16.0.20   │  │ 172.16.0.30  │  │     172.16.0.40          │ │
│  │ 192.168.100.20│  │192.168.100.30│  │    192.168.100.40        │ │
│  │               │  │              │  │                           │ │
│  │ Netbox        │  │ Ansible/AWX  │  │ Zabbix                   │ │
│  │ PostgreSQL    │◄─┤ Gitea        │  │ SNMP receiver            │ │
│  │ Redis         │  │ Python       │  │                           │ │
│  └───────────────┘  └──────────────┘  └──────────────────────────┘ │
│                                                                       │
│  ┌───────────────────────┐  ┌──────────────────────────────────────┐│
│  │     observability     │  │            logging                   ││
│  │     172.16.0.50       │  │          172.16.0.60                 ││
│  │     192.168.100.50    │  │          192.168.100.60              ││
│  │                       │  │                                       ││
│  │ Prometheus            │  │ Graylog                              ││
│  │ Grafana               │◄─┤ OpenSearch                           ││
│  │ Loki                  │  │ Syslog/SNMP trap receiver            ││
│  │ AlertManager          │  │                                       ││
│  └───────────────────────┘  └──────────────────────────────────────┘│
│                                                                       │
│  vmbr2 telemetry traffic (192.168.100.0/24) ─────────────────────── │
│  Prometheus scrapes, Loki log shipping, syslog relay all use vmbr2  │
└─────────────────────────────────────────────────────────────────────┘
```

### Data flows

```
Ansible (automation VM) ──SSH──► all VMs and network devices
Netbox (netbox VM) ──────API───► Ansible dynamic inventory source
AWX (automation VM) ─────pull──► Gitea (automation VM) for playbooks
Zabbix (monitoring VM) ──SNMP──► network devices (via network-lab VM)
Prometheus (obs. VM) ───scrape─► Node Exporters on all VMs
Prometheus (obs. VM) ───scrape─► SNMP Exporter on monitoring VM
Graylog (logging VM) ─←syslog── network devices + all VMs
Grafana (obs. VM) ──────query──► Prometheus + Loki
AlertManager (obs. VM) ─webhook► AWX (automation VM) — self-healing
Loki (obs. VM) ────────←push──── Promtail agents on all VMs
```

---

## 33.10 — Pre-Flight Checklist

Before moving to Part 34 (VM provisioning with Ansible), verify this foundation is solid:

```
[ ] Proxmox host has three bridges configured and UP:
    [ ] vmbr0 — 172.16.0.1/24 — management
    [ ] vmbr1 — no IP — lab data plane
    [ ] vmbr2 — 192.168.100.1/24 — services/telemetry

[ ] Ubuntu 24.04 cloud-init template created (VM ID 9000)
    [ ] qm list shows: 9000 ubuntu-2404-template template
    [ ] Template has ansible user + SSH key configured

[ ] Six VMs cloned and started:
    [ ] 101 network-lab  172.16.0.10  192.168.100.10  (8c/24GB/120GB)
    [ ] 102 netbox       172.16.0.20  192.168.100.20  (2c/4GB/40GB)
    [ ] 103 automation   172.16.0.30  192.168.100.30  (4c/8GB/60GB)
    [ ] 104 monitoring   172.16.0.40  192.168.100.40  (2c/4GB/40GB)
    [ ] 105 observability 172.16.0.50 192.168.100.50  (4c/8GB/80GB)
    [ ] 106 logging      172.16.0.60  192.168.100.60  (4c/12GB/100GB)

[ ] All VMs reachable from the Proxmox host:
    for ip in 172.16.0.{10,20,30,40,50,60}; do
      ping -c1 -W2 $ip > /dev/null && echo "✓ $ip" || echo "✗ $ip"
    done

[ ] SSH works from automation VM to all other VMs:
    # Run from 172.16.0.30 (automation VM)
    for host in 172.16.0.{10,20,40,50,60}; do
      ssh -o StrictHostKeyChecking=no ansible@$host hostname
    done

[ ] Telemetry network reachable:
    for ip in 192.168.100.{10,20,30,40,50,60}; do
      ping -c1 -W2 $ip > /dev/null && echo "✓ $ip" || echo "✗ $ip"
    done

[ ] inventory/hosts.yml updated with infrastructure group
[ ] inventory/group_vars/infrastructure.yml created
[ ] ansible-inventory --graph shows infrastructure group with all 6 VMs
[ ] ansible linux_vms -m ansible.builtin.ping -i inventory/hosts.yml
    — all 6 VMs return SUCCESS

[ ] Port reference table saved — no conflicts identified
[ ] Architecture diagram reviewed — data flows understood
```

---

*The foundation is in place. Six VMs are running, the network is segmented across three bridges, and Ansible can reach everything. Part 34 takes this further — instead of clicking through Proxmox or running qm commands by hand, every VM is provisioned, hardened, and prepared for its role by Ansible playbooks. The infrastructure becomes code.*

*Next up: **Part 34 — Infrastructure-as-Code: Provisioning VMs with Ansible***
