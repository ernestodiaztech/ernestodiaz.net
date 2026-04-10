---
draft: false
title: '4. Containerlab'
description: "Part 4 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: false
weight: 4
---

{{< badge "Ansible" >}}
{{< badge content="Git" color="blue" >}}
{{< badge content="Linux" color="red" >}}

---

Here I will setup a dedicated VM for Containerlab, install vrnetlab to build images, deploy 2 Cat9kv nodes, and veryify SSH reachability from the Ansible control node.

{{< objectives title="What I Will Be Completing In This Part" >}}
- Provision a dedicated VM for Containerlab with nested virtualization and KVM passthrough
- Install Docker and Containerlab
- Transfer the Cat9kv `.qcow` image and build a vrnetlab Docker image from it
- Write a Containerlab topology file deploying `wan-r1` and `wan-r2`
- Deploy the topology and verify console access to both devices
- Configure management network routing so `ansible-ctrl` can reach both nodes over SSH
{{< /objectives >}}

---

## VM Specifications

Containerlab runs network device images as containers, but vrnetlab images boot a full virtual machine inside each container using QEMU/KVM. This requires a lot of resources, so I need to add enough RAM and CPU to the Containerlab VM.

{{< codeblock lang="Specs" copy=false >}}
OS:        Ubuntu Server 22.04 LTS
Hostname:  clab
CPU:       16 vCPU
RAM:       64 GB
Disk:      100 GB
Network:   1 NIC (bridged to management network)
CPU Type:  host
{{< /codeblock >}}

I assigned the VM a static IP and added a DNS entry for `clab`.

---

## Docker Installation

I installed Docker the same I installed it on my Gitea VM.

First, I updated the packages

{{< codeblock lang="Bash" syntax="bash" >}}
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg
{{< /codeblock >}}

Add Docker GPG key

{{< codeblock lang="Bash" syntax="bash" >}}
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gng
sudo chmod a+r /etc/apt/keyrings/docker.gpg
{{< /codeblock >}}

Add Docker repo

{{< codeblock lang="Bash" syntax="bash" >}}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
{{< /codeblock >}}

Then install Docker

{{< codeblock lang="Bash" syntax="bash" >}}
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
{{< /codeblock >}}

Add user to docker group

{{< codeblock lang="Bash" syntax="bash" >}}
sudo usermod -aG docker $USER
newgrp docker
{{< /codeblock >}}

Then verify

{{< codeblock lang="Bash" syntax="bash" >}}
docker version
{{< /codeblock >}}

---

## Containerlab Installation

I used the provided script from the official website to install Containerlab.

{{< codeblock lang="Bash" syntax="bash" >}}
bash -c "$(curl -sL https://get.containerlab.dev)"
{{< /codeblock >}}

---

### Cat9kv Image

I created a directory on the `clab` VM to store the images.

{{< codeblock lang="Bash" syntax="bash" >}}
sudo mkdir -p /opt/vrnetlab-images
sudo chown $USER:$USER /opt/vrnetlab-images
{{< /codeblock >}}

Then I used WinSCP to transfer the image to that directory.

### VRNetLab

vrnetlab is a project that packages network OS images into Docker containers. Each container runs QEMU internally to boot the actual network OS, but Containerlab manages it like any other container.

I installed the build dependencies and cloned the vrnetlab repo:

{{< codeblock lang="Bash" syntax="bash" lines="true" >}}
sudo apt install -y make git qemu-utils
cd /opt
git clone https://github.com/hellt/vrnetlab.git
cd vrnetlab
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: `make` is the build tool used by vrnetlab's Makefules.

Line 3:
: This is the `hellt/vrnetlab` fork, which is the actively maintained version and the one Containerlab officially supports.
{{< /line-explain >}}

I then copied the Cat9kv `.qcow2` image into the Cat9kv build directory.

{{< codeblock lang="Bash" syntax="bash" >}}
cp /opt/vrnetlab-images/cat9kv-prd-17.15.01.qcow2 /opt/vrnetlab/cat9kv/
{{< /codeblock >}}

{{< lab-callout type="info" >}}
Each network OS has its own directory in the vrnetlab repo. The Makefile in each directory knows how to build a Docker image from the `.qcow2` file placed inside it.
{{< /lab-callout >}}

Then I ran the build:

{{< codeblock lang="Bash" syntax="bash" >}}
cd /opt/vrnetlab/cat9kv
make
{{< /codeblock >}}

The build takes several minutes since it creates a Docker image that packages the `.qcow2` file with QEMU and a launch script.

To verify the image was created:

{{< codeblock lang="Bash" syntax="bash" >}}
docker images | grep cat9kv
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
vrnetlab/vr-cat9kv   17.15.01   abc123def456   2 minutes ago   1.8GB
{{< /codeblock >}}

