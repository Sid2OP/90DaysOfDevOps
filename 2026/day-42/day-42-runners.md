Day 42 – Runners: GitHub-Hosted & Self-Hosted

## Overview

Every GitHub Actions job needs a **runner** — a machine that picks up the job, executes each step, and reports results back to GitHub. Runners come in two flavors: GitHub-hosted (fully managed by GitHub) and self-hosted (managed by you on your own infrastructure).

---

## Task 1: GitHub-Hosted Runners

### What is a GitHub-Hosted Runner?

A GitHub-hosted runner is a **virtual machine provisioned and managed entirely by GitHub**. When a job is triggered, GitHub spins up a fresh VM, runs your job, and then destroys it. You never touch the underlying infrastructure.

**Who manages it?** GitHub (Microsoft) manages everything — OS updates, security patches, pre-installed software, networking, and teardown.

### Multi-OS Workflow

```yaml
# .github/workflows/multi-os.yml
name: Multi-OS Runner Demo

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  ubuntu-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print OS Info
        run: |
          echo "OS: Ubuntu (Linux)"
          echo "Hostname: $(hostname)"
          echo "Current User: $(whoami)"

  windows-job:
    runs-on: windows-latest
    steps:
      - name: Print OS Info
        run: |
          echo "OS: Windows"
          echo "Hostname: $env:COMPUTERNAME"
          echo "Current User: $env:USERNAME"

  macos-job:
    runs-on: macos-latest
    steps:
      - name: Print OS Info
        run: |
          echo "OS: macOS"
          echo "Hostname: $(hostname)"
          echo "Current User: $(whoami)"
```
<img width="1912" height="908" alt="image" src="https://github.com/user-attachments/assets/67fd1f2c-8312-4dbf-b012-ffb8976d52a5" />

## Task 2: Pre-installed Software on ubuntu-latest

### Workflow to Check Versions

```yaml
# .github/workflows/check-tools.yml
name: Check Pre-installed Tools

on:
  workflow_dispatch:

jobs:
  check-tools:
    runs-on: ubuntu-latest
    steps:
      - name: Check Pre-installed Software Versions
        run: |
          echo "===== Docker ====="
          docker --version

          echo "===== Python ====="
          python3 --version

          echo "===== Node.js ====="
          node --version

          echo "===== Git ====="
          git --version
```

### Why Pre-installed Tools Matter

Pre-installed tools are critically important because:

- **Zero setup time** — You don't need `apt-get install` steps for common tools; they're ready immediately, keeping pipelines fast.
- **Version consistency** — GitHub pins specific versions per runner image, so your `docker build` or `python test` behaves the same every run.
- **Reduced workflow complexity** — No boilerplate installation steps cluttering your YAML.
- **Cost efficiency** — Fewer steps = less billable minutes on GitHub Actions.
- **Reliability** — Popular tools (Docker, Node, Python, Java, Go) are tested and validated by GitHub before image release.

### Full Pre-installed Software List

Reference: [GitHub Actions Runner Images](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)

Notable pre-installed tools on `ubuntu-latest`:

- Docker, Docker Compose
- Python 3.x, pip
- Node.js, npm, yarn
- Java (multiple versions via SDKMAN)
- Go, Ruby, .NET SDK
- Git, GitHub CLI (`gh`)
- AWS CLI, Azure CLI, Google Cloud SDK
- Terraform, kubectl, Helm
- jq, curl, wget

---
<img width="1918" height="897" alt="image" src="https://github.com/user-attachments/assets/8e3ea091-9159-4d33-a976-1325f9319c08" />

## Task 3: Self-Hosted Runner Setup

### What is a Self-Hosted Runner?

A self-hosted runner is a **machine you own and operate** (local server, VM, EC2, VPS) that connects to GitHub and listens for jobs assigned to it. You install the runner agent, register it with your repo, and GitHub sends jobs to it over HTTPS.

### Setup Steps

#### 1. Register the Runner in GitHub

Navigate to:

```
Your Repo → Settings → Actions → Runners → New self-hosted runner
```

Select **Linux** as the OS.

#### 2. Download and Configure on Your Machine/VM

```bash
# Create a dedicated directory
mkdir actions-runner && cd actions-runner

# Download the runner package (version from GitHub's instructions page)
curl -o actions-runner-linux-x64-2.x.x.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.x.x/actions-runner-linux-x64-2.x.x.tar.gz

# Extract
tar xzf ./actions-runner-linux-x64-2.x.x.tar.gz

# Configure — paste the token shown on GitHub's setup page
./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO \
            --token YOUR_REGISTRATION_TOKEN
```

#### 3. Start the Runner

