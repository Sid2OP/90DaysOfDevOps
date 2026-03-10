## Day 38 – YAML Basics

### What is YAML

YAML (YAML Ain't Markup Language) is a human-readable data serialization format used in configuration files and CI/CD pipelines such as **GitHub Actions**, **Jenkins**, and **Kubernetes**.

Key Rules:

- YAML uses **spaces for indentation (not tabs)**.
- **2 spaces** indentation is the standard.
- Key-value format is `key: value`.
- Lists use  or `[item1, item2]`.

---

# Task 1 – Key Value Pairs

### person.yaml

```yaml
name: Siddharth Todkar
role: DevOps Engineer (Learning)
experience_years: 1
learning: true
```

Verification

```bash
cat person.yaml
```

Check:

- No tabs
- Clean indentation

---

# Task 2 – Lists

### Updated person.yaml

```yaml
name: Siddharth Todkar
role: DevOps Engineer (Learning)
experience_years: 1
learning: true

tools:
  - docker
  - git
  - aws
  - linux
  - kubernetes

hobbies: [coding, learning-devops, gaming]
```

### Two ways to write lists in YAML

**1. Block Style**

```yaml
tools:
  - docker
  - kubernetes
  - git
```

**2. Inline Style**

```yaml
tools: [docker, kubernetes, git]
```

---

# Task 3 – Nested Objects

### server.yaml

```yaml
server:
  name: dev-server
  ip: 192.168.1.10
  port: 8080

database:
  host: localhost
  name: app_db
  credentials:
    user: admin
    password: secret123
```

Important:

If you replace spaces with **tabs**, YAML validation will fail.

Example error:

```
found character '\t' that cannot start any token
```

---

# Task 4 – Multi-line Strings

### server.yaml with scripts

```yaml
server:
  name: dev-server
  ip: 192.168.1.10
  port: 8080

database:
  host: localhost
  name: app_db
  credentials:
    user: admin
    password: secret123

startup_script_literal: |
  #!/bin/bash
  echo "Starting server"
  docker-compose up -d
  echo "Server started"

startup_script_folded: >
  This script starts the application
  container using docker compose
  and runs it in detached mode.
```

### Difference between `|` and `>`

| Symbol | Behavior |
| --- | --- |
| ` | ` |
| `>` | Converts multiple lines into one paragraph |

Example:

`|`

```
line1
line2
line3
```

`>`

```
line1 line2 line3
```

### When to use them

Use `|` when writing:

- scripts
- configuration blocks

Use `>` when writing:

- long descriptions
- documentation text

---

# Task 5 – Validate YAML

Tool options:

- Install **yamllint**
- Use online validator like **YAML Lint**

Install:

```bash
sudo apt install yamllint
```

Validate:

```bash
yamllint person.yaml
yamllint server.yaml
```

Example indentation error:

```
error: wrong indentation: expected 2 but found 1
```

Fix indentation and run validation again.

---

# Task 6 – Spot the Difference

### Block 1 (Correct)

```yaml
name: devops
tools:
  - docker
  - kubernetes
```

### Block 2 (Broken)

```yaml
name: devops
tools:
- docker
  - kubernetes
```

### What is wrong?

- `docker` is **not indented under tools**
- YAML expects list items to be **properly aligned**

Correct version:

```yaml
tools:
  - docker
  - kubernetes
```

---
