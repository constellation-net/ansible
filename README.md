# Constellation
This an Ansible Playbook and Inventory all in one. This repository provides the means to automatically configure all of Constellation's nodes in a consistent manner.

## What this does
The idea behind this repo is that it's a fire-and-forget way to quickly deploy Constellation with various security and cosmetic settings preconfigured for convenience. The full list is here:
- Sets a consistent MOTD shown at login, featuring figlet and neofetch
- Sets the hostname of the machine
- Configures SSH settings, such as disabling password login
- Disables root login and sets an informative nologin prompt
- Overclocks Raspberry Pis (if configured) and sets a boot config option needed for running K3s
- Adds any external drives to fstab so that they mount on boot
- Installs any vendor software for cases, e.g. Argon40 and Pironman5 fan controllers
- Installs K3s on each node
- Initalises the cluster and joins each server to the first master
- Joins agents to the cluster
- Adds any node-specific labels and taints that're used to identify what a node can do
- Copies the cluster's kubeconfig to the local machine for access without using SSH
- Installs Flux on the cluster and bootstraps it to the manifest repository

## What this doesn't do
There are some things that are out-of-scope for this Ansible playbook. In particular, these are:
- Formatting drives
- Installing ZFS and creating pools
- Managing RAID and RAIDZ arrays
- Configuring users and SSH keys

Typically, the reason for this is that it's difficult to make indempotent, or may cause data loss. This means almost anything related to drives (beyond simply mounting at boot) is immediately off the table. Separate Ansible playbooks that need to be deliberately ran could take over these jobs instead.

## Node Labels & Taints

### Labels
The cluster uses labels to attract certain pods to nodes which have a resource they need. They all use the key `affinity.node.cluster.starsystem.dev`, and there are several labels respected so far:
- `camera`: this node has a camera attached to it, which attracted pods can stream over the network
- `database`: indicates the node has a drive/partition available using the `database` storage class for storing db data, so the node can run database servers
- `games`: the node's resources can be used for running game servers as well as related pods (e.g. routers, plugins, monitors, etc)
- `minio`: indicates this node has a large amount of storage available for running a MinIO server that will be used for storing local backups
- `nfs`: the node has been allocated for running the internal NFS server & provisioner that provides remote storage to pods via the `nfs` storage class
- `prometheus`: means the node can be used for running a Prometheus replica, with `local` storage available
- `rtc`: means this node has a dedicated Real Time Clock (RTC) and can be used for running a local NTP server

### Taints
Taints are the opposite to labels in this cluster; they repel pods. Usually, a taint tells Kubernetes that a node lacks a certain resource (e.g. networking, storage, etc). However, no taint exists for lacking compute resources (i.e. CPU and RAM), as the K8s scheduler is already aware of these resources and what is required by each pod. The key for all taints is `taint.node.cluster.starsystem.dev`:
- `device` - NoSchedule, NoExecute: this node is an IoT device with limited resources, which need to be reserved for their special software (e.g. camera streams)
- `games` - PreferNoSchedule: non-game related pods will kept away from this node unless there are no other nodes available, helps keep resources reserved for the game servers if needed
- `hypervisor` - NoSchedule, NoExecute: prevents certain monitoring pods from running on here, which could report the machine's resources twice (due to the VMs it runs also being reported)
- `nas` - PreferNoSchedule: similar to `games`, this keeps resources reserved for a sudden spike in usage where necessary
- `wireless-net` - PreferNoSchedule: pods that don't tolerate this (i.e. those relying on a good network connection), will be repelled where possible and scheduled on a node with a better network connection

## About the Nodes
Currently, Constellation has 7 nodes in total, comprised of:
- 5 physical machines
- of which 4 are Raspberry Pis
- and the remaining 1 is a Proxmox hypervisor
- running 2 virtual machines

Each node's name is based on some sort of celestial object or concept, to fit within Constellation's theme. While I tried to be logical with my naming, try not to dig *too* deeply into it - I just wanted names that actually fit the theme.

