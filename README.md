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