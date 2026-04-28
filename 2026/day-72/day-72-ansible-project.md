# Day 72 – Ansible Project: Automate Docker and Nginx Deployment

## Overview

Five days of Ansible come together in one real project. Install Docker, pull and run a containerized app, configure Nginx as a reverse proxy, encrypt credentials with Vault — all in one command. This is what you actually do on the job.

---

## Project Structure

```
ansible-docker-project/
├── ansible.cfg
├── inventory.ini
├── site.yml                        # Master playbook
├── .vault_pass                     # gitignored
├── .gitignore
├── group_vars/
│   ├── all.yml                     # Common variables
│   └── web/
│       ├── vars.yml                # Nginx variables
│       └── vault.yml               # Encrypted Docker Hub credentials
└── roles/
    ├── common/                     # Baseline setup for all servers
    │   └── tasks/main.yml
    ├── docker/                     # Docker install + container management
    │   ├── tasks/main.yml
    │   ├── templates/docker-compose.yml.j2
    │   ├── handlers/main.yml
    │   └── defaults/main.yml
    └── nginx/                      # Nginx reverse proxy
        ├── tasks/main.yml
        ├── templates/
        │   ├── nginx.conf.j2
        │   └── app-proxy.conf.j2
        ├── handlers/main.yml
        └── defaults/main.yml
```

```bash
# Generate role skeletons
mkdir -p ansible-docker-project/roles && cd ansible-docker-project
ansible-galaxy init roles/common
ansible-galaxy init roles/docker
ansible-galaxy init roles/nginx

# Install required collection
ansible-galaxy collection install community.docker
```

```
# ansible.cfg
[defaults]
inventory         = inventory.ini
host_key_checking = False
vault_password_file = .vault_pass
```

---

## Task 2: Common Role

```yaml
# group_vars/all.yml
---
timezone: Asia/Kolkata
project_name: devops-app
app_env: development
common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - tree
  - jq
  - unzip
```

```yaml
# roles/common/tasks/main.yml
---
- name: Update package cache
  yum:
    update_cache: true
  tags: common

- name: Install common packages
  yum:
    name: "{{ common_packages }}"
    state: present
  tags: common

- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
  tags: common

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  tags: common

- name: Create deploy user
  user:
    name: deploy
    groups: wheel
    shell: /bin/bash
    state: present
  tags: common
```

---

## Task 3: Docker Role

```yaml
# roles/docker/defaults/main.yml
---
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

```yaml

# tasks file for roles/docker

# roles/docker/tasks/main.yml
---
- name: Install Docker on Amazon Linux 2
  yum:
    name: docker
    state: present
  tags: docker

- name: Start and enable Docker
  service:
    name: docker
    state: started
    enabled: true
  tags: docker

- name: Add deploy user to docker group
  user:
    name: deploy
    groups: docker
    append: true
  tags: docker

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64"
    dest: /usr/local/bin/docker-compose
    mode: '0755'
  tags: docker

- name: Log in to Docker Hub
  community.docker.docker_login:
    username: "{{ vault_docker_username }}"
    password: "{{ vault_docker_password }}"
  become_user: deploy
  when: vault_docker_username is defined
  tags: docker

- name: Pull application image
  community.docker.docker_image:
    name: "{{ docker_app_image }}"
    tag: "{{ docker_app_tag }}"
    source: pull
  tags: docker

- name: Run application container
  community.docker.docker_container:
    name: "{{ docker_app_name }}"
    image: "{{ docker_app_image }}:{{ docker_app_tag }}"
    state: started
    restart_policy: always
    ports:
      - "{{ docker_app_port }}:{{ docker_container_port }}"
  tags: docker

- name: Wait for container to be healthy
  uri:
    url: "http://localhost:{{ docker_app_port }}"
    status_code: 200
  retries: 5
  delay: 3
  register: health_check
  until: health_check.status == 200
  tags: docker

```

```yaml
# roles/docker/handlers/main.yml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```

---

## Task 4: Nginx Role

```yaml
# roles/nginx/defaults/main.yml
---
nginx_http_port: 80
nginx_upstream_port: 8080
nginx_server_name: "_"
```

```yaml
# tasks file for roles/nginx

# roles/nginx/tasks/main.yml
---
- name: Enable Amazon Linux extras for nginx
  command: amazon-linux-extras enable nginx1
  when: ansible_distribution == "Amazon"
  changed_when: false
  tags: nginx

- name: Clean yum metadata
  command: yum clean metadata
  when: ansible_distribution == "Amazon"
  changed_when: false
  tags: nginx

- name: Install Nginx
  yum:
    name: nginx
    state: present
  tags: nginx

- name: Remove default Nginx site config
  file:
    path: /etc/nginx/conf.d/default.conf
    state: absent
  tags: nginx

- name: Deploy main Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
  notify: Reload Nginx
  tags: nginx

- name: Deploy reverse proxy config
  template:
    src: app-proxy.conf.j2
    dest: "/etc/nginx/conf.d/{{ project_name }}.conf"
    owner: root
    mode: '0644'
  notify: Reload Nginx
  tags: nginx

- name: Test Nginx configuration
  command: nginx -t
  changed_when: false
  tags: nginx

- name: Start and enable Nginx
  service:
    name: nginx
    state: started
    enabled: true
  tags: nginx

```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload Nginx
  service:
    name: nginx
    state: reloaded

- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```

```
{# roles/nginx/templates/nginx.conf.j2 #}
# Managed by Ansible -- do not edit manually
user nginx;
worker_processes {{ ansible_processor_vcpus | default(1) }};
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;
    include /etc/nginx/conf.d/*.conf;
}
```

```
{# roles/nginx/templates/app-proxy.conf.j2 #}
# Reverse Proxy to Docker Container -- Managed by Ansible
upstream docker_app {
    server 127.0.0.1:{{ nginx_upstream_port }};
}

server {
    listen {{ nginx_http_port }};
    server_name {{ nginx_server_name }};

    location / {
        proxy_pass http://docker_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

{% if app_env == 'production' %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log;
{% else %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log debug;
{% endif %}
}
```

---

## Task 5: Encrypt Docker Hub Credentials

```bash
# Create vault file
ansible-vault create group_vars/web/vault.yml
```

```yaml
# Contents of vault.yml (before encryption)
vault_docker_username: your-dockerhub-username
vault_docker_password: your-dockerhub-access-token
```

```bash
# Create password file
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass

# .gitignore
echo ".vault_pass" >> .gitignore
echo "*.tfstate" >> .gitignore
echo ".terraform/" >> .gitignore
```

---

## Task 6: Master Playbook and Deploy

```yaml
# site.yml
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common
  tags: common

- name: Install Docker and run containers
  hosts: web
  become: true
  roles:
    - docker
  tags: docker

- name: Configure Nginx reverse proxy
  hosts: web
  become: true
  roles:
    - nginx
  tags: nginx
```

```bash
# Always dry-run first
ansible-playbook site.yml --check --diff

# Full deploy
ansible-playbook site.yml

# Selective execution with tags
ansible-playbook site.yml --tags docker        # only Docker tasks
ansible-playbook site.yml --tags nginx         # only Nginx tasks
ansible-playbook site.yml --skip-tags common   # skip common setup

# Verify
curl http://<SERVER_IP>:8080    # direct to container
curl http://<SERVER_IP>:80      # through Nginx reverse proxy
curl http://<SERVER_IP>/health  # Nginx health endpoint

# Check on server
ssh ec2-user@<IP> docker ps
```

---

## Task 7: Bonus — Deploy a Different App and Prove Idempotency

```bash
# Swap the Docker image without touching Nginx
ansible-playbook site.yml --tags docker \
  -e "docker_app_image=httpd docker_app_tag=latest docker_app_name=apache-app"

# Full re-run -- should show mostly 'ok', minimal 'changed'
ansible-playbook site.yml
```

---

## Traffic Flow

```
Internet
    |
    | :80
    v
[Nginx on server]
    |  proxy_pass
    | :8080
    v
[Docker container]
    |  port mapping 8080:80
    v
[App inside container :80]
```

---

## Concepts Used Per Day

| Day | Concept Used in This Project |
| --- | --- |
| 68 | Inventory, SSH setup, ad-hoc ping to verify connectivity |
| 69 | Playbooks, modules (yum, service, copy, file, uri), handlers |
| 70 | Variables, facts (`ansible_processor_vcpus`), conditionals (`when`), loops |
| 71 | Roles, Jinja2 templates, Galaxy collection (`community.docker`), Vault |
| 72 | Everything combined: 3 roles, tags, master playbook, idempotency proof |

---

## What to Add for Production

| Enhancement | Tool |
| --- | --- |
| SSL/TLS termination | certbot + `nginx` role update |
| Multi-container apps | Docker Compose via template |
| Log rotation | `logrotate` task in common role |
| Monitoring | Prometheus + node_exporter role |
| Firewall rules | `firewalld` or `ufw` tasks |
| Fail2ban | Galaxy role for SSH protection |

---

## Quick Reference

```bash
# Full deploy
ansible-playbook site.yml

# Dry run
ansible-playbook site.yml --check --diff

# Tags
ansible-playbook site.yml --tags docker,nginx
ansible-playbook site.yml --skip-tags common

# Override image
ansible-playbook site.yml --tags docker \
  -e "docker_app_image=httpd docker_app_tag=latest"

# Vault
ansible-vault create group_vars/web/vault.yml
ansible-vault edit group_vars/web/vault.yml
ansible-vault view group_vars/web/vault.yml

# Collections
ansible-galaxy collection install community.docker
ansible-galaxy install -r requirements.yml

# Cleanup
terraform destroy    # if EC2s were provisioned with Terraform
```
<img width="1653" height="397" alt="image" src="https://github.com/user-attachments/assets/baeb7e99-228d-47df-936c-a059e57cd75c" />