```bash
# Run interactively (foreground)
./run.sh

# OR install as a persistent background service
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

#### 4. Verify

Go to **Settings → Actions → Runners** — your runner should appear with a **green dot (Idle)**, meaning it's connected and waiting for jobs.

---

## Task 4: Workflow Using Self-Hosted Runner

```yaml
# .github/workflows/self-hosted.yml
name: Self-Hosted Runner Job

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  run-on-my-machine:
    runs-on: self-hosted

    steps:
      - name: Print Hostname
        run: echo "Running on: $(hostname)"

      - name: Print Working Directory
        run: |
          echo "Working directory: $(pwd)"
          ls -la

      - name: Create a File
        run: |
          echo "This file was created by GitHub Actions at $(date)" > /tmp/actions-proof.txt
          echo "Job ID: ${{ github.run_id }}" >> /tmp/actions-proof.txt
          cat /tmp/actions-proof.txt

      - name: Verify File Exists
        run: |
          if [ -f /tmp/actions-proof.txt ]; then
            echo "✅ File exists!"
            cat /tmp/actions-proof.txt
          else
            echo "❌ File not found!"
            exit 1
          fi
```

**Verification:** After the workflow runs, SSH into your machine and check:

```bash
cat /tmp/actions-proof.txt
```

You should see the file with the timestamp and run ID written by your pipeline.

---

## Task 5: Runner Labels

### Adding a Label

Labels can be set during `./config.sh` configuration:

```bash
./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO \
            --token YOUR_TOKEN \
            --labels my-linux-runner
```

Or added afterward via **Settings → Actions → Runners → (your runner) → Edit**.

### Using a Label in Your Workflow

```yaml
# .github/workflows/self-hosted-labeled.yml
name: Labeled Self-Hosted Runner

on:
  workflow_dispatch:

jobs:
  run-on-labeled-runner:
    runs-on: [self-hosted, linux, my-linux-runner]

    steps:
      - name: Confirm correct runner
        run: |
          echo "Runner labels matched!"
          echo "Hostname: $(hostname)"
          echo "User: $(whoami)"
```

### Why Labels Are Essential

When you operate **multiple self-hosted runners**, labels solve critical routing problems:

| Scenario | Without Labels | With Labels |
| --- | --- | --- |
| GPU job needs GPU machine | Might run on any runner | `runs-on: [self-hosted, gpu]` |
| Job needs Windows | Could land on Linux | `runs-on: [self-hosted, windows]` |
| Production vs staging | No control | `runs-on: [self-hosted, production]` |
| High-memory workloads | Random assignment | `runs-on: [self-hosted, high-memory]` |

Labels let you **target specific machines** by capability, environment, architecture, or any custom attribute — critical in real-world fleets.

---

## Task 6: GitHub-Hosted vs Self-Hosted — Comparison

| Dimension | GitHub-Hosted | Self-Hosted |
| --- | --- | --- |
| **Who manages it?** | GitHub (Microsoft) | You |
| **Cost** | Free minutes per plan; billed after | You pay for infra; runner agent is free |
| **Pre-installed tools** | Rich set pre-baked (Docker, Node, Python…) | Only what you've manually installed |
| **Good for** | Open source, standard CI/CD, ephemeral builds | GPU/ARM hardware, private VPC, long-running jobs |
| **Security concern** | GitHub isolates VMs; fresh per job | Untrusted code runs on your machine; risky for public repos |
| **Persistence** | Ephemeral — destroyed after each job | Persistent — files/caches survive between runs |
| **Startup time** | ~10–30s to provision VM | Near-instant (runner always listening) |
| **Network access** | Public internet only | Full access to private network/internal services |

---
<img width="1918" height="900" alt="image" src="https://github.com/user-attachments/assets/dd276c34-c01c-41ff-b7cb-26be996ac204" />

## Key Takeaways

1. **GitHub-hosted runners** are the fastest way to get started — zero configuration, GitHub handles everything, and jobs always start in a clean environment.
2. **Self-hosted runners** are essential when you need private network access, specialized hardware (GPUs, ARM), cost optimization at scale, or compliance requirements.
3. **Labels** transform a fleet of runners into a routing system — always label by capability, environment, and OS.
4. **Never use self-hosted runners on public repos** without careful sandboxing — a malicious PR could execute arbitrary code on your machine.
5. **Install as a service** (`sudo ./svc.sh install`) so your runner survives reboots.

---

## Quick Reference

```bash
# Start runner interactively
./run.sh

# Install as system service (persists across reboots)
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh stop
sudo ./svc.sh status
sudo ./svc.sh uninstall

# runs-on targets
runs-on: ubuntu-latest            # GitHub-hosted Ubuntu
runs-on: windows-latest           # GitHub-hosted Windows
runs-on: macos-latest             # GitHub-hosted macOS
runs-on: self-hosted              # Any self-hosted runner
runs-on: [self-hosted, linux]     # Self-hosted Linux runners
runs-on: [self-hosted, my-label]  # Specific labeled runner
```
**Key behavior:** All 3 jobs run **in parallel** by default since they have no dependencies on each other. GitHub spins up 3 separate VMs simultaneously.

---