### Sol
Sol has a small amount of storage available, as well as its own battery-powered Real Time Clock, gigabit ethernet, and a good amount of compute power. This makes it the most versatile node in the cluster, and can be used for any workloads that don't have specific requirements. It pairs with Luna, the only other node to have no taints. Sol is named after our Sun, as it is the most powerful Raspberry Pi in the cluster. 

**Model**: Raspberry Pi 5
**Operating System**: Raspberry Pi OS Lite Bookworm (64-bit)
**CPU**: 4 cores @ 3.0 GHz (overclocked)
**RAM**: 8 GB
**Storage**: 256 GB NVMe SSD
**Node labels**:
- `prometheus`
- `rtc`
**Node taints**: None

### Luna
This node has been designated to run all of the database servers needed by Constellation. It also runs the NFS server that is provisioned to pods that need a small amount of storage, but that don't need to be pinned to a node. Luna is named after Earth's moon, creating a Sun & Moon analogy with Sol.

**Model**: Raspberry Pi 4b
**Operating System**: Raspberry Pi OS Lite Bookworm (64-bit)
**CPU**: 4 cores @ 1.5 GHz
**RAM**: 2 GB
**Storage**: 256 GB SSD & 128 GB SSD in a ZFS Stripe
**Node labels**:
- `database`
- `nfs`
- `prometheus`
**Node taints**: None

### Voyager
Voyager is the first IoT device in Constellation. It currently serves as an IP camera and was named after the [NASA satellite programme](https://en.wikipedia.org/wiki/Voyager_program) due to being small in power compared to other nodes.

**Model**: Raspberry Pi Zero 2W
**Operating System**: Raspberry Pi OS Lite Bookworm (64-bit)
**CPU**: 4 cores @ 1.0 GHz
**RAM**: 512 MB
**Storage**: None
**Node labels**:
- `camera`
**Node taints**:
- `device`
- `wireless-net`

### Nebula
This node is the first hypervisor in Constellation. It runs Proxmox on the host, but doesn't do much besides the VMs it runs. It was named after a [giant cloud of gas](https://en.wikipedia.org/wiki/Nebula), with each star formed inside being a virtual machine.

**Model**: 
**Operating System**: Proxmox VE
**CPU**: 6 cores @ 4.1 GHz (Intel i8-8500)
**RAM**: 16 GB @ 2666 MHz (Crucial Pro DDR4)
**Storage**: 500 GB SSD (VM operating systems)
**Node labels**: None
**Node taints**:
- `hypervisor`

### Clover
This node serves as Constellation's dedicated gaming node. It has a lot of available resources, which are reserved for use by game servers. Clover runs on Nebula as a virtual machine.

Clover is named after the [quasar](https://en.wikipedia.org/wiki/Quasar) called [Cloverleaf](https://en.wikipedia.org/wiki/Cloverleaf_quasar). I picked the name because the image we have of it shows the quasar split into four different parts - leaves - by [gravitational lensing](https://en.wikipedia.org/wiki/Gravitational_lens), where each leaf could represent a different game server running on the node.

**Operating System**: Debian 12
**CPU**: 4 cores @ 4.1 GHz
**RAM**: 16GB
**Storage**: 1 TB NVMe SSD
**Node labels**: 
- `games`
**Node taints**:
- `games`

### Sagittarius
Sagittarius is one of the VMs running under Nebula. It serves as a NAS and backup server for Constellation, where NAS files are served via Samba and backups are managed by a MinIO server. Sagittarius was named after the supermassive Black Hole [Sagittarius A*](https://en.wikipedia.org/wiki/Sagittarius_A*), due to the huge storage capacity of the node.

**Operating System**: TrueNAS Scale
**CPU**: 2 cores @ 4.1 GHz
**RAM**: 8 GB
**Storage**: 4x WD Red 4TB HDDs in RAIDZ1 (NAS), WD Red 4TB HDD (local backups)
**Node labels**:
- `minio`
**Node taints**:
- `nas`