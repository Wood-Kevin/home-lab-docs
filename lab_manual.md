# Home Lab Infrastructure Runbook & Operations Wiki

## 1. Executive Overview & Architecture
This local, software-defined home lab simulates a multi-node enterprise environment to test automation, infrastructure-as-code, and centralized logging. The entire infrastructure is hosted locally on an Apple Silicon M5 platform using Type-2 virtualization.

### 1.1 Hypervisor Platform
* **Host System:** MacBook Air (Apple M5, 16GB Unified Memory)
* **Hypervisor:** VMware Fusion (ARM64 Native)
* **Base OS Template:** Ubuntu Server 24.04 LTS (ARM64 / aarch64)

---

## 2. Network Topology & IP Allocation

### 2.1 Subnet Architecture
The environment utilizes VMware Fusion's isolated NAT virtual switch interface. The host machine acts as the management gateway and routing engine for all internal VM traffic.
* **Network Type:** VMware Fusion NAT
* **Subnet Range:** `192.168.26.0/24`
* **Netmask:** `255.255.255.0`
* **Gateway IP (Mac Host Virtual Interface):** `192.168.26.1`
* **DHCP Scope:** Dynamic leases assigned starting at `192.168.26.128`

### 2.2 Infrastructure Inventory Map

| Node Name | Role / Function | IP Address | Compute Specs | Primary Services |
| :--- | :--- | :--- | :--- | :--- |
| **macbook-host** | Hypervisor Gateway / Client | `192.168.26.1` | 8 Cores / 16GB | SSH Client, Terminal |
| **node1** | Automation Control Node | `192.168.26.131`| 2 vCPU / 2GB | Ansible, Python, Git |
| **node2** | Target Environment Host | `192.168.26.132`| 2 vCPU / 2GB | Managed Target, UFW |
| **node3** | Central Logging Server  | `192.168.26.133`| 2 vCPU / 2GB | Syslog Receiver, Docker |

---

## 3. Host-to-Guest Client Access Layout
To bypass manual IP address entry, the host Mac's local SSH client profile is mapped directly to the internal lab subnet.

### 3.1 Host Client Profile (~/.ssh/config)
The configuration block located on the MacBook workstation must mirror the following entries to facilitate rapid node access:

Host node1
    HostName 192.168.26.131
    User kwood
    IdentityFile ~/.ssh/id_ed25519

Host node2
    HostName 192.168.26.132
    User kwood
    IdentityFile ~/.ssh/id_ed25519

Host node3
    HostName 192.168.26.133
    User kwood
    IdentityFile ~/.ssh/id_ed25519

---

## 4. Systems Administration & Operations Standards

### 4.1 Hostname Architecture
All hostnames must be lowercase, alphanumeric, and explicitly indicate the node's function and instance number. Cryptic or arbitrary naming conventions are prohibited.
* **Format:** `[function][instance_number]`
* **Examples:** `node1`, `node2`, `logging01`, `db01`

### 4.2 User & Privilege Management
* **Administrative Account:** The primary administrative account across all Linux nodes is `kwood`.
* **Root Access:** Direct SSH login as the `root` user is strictly disabled. 
* **Privilege Escalation:** All administrative tasks must execute via `sudo`. The `sudo` execution policy must require explicit user authentication unless handled by an authorized automation service account.

### 4.3 Configuration Management (The Backup Rule)
Before any native system configuration file (under /etc/ or /var/) is modified manually, an immutable backup copy must be generated in the same directory.
* **Format:** [filename].[YYYYMMDD].bak
* **Example Command:** sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.20260517.bak

---

## 5. Security & Network Baseline

### 5.1 Cryptographic Standards (SSH)
* **Authorized Protocols:** `Ed25519` is the mandatory standard for all SSH key pairs. Legacy RSA or ECDSA keys are deprecated and restricted from the environment.
* **Authentication Policy:** Password-based authentication over SSH must be completely disabled on production-simulated nodes once the initialization key is deployed.

