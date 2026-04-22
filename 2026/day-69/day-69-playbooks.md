# Day 69 – Ansible Playbooks and Modules

## Overview

Ad-hoc commands are useful for quick checks, but real automation lives in **playbooks**. A playbook is a YAML file describing the desired state of your servers. Write it once, run it a hundred times, get the same result every time — that's **idempotency**.

---

## Task 1: Your First Playbook

```yaml
# install-nginx.yml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:                          # use 'apt' for Ubuntu
        name: nginx
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Create a custom index page
      copy:
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /usr/share/nginx/html/index.html
```

```bash
ansible-playbook -i inventory.ini install-nginx.yml

# First run output:
# TASK [Install Nginx]        changed
# TASK [Start Nginx]          changed
# TASK [Create index page]    changed

# Second run output:
# TASK [Install Nginx]        ok
# TASK [Start Nginx]          ok
# TASK [Create index page]    ok

# Verify
curl http://<WEB_SERVER_PUBLIC_IP>
# <h1>Deployed by Ansible - TerraWeek Server</h1>
```

**Idempotency:** On the second run, every task shows `ok` instead of `changed`. Ansible checked the state, found it already matches what was requested, and made zero changes. This is what makes Ansible safe to run repeatedly.

---

## Task 2: Playbook Structure

```yaml
---                                   # YAML document start (required)
- name: Play name                     # PLAY -- a named scenario targeting hosts
  hosts: web                          # which inventory group to run on
  become: true                        # all tasks run as root (sudo)

  tasks:                              # ordered list of tasks
    - name: Task name                 # TASK -- one unit of work
      module_name:                    # MODULE -- what Ansible executes
        key: value                    # module arguments
```

### Key Concepts

| Concept | Definition |
| --- | --- |
| **Play** | A mapping of hosts to tasks; a playbook can contain multiple plays |
| **Task** | A single call to one module; the smallest unit of automation |
| **Module** | Built-in functionality (yum, service, copy, file, etc.) |
| **become: true** at play level | All tasks in the play run with sudo |
| **become: true** at task level | Only that specific task runs with sudo |

**Task failure behavior:** By default, if a task fails on a host, Ansible stops running further tasks on that host but continues on other hosts. Use `ignore_errors: true` on a task to continue despite failure, or `any_errors_fatal: true` on a play to stop everything.

**Multiple plays in one playbook:** Yes. Each play can target different host groups with different tasks. They run sequentially.

---
<img width="1620" height="622" alt="image" src="https://github.com/user-attachments/assets/25e5b2e6-9b0e-44d8-afe9-cf49c262d1f4" />

## Task 3: Essential Modules

```yaml
# essential-modules.yml
---
- name: Essential Ansible Modules Demo
  hosts: all
  become: true

  tasks:
    # --- yum/apt: install packages ---
    - name: Install multiple packages
      yum:
        name:
          - git
          - curl
          - wget
          - tree
        state: present

    # --- service: manage services ---
    - name: Ensure Nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: true

    # --- copy: copy file from control node ---
    - name: Copy config file
      copy:
        src: files/app.conf
        dest: /etc/app.conf
        owner: root
        group: root
        mode: '0644'

    # --- file: create directories / set permissions ---
    - name: Create application directory
      file:
        path: /opt/myapp
        state: directory
        owner: ec2-user
        mode: '0755'

    # --- command: run a command (no shell features) ---
    - name: Check disk space
      command: df -h
      register: disk_output

    - name: Print disk space
      debug:
        var: disk_output.stdout_lines

    # --- shell: run with shell features (pipes, redirects) ---
    - name: Count running processes
      shell: ps aux | wc -l
      register: process_count

    - name: Show process count
      debug:
        msg: "Total processes: {{ process_count.stdout }}"

    # --- lineinfile: add or modify one line in a file ---
    - name: Set timezone in environment
      lineinfile:
        path: /etc/environment
        line: 'TZ=Asia/Kolkata'
        create: true
```

```bash
# Create the files directory and sample config
mkdir -p files
echo "[server]\nport=8080" > files/app.conf

ansible-playbook essential-modules.yml
```

### Module Quick Reference

| Module | Key args | Use for |
| --- | --- | --- |
| `yum` / `apt` | `name`, `state: present/absent/latest` | Package management |
| `service` | `name`, `state: started/stopped/restarted`, `enabled` | Service management |
| `copy` | `src`, `dest`, `owner`, `mode` | Copy files from control node |
| `template` | `src`, `dest` | Copy with Jinja2 variable substitution |
| `file` | `path`, `state: directory/file/absent`, `mode` | Directories and permissions |
| `command` | `cmd` or free-form | Simple commands, no shell |
| `shell` | free-form | Commands needing pipes, redirects |
| `lineinfile` | `path`, `line`, `regexp` | Add/modify one line in a file |
| `debug` | `var` or `msg` | Print variables during playbook run |
| `register` | (used with any task) | Save task output to a variable |

### `command` vs `shell`

|  | `command` | `shell` |  |
| --- | --- | --- | --- |
| Shell features (pipes, `>`, `&&`) | No | Yes |  |
| Security | Safer (no shell injection) | Less safe |  |
| Use for | Simple commands | Commands needing ` | `,` &&`,` >`,` $VAR` |

