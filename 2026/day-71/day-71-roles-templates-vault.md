# Day 71 – Roles, Galaxy, Templates and Vault

## Overview

Playbooks grow big fast. Tasks, variables, handlers, files — all in one file. **Roles** give them structure. **Templates** make config files dynamic. **Galaxy** gives you community automation. **Vault** keeps your secrets safe. Today you learn all four.

---

## Task 1: Jinja2 Templates

Templates generate config files dynamically from variables and facts. They use `.j2` extension (Jinja2).

```
{# templates/nginx-vhost.conf.j2 #}
# Managed by Ansible -- do not edit manually
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

```yaml
# template-demo.yml
---
- name: Deploy Nginx with template
  hosts: web
  become: true
  vars:
    app_name: terraweek-app
    http_port: 80

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Create web root
      file:
        path: "/var/www/{{ app_name }}"
        state: directory
        mode: '0755'

    - name: Deploy vhost config from template
      template:
        src: templates/nginx-vhost.conf.j2
        dest: "/etc/nginx/conf.d/{{ app_name }}.conf"
        owner: root
        mode: '0644'
      notify: Restart Nginx

    - name: Deploy index page
      copy:
        content: "<h1>{{ app_name }}</h1><p>Host: {{ ansible_hostname }} | IP: {{ ansible_default_ipv4.address }}</p>"
        dest: "/var/www/{{ app_name }}/index.html"

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

```bash
# Run with --diff to see rendered template
ansible-playbook template-demo.yml --diff

# Verify on the server
ssh ec2-user@<IP> cat /etc/nginx/conf.d/terraweek-app.conf
```

### Jinja2 Template Syntax

| Syntax | Purpose |
| --- | --- |
| `{{ variable }}` | Render a variable value |
| `{% if condition %}` | Conditional block |
| `{% for item in list %}` | Loop block |
| `{{ var \ | default('val') }}` |
| `{{ var \ | upper }}` |
| `{# comment #}` | Template comment (not rendered) |

---

## Task 2: Role Structure

```bash
# Generate role skeleton
ansible-galaxy init roles/webserver
```

```
roles/webserver/
├── tasks/
│   └── main.yml         # Main task list -- loaded automatically
├── handlers/
│   └── main.yml         # Handlers
├── templates/
│   └── nginx.conf.j2    # Jinja2 templates
├── files/
│   └── index.html       # Static files (no templating)
├── vars/
│   └── main.yml         # High-priority variables
├── defaults/
│   └── main.yml         # Low-priority default variables
├── meta/
│   └── main.yml         # Role metadata and dependencies
└── README.md
```

### `defaults/` vs `vars/`

|  | `defaults/main.yml` | `vars/main.yml` |
| --- | --- | --- |
| Priority | Lowest — easily overridden | High — hard to override |
| Purpose | Sensible defaults the caller can change | Values that should not change |
| Use for | `http_port: 80`, `app_name: myapp` | Internal role constants |

Rule of thumb: put almost everything in `defaults/`. Only put truly internal, non-overridable values in `vars/`.

---

## Task 3: Build a Custom Webserver Role

```yaml
# roles/webserver/defaults/main.yml
---
http_port: 80
app_name: myapp
max_connections: 512
```