### 5.2 Network Zoning & Firewalls
* **Local Daemon:** The Uncomplicated Firewall (`ufw`) must be active on all Ubuntu nodes.
* **Default Policy:** Deny all incoming traffic, allow all outgoing traffic.
* **Rule Documentation:** Every allowed port must be documented with its explicit business case (e.g., Port `22/tcp` for Management SSH).

---

## 6. Code & Automation Style Guides

### 6.1 YAML / Ansible Standards
Ansible playbooks and variable files must strictly adhere to the standard YAML formatting specifications to prevent syntax compilation failures.
* **Indentation:** Exactly 2 spaces per indentation level. The use of hard tabs (\t) is strictly prohibited.
* **Naming Convention:** Every task block within an Ansible playbook must include a descriptive, lowercase `name:` string explaining its exact function.

### 6.2 Shell Scripting Baseline (Bash)
Any standalone shell scripts executed within the infrastructure must conform to safe, predictable execution patterns.
* **Interpreter Directive:** Every script must begin with `#!/usr/bin/env bash`.
* **Strict Mode:** The shell environment must enforce immediate failure constraints to stop execution upon errors. The following line is mandatory directly below the shebang: `set -euo pipefail`

---

## 7. Verification & Definition of Done (DoD)
A lab project or deployment is not considered complete until it meets the following criteria:
1. **Functional Validation:** The service is actively running and verified via system metrics (`systemctl status` or custom verification script).
2. **Security Validation:** Firewall rules are updated, and unnecessary ports remain closed.
3. **Documentation Sync:** This log manual is updated with any new structural changes or software layers added to the cluster nodes.

---

## 8. Node Initialization & Automation Bootstrapping

### 8.1 SSH Trust Mesh Setup (Execution Path: node1)
To enable passwordless orchestration, Node 1 acts as the central execution authority. The following routine initializes its identity and distributes its public key to the target environment hosts:

# Generate Node 1 local identity
ssh-keygen -t ed25519 -C "ansible-control-node"

# Authorize Node 1 access across the local subnet pool
ssh-copy-id kwood@192.168.26.132
ssh-copy-id kwood@192.168.26.133

### 8.2 Ansible Engine Installation
The automation core is deployed explicitly to the controller node (node1) utilizing native upstream repositories:

sudo apt update && sudo apt install ansible -y

### 8.3 Control Workspace & Inventory Framework
All automation playbooks and orchestration assets live within a unified user workspace directory structure on node1.
* **Workspace Path:** ~/ansible-lab/
* **Inventory Control File:** ~/ansible-lab/hosts

**Inventory Configuration Layout (hosts):**
[targets]
node2 ansible_host=192.168.26.132
node3 ansible_host=192.168.26.133

[targets:vars]
ansible_user=kwood
ansible_ssh_private_key_file=~/.ssh/id_ed25519

### 8.4 Verification Standard (The Ad-Hoc Connectivity Test)
The infrastructure cluster communication is verified using the Ansible ad-hoc native ping module:

ansible targets -i hosts -m ping

## 9. Centralized Logging Infrastructure (rsyslog)

### 9.1 Receiver Configuration (Execution Path: node3)
Node 3 is designated as the centralized log repository, listening on standard port 514 for both UDP and TCP traffic. Logs are dynamically separated into structured directories based on the originating host.

**Configuration Baseline (/etc/rsyslog.conf):**
* UDP/TCP Modules Enabled: imudp, imtcp
* Listening Port: 514
* Storage Template Rule (Appended to bottom of file):
  $template RemoteLogs,"/var/log/nodes/%HOSTNAME%/%PROGRAMNAME%.log"
  *.* ?RemoteLogs
  & stop

**Service Management:**
sudo cp /etc/rsyslog.conf /etc/rsyslog.conf.20260517.bak
sudo systemctl restart rsyslog
sudo ss -tulnp | grep 514

### 9.2 Client Shipper Configuration (Execution Path: node1 & node2)
To forward system events to the centralized receiver, endpoints deploy an explicit forwarding policy within the local drop-in directory structure.

**Configuration Baseline (/etc/rsyslog.d/60-remotelog.conf):**
*.* @@192.168.26.133:514

**Service Lifecycle Activation:**
sudo systemctl restart rsyslog