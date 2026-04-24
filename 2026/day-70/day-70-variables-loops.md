# Day 70 – Variables, Facts, Conditionals and Loops

## Overview

Your playbooks work, but they are static. Real infrastructure is not — web servers need Nginx, app servers need Node.js, production gets more memory than dev. **Variables, facts, conditionals, and loops** turn a rigid script into flexible automation that adapts to each host, group, and environment.

---

## Task 1: Variables in Playbooks

```yaml
# variables-demo.yml
---
- name: Variable demo
  hosts: all
  become: true

  vars:
    app_name: terraweek-app
    app_port: 8080
    app_dir: "/opt/{{ app_name }}"
    packages:
      - git
      - curl
      - wget

  tasks:
    - name: Print app details
      debug:
        msg: "Deploying {{ app_name }} on port {{ app_port }} to {{ app_dir }}"

    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Install required packages
      yum:
        name: "{{ packages }}"
        state: present
```

```bash
ansible-playbook variables-demo.yml

# Override from CLI -- -e always wins
ansible-playbook variables-demo.yml -e "app_name=my-custom-app app_port=9090"
```

CLI variables (`-e`) override playbook `vars`. This is intentional — `-e` is the highest priority override.

---
<img width="1658" height="697" alt="image" src="https://github.com/user-attachments/assets/16044bcb-77e6-41e4-a272-829b735e6cff" />

## Task 2: group_vars and host_vars

Variables should live in dedicated files, not inside playbooks. Ansible automatically loads these based on the inventory group or host name.

```
ansible-practice/
├── inventory.ini
├── ansible.cfg
├── group_vars/
│   ├── all.yml          # applies to every host
│   ├── web.yml          # applies only to [web] group
│   └── db.yml           # applies only to [db] group
├── host_vars/
│   └── web-server.yml   # applies only to this specific host
└── playbooks/
    └── site.yml
```

```yaml
# group_vars/all.yml
---
ntp_server: pool.ntp.org
app_env: development
common_packages:
  - vim
  - htop
  - tree
```

```yaml
# group_vars/web.yml
---
http_port: 80
max_connections: 1000
web_packages:
  - nginx
```

```yaml
# group_vars/db.yml
---
db_port: 3306
db_packages:
  - mysql-server
```

```yaml
# host_vars/web-server.yml
---
max_connections: 2000       # overrides group_vars/web.yml value
custom_message: "This is the primary web server"
```

```yaml
# playbooks/site.yml
---
- name: Apply common config
  hosts: all
  become: true
  tasks:
    - name: Install common packages
      yum:
        name: "{{ common_packages }}"
        state: present
    - name: Show environment
      debug:
        msg: "Environment: {{ app_env }}"

- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Show web config
      debug:
        msg: "HTTP port: {{ http_port }}, Max connections: {{ max_connections }}"
    - name: Show host-specific message
      debug:
        msg: "{{ custom_message }}"
```

### Variable Precedence (low → high)

| Priority | Source |
| --- | --- |
| 1 (lowest) | Role defaults |
| 2 | `group_vars/all.yml` |
| 3 | `group_vars/<group>.yml` |
| 4 | `host_vars/<host>.yml` |
| 5 | Playbook `vars:` block |
| 6 | Task `vars:` |
| 7 (highest) | `-e` / `--extra-vars` CLI flag |

`host_vars` beats `group_vars`. `-e` beats everything.

---

## Task 3: Ansible Facts

Ansible automatically collects hundreds of facts about each managed node before running tasks.

```bash
# See ALL facts for a host
ansible web-server -m setup

# Filter specific facts
ansible web-server -m setup -a "filter=ansible_os_family"
ansible web-server -m setup -a "filter=ansible_distribution*"
ansible web-server -m setup -a "filter=ansible_memtotal_mb"
ansible web-server -m setup -a "filter=ansible_default_ipv4"
```
<img width="1606" height="748" alt="image" src="https://github.com/user-attachments/assets/f4f24963-a804-4815-876d-417ae35721b2" />
<img width="1640" height="775" alt="image" src="https://github.com/user-attachments/assets/621f85f4-097c-4bba-b4b5-e9948f947ce8" />

```yaml
# facts-demo.yml
---
- name: Facts demo
  hosts: all
  tasks:
    - name: Show OS and system info
      debug:
        msg: >
          Hostname: {{ ansible_hostname }},
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }},
          RAM: {{ ansible_memtotal_mb }}MB,
          IP: {{ ansible_default_ipv4.address }}

    - name: Show all network interfaces
      debug:
        var: ansible_interfaces
```

### Five Facts to Use in Real Playbooks

| Fact | Value | Real use |
| --- | --- | --- |
| `ansible_distribution` | `Amazon`, `Ubuntu`, `CentOS` | Conditional package manager selection (yum vs apt) |
| `ansible_memtotal_mb` | `985` | Tune JVM heap size or warn on low-memory hosts |
| `ansible_default_ipv4.address` | `10.0.1.5` | Write config files with the correct bind address |
| `ansible_hostname` | `web-server` | Name log files, config files, reports per host |
| `ansible_processor_vcpus` | `2` | Set worker process count based on CPU count |

---

## Task 4: Conditionals with `when`

