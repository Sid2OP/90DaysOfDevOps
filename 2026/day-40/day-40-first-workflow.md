# 1️⃣ Repository Setup

Create a repository:

```
github-actions-practice
```

Clone it:

```bash
git clone https://github.com/YOUR_USERNAME/github-actions-practice.git
cd github-actions-practice
```

Create workflow folder:

```bash
mkdir -p .github/workflows
```

---

# 2️⃣ Workflow File

Create:

```
.github/workflows/hello.yml
```

### hello.yml

```yaml
name: First GitHub Actions Workflow

on:
  push:

jobs:
  greet:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Say Hello
        run: echo "Hello from GitHub Actions!"

      - name: Print Date
        run: date

      - name: Print Branch Name
        run: echo "Branch: ${{ github.ref_name }}"

      - name: List Files
        run: ls -la

      - name: Print Runner OS
        run: echo "Runner OS: $RUNNER_OS"
```

Push it:

```bash
git add .
git commit -m "Add first GitHub Actions workflow"
git push
```

Then go to **Actions tab** in your repo and watch it run in **GitHub Actions**.

If everything is correct → pipeline will show **green checkmark ✅**

---

# 3️⃣ day-40-first-workflow.md

You can create this markdown file.

---

# Day 40 – First GitHub Actions Workflow

## What is GitHub Actions

**GitHub Actions** is a CI/CD platform that allows automation of:

- building code
- running tests
- deploying applications

Workflows are defined using **YAML files** inside `.github/workflows/`.

---

# Task 1 – Setup

Created repository:

```
github-actions-practice
```

Created workflow directory:

```
.github/workflows/
```

---

# Task 2 – First Workflow

Workflow file:

```
.github/workflows/hello.yml
```

Pipeline steps:

1. Trigger on push
2. Run job `greet`
3. Checkout repository
4. Print greeting message

Pipeline result: **Successful (Green)**

---

# Task 3 – Workflow Anatomy

### on:

Defines **what event triggers the workflow**

Example:

```yaml
on: push
```

This means the pipeline runs every time code is pushed.

---

### jobs:

Defines the **jobs executed in the workflow**

Each job runs independently.

Example:

```yaml
jobs:
  greet:
```

---

### runs-on:

Defines the **runner environment**

Example:

```
runs-on: ubuntu-latest
```

This runs the job on a Linux virtual machine.

---

### steps:

Defines **individual tasks inside a job**

Example:

```yaml
steps:
```

---

### uses:

Uses an existing **GitHub Action**

Example:

```yaml
uses: actions/checkout@v4
```

This action checks out the repository code.

---

### run:

Executes **shell commands**

Example:

```yaml
run: echo "Hello"
```

---

### name:

Adds a **label for the step**

Example:

```yaml
name: Print Date
```

This helps identify steps in the pipeline logs.

---

# Task 4 – Added Extra Steps

Added steps to:

- Print current date
- Print branch name
- List repository files
- Show runner OS

Example commands used:

```bash
date
ls -la
echo $RUNNER_OS
```

---

# Task 5 – Break the Pipeline

Added failing step:

```yaml
- name: Break Pipeline
  run: exit 1
```

Result:

- Pipeline **failed**
- GitHub showed **red ❌**

Error logs showed which step failed.

---

# What a Failed Pipeline Looks Like

When a step fails:

- Job stops execution
- Remaining steps do not run
- Pipeline shows **red status**

Logs show:

- failed step
- command executed
- error message

This helps developers debug quickly.

---
<img width="1897" height="905" alt="image" src="https://github.com/user-attachments/assets/6e282219-1151-4cd7-a054-61f19a1a6724" />