---

## Topology File

Next, I created a directory for the topology files and wrote the first one that will deploy 2 Cat9kv nodes as WAN routers.

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p /opt/clab/topologies
cd /opt/clab/topologies
{{< /codeblock >}}

{{< codeblock file="/opt/clab/topologies/wan.clab.yml" syntax="yaml" lines="true" >}}
---
name: wan

mgmt:
  network: mgmt-net
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:
    wan-r1:
      kind: cisco_cat9kv
      image: vrnetlab/vr-cat9kv:17.15.01
      mgmt-ipv4: 172.20.20.11
      startup-config: ../configs/wan-r1.cfg

    wan-r2:
      kind: cisco_cat9kv
      image: vrnetlab/vr-cat9kv:17.15.01
      mgmt-ipv4: 172.20.20.12
      startup-config: ../configs/wan-r2.cfg

  links:
    - endpoints: ["wan-r1:GigabitEthernet2", "wan-r2:GigabitEthernet2"]
{{< /codeblock >}}

{{< line-explain >}}
Line 2:
: The topology name. Containerlab prefixes all container names with `clab-`.

Lines 4-6:
: The management network configuration. Continerlab creates a Docker bridge network named `mgmt-net` and attaches node's management interface to it.

Lines 11, 17:
: `kind: cisco_cat9kv` tells Containerlab this is a vrnetlab-based Cat9kv node.

Lines 13, 19:
: Static management IP.

Lines 14, 20:
: Startup configuration files that Containerlab pushes to the devices on first boot.

Lines 22-23:
: A point-to-point link between the 2 routers.
{{< /line-explain >}}

I then created the startup config files that will be applied on boot.

{{< codeblock lang="Bash" syntax="bash" >}}
mkdir -p /opt/clab/configs
{{< /codeblock >}}

{{< codeblock file="/opt/clab/configs/wan-r1.cfg" syntax="cfg" lines="true" >}}
hostname wan-r1
!
username admin privilege 15 secret admin
!
ip domain-name lab.local
crypto key generate rsa modulus 2048
!
interface GigabitEthernet1
 ip address dhcp
 no shutdown
!
ip ssh version 2
ip scp server enable
!
line vty 0 4
 login local
 transport input ssh
!
enable secret admin
!
{{< /codeblock >}}

{{< line-explain >}}
Line 1:
: Explanation here.

Line 2:
: Explanation here.
{{< /line-explain >}}

{{< codeblock file="/opt/clab/configs/wan-r2.cfg" syntax="yaml" >}}
hostname wan-r2
!
username admin privilege 15 secret admin
!
ip domain-name lab.local
crypto key generate rsa modulus 2048
!
interface GigabitEthernet1
 ip address dhcp
 no shutdown
!
ip ssh version 2
ip scp server enable
!
line vty 0 4
 login local
 transport input ssh
!
enable secret admin
!
{{< /codeblock >}}

{{< lab-callout type="warning" >}}
The credentials in the startup config are intentionally simple bootstrap credentials. It's so Ansible can connect immediately.
{{< /lab-callout >}}

{{< lab-callout type="tip" >}}
Make sure the credentials in the startup configs match what's in the Ansible vault files.
{{< /lab-callout >}}

---

### Deploying the Topology

{{< codeblock lang="Bash" syntax="bash" >}}
cd /opt/clab/topologies
sudo clab deploy -t wan.clab.yml
{{< /codeblock >}}

{{< line-explain >}}
-t:
: Specifies the topology file to deply
{{< /line-explain >}}

Once Containerlab reports the deployment is complete, I checked the status:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo clab inspect -t wan.clab.yml
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
+---+------------------+--------------+---------------------+------+---------+----------------+------+
| # |       Name       | Container ID |        Image        | Kind |  State  |  IPv4 Address  | IPv6 |
+---+------------------+--------------+---------------------+------+---------+----------------+------+
| 1 | clab-wan-wan-r1  | abc123...    | vrnetlab/vr-cat9kv  | ...  | running | 172.20.20.11/24|      |
| 2 | clab-wan-wan-r2  | def456...    | vrnetlab/vr-cat9kv  | ...  | running | 172.20.20.12/24|      |
+---+------------------+--------------+---------------------+------+---------+----------------+------+
{{< /codeblock >}}

---

## Accessing Devices

I verified console and SSH access from the `clab` host directly before setting up cross-VM connectivity.

**Console access via Containerlab**

{{< codeblock lang="Bash" syntax="bash" >}}
sudo docker exec -it clab-wan-wan-r1 telnet localhost 5000
{{< /codeblock >}}