```yaml
# conditional-demo.yml
---
- name: Conditional tasks demo
  hosts: all
  become: true

  tasks:
    - name: Install Nginx (only on web servers)
      yum:
        name: nginx
        state: present
      when: "'web' in group_names"

    - name: Install MySQL (only on db servers)
      yum:
        name: mysql-server
        state: present
      when: "'db' in group_names"

    - name: Warn on low memory
      debug:
        msg: "WARNING: This host has less than 1GB RAM"
      when: ansible_memtotal_mb < 1024

    - name: Run only on Amazon Linux
      debug:
        msg: "This is an Amazon Linux machine"
      when: ansible_distribution == "Amazon"

    - name: Run only on Ubuntu
      debug:
        msg: "This is an Ubuntu machine"
      when: ansible_distribution == "Ubuntu"

    - name: Run only in production
      debug:
        msg: "Production settings applied"
      when: app_env == "production"

    - name: Multiple conditions (AND -- both must be true)
      debug:
        msg: "Web server with enough memory"
      when:
        - "'web' in group_names"
        - ansible_memtotal_mb >= 512

    - name: OR condition
      debug:
        msg: "Either web or app server"
      when: "'web' in group_names or 'app' in group_names"
```

### `when` Rules

- No `{{ }}` needed in `when` — variables are referenced directly
- List syntax = AND (all conditions must be true)
- Use `or` inline for OR logic
- Skipped tasks show `skipping: [hostname]` in output
- `group_names` is a built-in variable listing all groups the current host belongs to

---

## Task 5: Loops

```yaml
# loops-demo.yml
---
- name: Loops demo
  hosts: all
  become: true

  vars:
    users:
      - name: deploy
        groups: wheel
      - name: monitor
        groups: wheel
      - name: appuser
        groups: users

    directories:
      - /opt/app/logs
      - /opt/app/config
      - /opt/app/data
      - /opt/app/tmp

  tasks:
    - name: Create multiple users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop: "{{ users }}"

    - name: Create multiple directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ directories }}"

    - name: Install multiple packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - unzip
        - jq

    - name: Print each user created
      debug:
        msg: "Created user {{ item.name }} in group {{ item.groups }}"
      loop: "{{ users }}"
```

### `loop` vs `with_items`

|  | `loop` | `with_items` |
| --- | --- | --- |
| Status | Modern — recommended | Legacy — still works but deprecated |
| Syntax | `loop: [...]` | `with_items: [...]` |
| Item reference | `{{ item }}` | `{{ item }}` |
| Nested loops | Use `include_tasks` | `with_nested` |

Use `loop` for all new playbooks. `with_items` works identically but is not the future direction.

---

## Task 6: Combine Everything — Server Health Report

```yaml
# server-report.yml
---
- name: Server Health Report
  hosts: all

  tasks:
    - name: Check disk space
      command: df -h /
      register: disk_result

    - name: Check memory
      command: free -m
      register: memory_result

    - name: Check running services
      shell: systemctl list-units --type=service --state=running | head -20
      register: services_result

    - name: Generate report
      debug:
        msg:
          - "========== {{ inventory_hostname }} =========="
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "IP: {{ ansible_default_ipv4.address }}"
          - "RAM: {{ ansible_memtotal_mb }}MB"
          - "Disk: {{ disk_result.stdout_lines[1] }}"
          - "Running services: {{ services_result.stdout_lines | length }}"

    - name: Flag critically low disk
      debug:
        msg: "ALERT: Check disk space on {{ inventory_hostname }}"
      when: "'9' in disk_result.stdout and '%' in disk_result.stdout"

    - name: Save report to file
      copy:
        content: |
          Server: {{ inventory_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          RAM: {{ ansible_memtotal_mb }}MB
          Disk: {{ disk_result.stdout }}
          Checked at: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/server-report-{{ inventory_hostname }}.txt"
      become: true
```

```bash
ansible-playbook server-report.yml

# Verify on the server
cat /tmp/server-report-web-server.txt
```

---
<img width="1652" height="500" alt="image" src="https://github.com/user-attachments/assets/d7e5c52d-9c6d-49d7-841c-c1f38e4e0a08" />

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `vars:` in playbook | Defined inside the play; overridden by host_vars, -e |
| `group_vars/` | Auto-loaded per group; filename must match group name |
| `host_vars/` | Auto-loaded per host; overrides group_vars |
| `-e` flag | Highest priority; overrides everything |
| Facts | Auto-collected system info; `ansible_distribution`, `ansible_memtotal_mb`, etc. |
| `when:` | Conditional execution; no `{{ }}` needed; list = AND logic |
| `group_names` | Built-in list of all groups the current host belongs to |
| `loop:` | Modern loop syntax; `item` references the current element |
| `register:` | Saves task output (stdout, stderr, rc, stdout_lines) to a variable |
| `debug:` | Prints `var` (raw) or `msg` (formatted) during playbook run |

---

## Quick Reference

```bash
# See all facts
ansible <host> -m setup
ansible <host> -m setup -a "filter=ansible_distribution*"

# Override variables
ansible-playbook playbook.yml -e "key=value key2=value2"
```

```yaml
# Variable sources
vars:
  key: value                         # inline
  path: "/opt/{{ app_name }}"        # interpolation

# Conditional
when: ansible_distribution == "Ubuntu"
when: "'web' in group_names"
when:                                # AND
  - condition_one
  - condition_two
when: cond_one or cond_two           # OR

# Loop
loop:
  - item1
  - item2
# access with {{ item }}

# Loop over dicts
loop: "{{ users }}"
# access with {{ item.name }}, {{ item.groups }}

# Register + use
- command: df -h
  register: result
- debug: { var: result.stdout_lines }
```