---

## Task 4: Handlers

Handlers are tasks that only run when **notified** by another task. They run once at the end of all tasks, even if notified multiple times. Perfect for service restarts that should only happen when a config actually changed.

```yaml
# nginx-config.yml
---
- name: Configure Nginx with a custom config
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Deploy Nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart Nginx      # triggers handler IF this task is 'changed'

    - name: Deploy custom index page
      copy:
        content: "<h1>Managed by Ansible</h1><p>Server: {{ inventory_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart Nginx          # name must match notify string exactly
      service:
        name: nginx
        state: restarted
```

```bash
mkdir -p files
cat > files/nginx.conf << 'EOF'
events {}
http {
  server {
    listen 80;
    root /usr/share/nginx/html;
  }
}
EOF

# First run: config file is new -> changed -> handler triggers
ansible-playbook nginx-config.yml
# RUNNING HANDLERS: Restart Nginx  changed

# Second run: config unchanged -> ok -> handler does NOT run
ansible-playbook nginx-config.yml
# (no RUNNING HANDLERS section)
```

**Handler rules:**

- Handlers only run if their notifying task is `changed` (not `ok`)
- Handlers run once at the end, regardless of how many tasks notify them
- Handler name in `notify:` must match exactly

---

## Task 5: Dry Run, Diff, and Verbosity

```bash
# Dry run -- shows what WOULD change without changing anything
ansible-playbook install-nginx.yml --check

# Diff mode -- shows exact file content differences
ansible-playbook nginx-config.yml --check --diff

# Verbosity levels
ansible-playbook install-nginx.yml -v       # task results
ansible-playbook install-nginx.yml -vv      # module args + results
ansible-playbook install-nginx.yml -vvv     # connection debugging
ansible-playbook install-nginx.yml -vvvv    # full plugin debugging

# Limit to specific host
ansible-playbook install-nginx.yml --limit web-server

# Preview without running
ansible-playbook install-nginx.yml --list-hosts
ansible-playbook install-nginx.yml --list-tasks

# Syntax validation
ansible-playbook install-nginx.yml --syntax-check
```

### Why `--check --diff` is Critical for Production

`--check` prevents any changes from being made. `--diff` shows the exact before/after of every file that would be modified. Together they give you a full preview of what a playbook will do — like `terraform plan` but for configuration. Always run `--check --diff` before applying playbooks to production systems.

---

## Task 6: Multiple Plays in One Playbook

```yaml
# multi-play.yml
---
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      yum: { name: nginx, state: present }
    - name: Start Nginx
      service: { name: nginx, state: started, enabled: true }

- name: Configure app servers
  hosts: app
  become: true
  tasks:
    - name: Install build tools
      yum:
        name: [gcc, make]
        state: present
    - name: Create app directory
      file: { path: /opt/app, state: directory, mode: '0755' }

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Install MySQL client
      yum: { name: mysql, state: present }
    - name: Create data directory
      file: { path: /var/lib/appdata, state: directory, mode: '0700' }
```

```bash
ansible-playbook multi-play.yml
# PLAY [Configure web servers]  -- runs only on web group
# PLAY [Configure app servers]  -- runs only on app group
# PLAY [Configure db servers]   -- runs only on db group
```

Each play targets a different host group. Tasks in the web play never run on db servers and vice versa.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Playbook | YAML file of one or more plays; defines desired state |
| Play | Maps hosts to tasks; one playbook can have many plays |
| Task | Single module call; smallest unit of work |
| Idempotency | Running a playbook multiple times gives the same result; only changes what needs changing |
| `become: true` | Run as root (sudo); set at play level (all tasks) or task level (one task) |
| Handler | Runs only when notified by a changed task; runs once at end of play |
| `register` | Saves task output to a variable for later use |
| `debug` | Prints variables or messages during a playbook run |
| `--check` | Dry run; no changes made |
| `--diff` | Shows file content differences |
| `--syntax-check` | Validates YAML before running |

---

## Quick Reference

```bash
# Run playbook
ansible-playbook playbook.yml
ansible-playbook playbook.yml --limit web-server
ansible-playbook playbook.yml --check
ansible-playbook playbook.yml --check --diff
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml --list-hosts
ansible-playbook playbook.yml --list-tasks
```

```yaml
# Task patterns
- name: Install package
  yum: { name: nginx, state: present }

- name: Manage service
  service: { name: nginx, state: started, enabled: true }

- name: Run command and capture output
  command: df -h
  register: result
- debug: { var: result.stdout_lines }

- name: Copy file with permissions
  copy: { src: files/app.conf, dest: /etc/app.conf, owner: root, mode: '0644' }

- name: Create directory
  file: { path: /opt/app, state: directory, mode: '0755' }

- name: Add line to file
  lineinfile: { path: /etc/environment, line: 'KEY=value', create: true }

# Handler pattern
tasks:
  - name: Deploy config
    copy: { src: nginx.conf, dest: /etc/nginx/nginx.conf }
    notify: Restart Nginx

handlers:
  - name: Restart Nginx
    service: { name: nginx, state: restarted }
```
