# Day 68 – Introduction to Ansible and Inventory Setup

## Overview

Terraform provisions infrastructure. But who installs packages, configures services, manages users, and keeps servers in the desired state after they exist? That is **Ansible** — the industry-standard configuration management tool. It is agentless: SSH is all it needs.

---

## Task 1: Understand Ansible

### What is Configuration Management?

Configuration management is the practice of keeping servers in a known, consistent, desired state automatically. Without it, servers drift over time — one has a different package version, another has a config file someone edited manually. Config management tools define the desired state in code and enforce it repeatedly.

### Ansible vs Chef, Puppet, Salt

| Tool | Approach | Agent required? | Language | Learning curve |
| --- | --- | --- | --- | --- |
| **Ansible** | Agentless, push-based | No — uses SSH | YAML | Low |
| **Chef** | Agent-based, pull | Yes | Ruby DSL | High |
| **Puppet** | Agent-based, pull | Yes | Puppet DSL | High |
| **Salt** | Agent-based (optional agentless) | Optional | YAML + Python | Medium |

### What "Agentless" Means

Ansible does not require any software installed on the servers it manages. It connects over SSH, copies a small Python script to the remote machine, executes it, then removes it. The managed nodes only need: SSH access + Python (usually pre-installed).

### Ansible Architecture

```
┌─────────────────────────────────────────┐
│           CONTROL NODE                  │
│  (your laptop or jump server)           │
│                                         │
│  ┌─────────┐  ┌──────────┐  ┌────────┐ │
│  │Inventory│  │Playbooks │  │Modules │ │
│  │(hosts)  │  │(YAML)    │  │(tasks) │ │
│  └─────────┘  └──────────┘  └────────┘ │
└──────────────────┬──────────────────────┘
                   │ SSH
        ┌──────────┼──────────┐
        ▼          ▼          ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │  web-01  │ │  app-01  │ │  db-01   │
  │MANAGED   │ │MANAGED   │ │MANAGED   │
  │NODE      │ │NODE      │ │NODE      │
  └──────────┘ └──────────┘ └──────────┘
```

| Component | What it is |
| --- | --- |
| **Control Node** | Machine where Ansible is installed and runs from |
| **Managed Nodes** | Target servers Ansible configures (no agent needed) |
| **Inventory** | List of managed nodes, grouped and annotated |
| **Modules** | Units of work: `yum`, `copy`, `service`, `user`, `template`, etc. |
| **Playbooks** | YAML files defining what tasks to run on which hosts |

---

## Task 2: Set Up Lab Environment (with Terraform)

```hcl
# main.tf -- provision 3 EC2 instances for Ansible practice
resource "aws_key_pair" "ansible" {
  key_name   = "ansible-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

resource "aws_instance" "web" {
  ami                         = data.aws_ami.amazon_linux.id
  instance_type               = "t2.micro"
  key_name                    = aws_key_pair.ansible.key_name
  vpc_security_group_ids      = [aws_security_group.ssh.id]
  associate_public_ip_address = true
  tags = { Name = "ansible-web" }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.ansible.key_name
  vpc_security_group_ids = [aws_security_group.ssh.id]
  associate_public_ip_address = true
  tags = { Name = "ansible-app" }
}
```

```bash
# Verify SSH access to each instance
ssh -i ~/.ssh/id_rsa ec2-user@<PUBLIC_IP_1>
ssh -i ~/.ssh/id_rsa ec2-user@<PUBLIC_IP_2>

```
<img width="1467" height="698" alt="1" src="https://github.com/user-attachments/assets/b7c71e8c-c9f8-4b2f-9704-0db14e54f1e7" />

<img width="1328" height="735" alt="2" src="https://github.com/user-attachments/assets/b79b12fe-8a5d-41c3-9974-648daa8b74c7" />

## Task 3: Install Ansible

```bash
# macOS
brew install ansible

# Ubuntu/Debian
sudo apt update && sudo apt install ansible -y

# Amazon Linux / RHEL
sudo yum install ansible -y
# or via pip
pip3 install ansible

# Verify
ansible --version
# ansible [core 2.x.x]
#   config file = /etc/ansible/ansible.cfg
#   python version = 3.x.x
```

Ansible is only installed on the **control node** because that is where all the logic runs. It SSHes into managed nodes and executes modules remotely — managed nodes need nothing special beyond SSH + Python.

---

<img width="931" height="573" alt="image" src="https://github.com/user-attachments/assets/697bc4b5-2b04-4b35-a53e-4cfbb6dbdae7" />


---

### Troubleshooting Ping Failures

| Symptom | Fix |
| --- | --- |
| `Permission denied` | Wrong SSH key path or wrong `ansible_user` |
| `Connection timed out` | Security group missing SSH rule from your IP |
| `Host key verification failed` | Set `host_key_checking = False` in `ansible.cfg` |
| Amazon Linux user | `ec2-user` |
| Ubuntu user | `ubuntu` |

