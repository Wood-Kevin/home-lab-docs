# Engineering Home Lab Documentation

## Overview
This local, software-defined home lab simulates a multi-node enterprise environment to test automation, infrastructure-as-code, and centralized logging. The entire infrastructure is hosted locally on an Apple Silicon M5 platform using Type-2 virtualization.

## Platform Architecture
* **Host System:** MacBook Air (Apple M5, 16GB Unified Memory)
* **Hypervisor:** VMware Fusion (ARM64 Native)
* **Base OS Template:** Ubuntu Server 24.04 LTS (ARM64 / aarch64)

---

## Network Architecture & Topology
The environment utilizes VMware Fusion's isolated NAT virtual switch interface. The host machine acts as the management gateway.

| Node Name | Role | IP Address | Compute Specs | Primary Services |
| :--- | :--- | :--- | :--- | :--- |
| **macbook-host** | Gateway / Workstation | `192.168.26.1` | 8 Cores / 16GB | SSH Client, Terminal |
| **node1** | Automation Control Node | `192.168.26.131`| 2 vCPU / 2GB | Ansible, Python, Git |
| **node2** | Target Environment Host | `192.168.26.132`| 2 vCPU / 2GB | Managed Node, UFW |
| **node3** | Central Logging Server  | `192.168.26.133`| 2 vCPU / 2GB | Syslog Receiver, Docker |

---

## Core Access & Security
* **Authentication Method:** Key-based SSH (`Ed25519` protocol only). Passwords disabled for administrative SSH access.
* **Master Private Key Location:** `~/.ssh/id_ed25519` (Host Machine)
* **Administrative Account:** `kwood` (Sudoers privileges active)