{{< line-explain >}}
telnet localhost 5000:
: vrnetlab containers expose the device console on port 5000 inside the container. This connection is equivalent to connecting a console cable. Use `Ctrl+]` then `quite` to exit telnet.
{{< /line-explain >}}

I should see the IOS-XE prompt:

{{< codeblock lang="Expected Output" copy="false" >}}
wan-r1#
{{< /codeblock >}}

---

**SSH access from the clab host**

{{< codeblock lang="Bash" syntax="bash" >}}
ssh admin@172.20.20.11
{{< /codeblock >}}

After accepting the host key and entering the password, I should land on the IOS-XE privileged EXEC prompt.

---

## Management Network Connectivity

Now to make routing between 2 subnets possible, since I need the Docker network inside the `clab` VM to be able to communicate with the Ansible control VM.

The approach I took is to make the `clab` VM act as a router between the physical management network and the Containerlab management network. I began by enabling IP forwarding on the `clab` VM and added a static route on `ansible-ctrl`.

**1. Enable IP forwarding on clab**

Enable it:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo sysctl -w net.ipv4.ip_foward=1
{{< /codeblock >}}

{{< line-explain >}}
This enables Linux kernel's IP forwarding capability.
{{< /line-explain >}}

The made it persistent:

{{< codeblock lang="Bash" syntax="bash" >}}
echo "net.ipv4.ip_foward=1" | sudo tee -a /etc/sysctl.conf
{{< /codeblock >}}

{{< line-explain >}}
Writes the setting to `sysctl.conf` so it survives reboots.
{{< /line-explain >}}

---

**2. Add an iptables rule for NAT**

Docker's default networking applies NAT to outgoing traffic from containers, but it doesn't automatically allow incoming routed traffic to reach containers. I added an iptables rule to allow forwarded traffic to the Containerlab management bridge:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo iptables -I DOCKER-USER -d 172.20.20.0/24 -j ACCEPT
sudo iptables -I DOCKER-USER -s 172.20.20.0/24 -j ACCEPT
{{< /codeblock >}}

{{< line-explain >}}
DOCKER-USER:
: Docker inserts its own iptables rules that can block forwarded traffic. The `DOCKER-USER` chain is specifically designed for user rules that should be evaluated before Docker's own filtering.
{{< /line-explain >}}

Then made it persistent:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
{{< /codeblock >}}

---

**3. Add a static route on ansible-ctrl**

Added the route:

{{< codeblock lang="Bash" syntax="bash" >}}
sudo ip route add 172.20.20.0/24 via 10.33.99.61
{{< /codeblock >}}

{{< line-explain >}}
This tells `ansible-ctrl` that the `172.20.20.0/24` network is reachable through the `clab` VM.
{{< /line-explain >}}

Then made it persistent by adding it to the Netplan configuration:

{{< codeblock file="/etc/netplan/00-installer-config.yaml" syntax="yaml" >}}
      routes:
        - to: 172.20.20.0/24
          via: <clab-ip>
{{< /codeblock >}}

{{< codeblock lang="Bash" syntax="bash" >}}
sudo netplan apply
{{< /codeblock >}}

---

### Verification

I then verified connectivity from the Ansible control node.

**Ping test**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ping -c 3 172.20.20.11
(.venv) $ ping -c 3 172.20.20.12
{{< /codeblock >}}

---

**SSH test**

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ ssh admin@172.20.20.11
{{< /codeblock >}}

{{< codeblock lang="Expected Output" copy="false" >}}
Password:
wan-r1#
{{< /codeblock >}}

---

## Commit & Push

The Containerlab topology and startup configs are infrastructure-as-code so they belong in the Git repo. I copied them to the project directory on `ansible-ctrl` and committed them.

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ cd ~/network-automation-lab
(.venv) $ mkdir -p containerlab/topologies
(.venv) $ mkdir -p containerlab/configs
{{< /codeblock >}}

I used `scp` to transfer the files from the `clab` VM to the project directory:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ scp clab:/opt/clab/topologies/wan.clab.yml containerlab/topologies/
(.venv) $ scp clab:/opt/clab/configs/wan-r1.cfg containerlab/configs/
(.venv) $ scp clab:/opt/clab/configs/wan-r2.cfg containerlab/configs/
{{< /codeblock >}}

Then I committed using the feature branch workflow:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout -b feat/containerlab-wan
(.venv) $ git add -A
(.venv) $ git commit -m "feat: add containerlab topology for wan-r1 and wan-r2"
(.venv) $ git push -u origin feat/containerlab-wan
{{< /codeblock >}}

Then approved on Gitea.

After merging:

{{< codeblock lang="Bash" syntax="bash" >}}
(.venv) $ git checkout main
(.venv) $ git pull origin main
{{< /codeblock >}}

---

Now I have a Containerlab VM with Docker setup.