```bash
# Fix SSH key permissions if needed
chmod 400 ~/.ssh/id_rsa
```

---

## Task 5: Run Ad-Hoc Commands

Ad-hoc commands run a single module directly from the CLI — no playbook needed.

```bash
# Check uptime on all servers
ansible all -i inventory.ini -m command -a "uptime"

# Check free memory on web servers only
ansible web -i inventory.ini -m command -a "free -h"

# Check disk space on all servers
ansible all -i inventory.ini -m command -a "df -h"

# Install git on web group (requires sudo -- use --become)
ansible web -i inventory.ini -m yum -a "name=git state=present" --become
# Ubuntu: use -m apt instead of -m yum

# Copy a file to all servers
echo "Hello from Ansible" > hello.txt
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"

# Verify the file was copied
ansible all -i inventory.ini -m command -a "cat /tmp/hello.txt"
```
<img width="1237" height="677" alt="image" src="https://github.com/user-attachments/assets/d837db6b-3cfb-4af8-b809-8355d782cf31" />

<img width="1621" height="757" alt="image" src="https://github.com/user-attachments/assets/308349ae-02b7-48ad-8c68-0ae50bf0f952" />


### What `--become` Does

`--become` escalates privileges to root (equivalent to `sudo`) on the managed node. Needed for:

- Installing/removing packages
- Managing system services
- Writing to system directories (`/etc/`, `/var/`, etc.)
- Managing users and groups

Without `--become`, the command runs as `ansible_user` (e.g., `ec2-user`) which has no sudo rights by default.

### `command` vs `shell` Module

| Module | Supports pipes/redirects? | Use for |  |
| --- | --- | --- | --- |
| `command` | No | Simple commands, safer |  |
| `shell` | Yes (` | `,` >`,` &&`) | Commands needing shell features |

---

## Task 6: Inventory Groups and Patterns

```
# inventory.ini -- updated with group of groups
[web]
web-server ansible_host=<PUBLIC_IP_1>

[app]
app-server ansible_host=<PUBLIC_IP_2>

[db]
db-server ansible_host=<PUBLIC_IP_3>

[application:children]
web
app

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

```bash
# Target specific groups
ansible application -i inventory.ini -m ping    # web + app


# Patterns
ansible 'web:app' -i inventory.ini -m ping      # OR: web or app

```

### ansible.cfg — Stop Typing `-i inventory.ini`

```
# ansible.cfg  (in the project directory)
[defaults]
inventory         = inventory.ini
host_key_checking = False
remote_user       = ec2-user
private_key_file  = ~/.ssh/id_rsa
```

```bash
# Now you can drop the -i flag entirely
ansible all -m ping
ansible web -m command -a "uptime"
```

**Config file precedence (first found wins):**

1. `./ansible.cfg` (current directory)
2. `~/.ansible.cfg` (user home)
3. `/etc/ansible/ansible.cfg` (system-wide)

---
<img width="1172" height="460" alt="image" src="https://github.com/user-attachments/assets/3a80aa81-5494-4d69-8cdc-51df2dff397b" />

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Agentless | No software installed on managed nodes; uses SSH + Python |
| Control node | Where Ansible is installed; runs all playbooks and ad-hoc commands |
| Inventory | List of managed hosts; grouped by role (`[web]`, `[db]`, etc.) |
| `[group:children]` | Group of groups for hierarchical targeting |
| Ad-hoc command | One-off module execution from CLI; `-m module -a args` |
| `--become` | Escalate to root (sudo); required for packages, services, system files |
| `ansible.cfg` | Per-project config; avoids repeating `-i` and other flags |
| `command` module | Default module; simple commands; no shell features |
| `shell` module | Supports pipes, redirects, and shell constructs |
| `ping` module | Tests SSH connectivity and Python; not ICMP ping |

---

## Quick Reference

```bash
# Install
brew install ansible          # macOS
sudo apt install ansible -y   # Ubuntu

# Connectivity test
ansible all -m ping
ansible all -i inventory.ini -m ping

# Ad-hoc commands
ansible all -m command -a "uptime"
ansible web -m yum -a "name=git state=present" --become
ansible all -m copy -a "src=file.txt dest=/tmp/file.txt"
ansible all -m shell -a "ps aux | grep nginx"

# Patterns
ansible 'web:app' -m ping      # OR
ansible 'all:!db' -m ping      # NOT
ansible application -m ping    # group of groups

# Check config
ansible --version              # shows config file path
ansible-inventory --list       # show inventory as JSON
ansible-inventory --graph      # show inventory as tree
```

```
# inventory.ini skeleton
[web]
web1 ansible_host=1.2.3.4

[db]
db1 ansible_host=5.6.7.8

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/key.pem
```