```yaml
# roles/webserver/tasks/main.yml
---
- name: Install Nginx
  yum:
    name: nginx
    state: present

- name: Deploy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
  notify: Restart Nginx

- name: Deploy vhost config
  template:
    src: vhost.conf.j2
    dest: "/etc/nginx/conf.d/{{ app_name }}.conf"
    owner: root
    mode: '0644'
  notify: Restart Nginx

- name: Create web root
  file:
    path: "/var/www/{{ app_name }}"
    state: directory
    mode: '0755'

- name: Deploy index page
  template:
    src: index.html.j2
    dest: "/var/www/{{ app_name }}/index.html"
    mode: '0644'

- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# roles/webserver/handlers/main.yml
---
- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

```
{# roles/webserver/templates/index.html.j2 #}
<h1>{{ app_name }}</h1>
<p>Server: {{ ansible_hostname }}</p>
<p>IP: {{ ansible_default_ipv4.address }}</p>
<p>Environment: {{ app_env | default('development') }}</p>
<p>Managed by Ansible</p>
```
```
{# roles/webserver/templates/nginx.conf.j2 #}
# Managed by Ansible -- do not edit manually
user nginx;
worker_processes {{ ansible_processor_vcpus | default(1) }};
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections {{ max_connections | default(512) }};
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    sendfile        on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```


```
{# roles/webserver/templates/vhost.conf.j2 #}
server {
    listen {{ http_port }};
    server_name {{ ansible_hostname }};
    root /var/www/{{ app_name }};
    index index.html;
    location / { try_files $uri $uri/ =404; }
}
```

```yaml
# site.yml -- calling the role
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80
```

```bash
ansible-playbook site.yml
curl http://<WEB_SERVER_IP>
```

---
<img width="1650" height="708" alt="image" src="https://github.com/user-attachments/assets/2d2629db-27fd-45a8-a214-720ce9407495" />

<img width="917" height="138" alt="image" src="https://github.com/user-attachments/assets/b5d0ba0c-92c2-4f91-a762-7ab8cf47930e" />

## Task 4: Ansible Galaxy

Galaxy is a marketplace of pre-built, community-tested roles.

```bash
# Search
ansible-galaxy search nginx --platforms EL
ansible-galaxy search mysql

# Install a role
ansible-galaxy install geerlingguy.docker

# See installed roles
ansible-galaxy list

# Use it in a playbook
```

```yaml
# docker-setup.yml
---
- name: Install Docker using Galaxy role
  hosts: app
  become: true
  roles:
    - geerlingguy.docker
```

```bash
ansible-playbook docker-setup.yml
```

### requirements.yml — Manage Multiple Roles

```yaml
# requirements.yml
---
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
  - name: geerlingguy.ntp
    version: "2.3.3"
```

```bash
# Install all roles at once
ansible-galaxy install -r requirements.yml
```

**Why `requirements.yml` instead of manual installs:**

- Pin exact versions so upgrades don't break your playbooks
- One command installs everything on a new machine or in CI
- Committed to Git — team always uses the same role versions
- Works with `ansible-galaxy collection install -r requirements.yml` for collections too

---

## Task 5: Ansible Vault

Never store passwords, API keys, or tokens in plain text. Vault encrypts them at rest.

```bash
# Create an encrypted file
ansible-vault create group_vars/db/vault.yml
# Prompts for password, opens editor
```

```yaml
# Inside vault.yml (before encryption -- what you type in the editor)
vault_db_password: SuperSecretP@ssw0rd
vault_db_root_password: R00tP@ssw0rd123
vault_api_key: sk-abc123xyz789
```

```bash
# After saving -- file is fully encrypted on disk
cat group_vars/db/vault.yml
# $ANSIBLE_VAULT;1.1;AES256
# 3262613934363535...

# Vault operations
ansible-vault edit group_vars/db/vault.yml     # edit encrypted file
ansible-vault view group_vars/db/vault.yml     # view without editing
ansible-vault encrypt group_vars/db/secrets.yml  # encrypt existing file
ansible-vault decrypt group_vars/db/vault.yml  # decrypt to plain text
ansible-vault rekey group_vars/db/vault.yml    # change password

# Encrypt a single value inline
ansible-vault encrypt_string 'MyPassword' --name 'db_password'
```

```yaml
# db-setup.yml
---
- name: Configure database
  hosts: db
  become: true
  tasks:
    - name: Confirm DB password is set
      debug:
        msg: "DB password is set: {{ vault_db_password | length > 0 }}"
```

```bash
# Run with vault password prompt
ansible-playbook db-setup.yml --ask-vault-pass

# Better: use a password file
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore

ansible-playbook db-setup.yml --vault-password-file .vault_pass
```

```
# ansible.cfg -- avoid typing --vault-password-file every time
[defaults]
vault_password_file = .vault_pass
```

**Why `--vault-password-file` beats `--ask-vault-pass` for pipelines:** CI/CD pipelines are non-interactive — there is no terminal to type a password into. A password file (or env var `ANSIBLE_VAULT_PASSWORD_FILE`) lets pipelines decrypt secrets without human input. Store the vault password in your CI/CD secret store (GitHub Actions Secret, GitLab CI variable, etc.) and write it to the file at pipeline start.

---

## Task 6: Combine Everything

```yaml
# site.yml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80

- name: Configure app servers with Docker
  hosts: app
  become: true
  roles:
    - geerlingguy.docker

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Deploy DB config with vault secrets
      template:
        src: templates/db-config.j2
        dest: /etc/db-config.env
        owner: root
        mode: '0600'          # secrets file -- only root can read
```

```
{# templates/db-config.j2 #}
# Database Configuration -- Managed by Ansible
DB_HOST={{ ansible_default_ipv4.address }}
DB_PORT={{ db_port | default(3306) }}
DB_PASSWORD={{ vault_db_password }}
DB_ROOT_PASSWORD={{ vault_db_root_password }}
```

```bash
ansible-playbook site.yml

# Verify db config on server
ssh ec2-user@<DB_IP> sudo cat /etc/db-config.env
# Check permission
ssh ec2-user@<DB_IP> ls -la /etc/db-config.env
# -rw------- 1 root root ... /etc/db-config.env
```

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Jinja2 template | `.j2` file with `{{ var }}`, `{% if %}`, `{% for %}` syntax |
| `template` module | Renders `.j2` file with variables and copies to target |
| `\ | default(val)` |
| Role | Fixed directory structure; reusable, self-contained automation unit |
| `defaults/main.yml` | Lowest priority; easily overridden by callers |
| `vars/main.yml` | High priority; not easily overridden |
| Ansible Galaxy | Community marketplace for roles and collections |
| `requirements.yml` | Pin and install multiple roles/collections at once |
| Ansible Vault | Encrypts files and strings at rest; decrypted transparently at runtime |
| `--vault-password-file` | Non-interactive vault decryption for CI/CD pipelines |

---

## Quick Reference

```bash
# Roles
ansible-galaxy init roles/myrole
ansible-galaxy install geerlingguy.docker
ansible-galaxy install -r requirements.yml
ansible-galaxy list

# Vault
ansible-vault create file.yml
ansible-vault edit file.yml
ansible-vault view file.yml
ansible-vault encrypt file.yml
ansible-vault decrypt file.yml
ansible-vault rekey file.yml
ansible-vault encrypt_string 'secret' --name 'var_name'

# Run with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

```yaml
# Call a role
roles:
  - role: webserver
    vars:
      app_name: myapp

# Template task
- template:
    src: config.j2
    dest: /etc/config.conf
    owner: root
    mode: '0644'
  notify: Restart service
```
