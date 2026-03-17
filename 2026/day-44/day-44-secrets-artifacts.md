# Day 44 – Secrets, Artifacts & Running Real Tests in CI

## Overview

Today the pipeline starts doing **real work** — storing sensitive values securely with GitHub Secrets, persisting build outputs with Artifacts, and running actual tests in CI. These three concepts together form the backbone of any production-grade pipeline.

---

## Task 1: GitHub Secrets

### Setting Up a Secret

Navigate to: `Repo → Settings → Secrets and Variables → Actions → New repository secret`

Create a secret named `MY_SECRET_MESSAGE` with any value.

### Workflow — Reading a Secret Safely

```yaml
# .github/workflows/secrets-demo.yml
name: Secrets Demo

on:
  workflow_dispatch:

jobs:
  check-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Confirm secret is set (safe)
        run: |
          if [ -n "${{ secrets.MY_SECRET_MESSAGE }}" ]; then
            echo "The secret is set: true"
          else
            echo "The secret is set: false"
          fi

      - name: What happens if you try to print directly?
        run: echo "${{ secrets.MY_SECRET_MESSAGE }}"
        # GitHub replaces the value with *** in logs automatically
```

### What Happens When You Print a Secret Directly?

GitHub automatically **masks** any secret value in logs, replacing it with `***`. So even if you accidentally echo a secret, the actual value never appears in the log output.

### Why You Should Never Print Secrets in CI Logs

- **Logs are not private** — CI logs are often visible to all repo collaborators, and in public repos, to everyone on the internet.
- **Masking is not foolproof** — If a secret is split across multiple echo calls or transformed (base64, URL-encoded), masking may not catch it.
- **Audit trail risk** — Logs are retained and searchable. Even a briefly exposed secret can be harvested.
- **Third-party log aggregators** — If you forward logs to Datadog, Splunk, or similar, secrets could end up in external systems where GitHub's masking doesn't apply.
- **Principle of least exposure** — Only confirm a secret *exists* (truthy check), never its value.

---
<img width="1918" height="901" alt="image" src="https://github.com/user-attachments/assets/fefc81f5-31e9-4796-9599-2370152ccb12" />


## Task 2: Secrets as Environment Variables

The correct pattern is to inject secrets as env vars — never interpolate them directly into shell commands.

```yaml
# .github/workflows/secrets-as-env.yml
name: Secrets as Env Vars

on:
  workflow_dispatch:

jobs:
  use-secrets:
    runs-on: ubuntu-latest
    steps:
      - name: Use secret as env var
        env:
          SECRET_MSG: ${{ secrets.MY_SECRET_MESSAGE }}
          DOCKER_USER: ${{ secrets.DOCKER_USERNAME }}
        run: |
          # Use $SECRET_MSG in shell — value is never hardcoded
          echo "Secret length: ${#SECRET_MSG}"
          echo "Docker user set: $( [ -n "$DOCKER_USER" ] && echo true || echo false )"

          # Example: use in a command without revealing the value
          # docker login -u "$DOCKER_USER" -p "$DOCKER_TOKEN"
```

### Secrets to Add Now (for Day 45 – Docker)

Go to `Settings → Secrets and Variables → Actions` and add:

| Secret Name | Value |
| --- | --- |
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_TOKEN` | Docker Hub access token (not password) |

> Generate a Docker Hub token at: [hub.docker.com](http://hub.docker.com) → Account Settings → Security → Access Tokens
> 

### Safe vs Unsafe Secret Usage

```yaml
# ❌ UNSAFE — secret interpolated directly into shell string
run: curl -u admin:${{ secrets.API_KEY }} https://api.example.com

# ✅ SAFE — secret injected as env var, used as shell variable
env:
  API_KEY: ${{ secrets.API_KEY }}
run: curl -u admin:"$API_KEY" https://api.example.com
```

---

<img width="1917" height="772" alt="image" src="https://github.com/user-attachments/assets/408cbc67-8a05-4847-807c-315c07122cb8" />

## Task 3: Upload Artifacts

Artifacts let you persist files from a workflow run — test reports, binaries, logs — and download them from the GitHub UI.

```yaml
# .github/workflows/upload-artifact.yml
name: Upload Artifact Demo

on:
  workflow_dispatch:

jobs:
  generate-and-upload:
    runs-on: ubuntu-latest
    steps:
      - name: Generate a test report
        run: |
          mkdir -p reports
          echo "Test Report - $(date)" > reports/test-report.txt
          echo "Tests passed: 42" >> reports/test-report.txt
          echo "Tests failed: 0" >> reports/test-report.txt
          cat reports/test-report.txt

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: reports/
          retention-days: 7    # artifact is kept for 7 days
```

**Verify:** After the run, go to `Actions → (your run) → Artifacts` section at the bottom. Click the artifact name to download a zip containing the report files.

---

<img width="1887" height="890" alt="image" src="https://github.com/user-attachments/assets/c15972da-de51-4218-b84d-97939e69f270" />

## Task 4: Download Artifacts Between Jobs

Artifacts are the correct mechanism for passing files between jobs (jobs run on separate VMs and don't share a filesystem).

```yaml
# .github/workflows/artifact-between-jobs.yml
name: Artifact Between Jobs

on:
  workflow_dispatch:

jobs:
  job-1-produce:
    runs-on: ubuntu-latest
    steps:
      - name: Generate file
        run: |
          mkdir output
          echo "Build ID: ${{ github.run_id }}" > output/build-info.txt
          echo "Branch: ${{ github.ref_name }}" >> output/build-info.txt
          echo "Commit: ${{ github.sha }}" >> output/build-info.txt

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-info
          path: output/

  job-2-consume:
    runs-on: ubuntu-latest
    needs: job-1-produce

    steps:
      - name: Download artifact from job 1
        uses: actions/download-artifact@v4
        with:
          name: build-info
          path: downloaded/

      - name: Print the artifact contents
        run: |
          echo "=== Contents from Job 1 ==="
          cat downloaded/build-info.txt
```

### When Would You Use Artifacts in a Real Pipeline?

| Use Case | Example |
| --- | --- |
| **Build → Test** | Compile a binary in `build`, download it in `test` to run tests against the exact binary |
| **Test reports** | Save JUnit XML or coverage HTML from `test`, download from UI for review |
| **Docker image layer cache** | Save layer tarballs between jobs to speed up multi-stage builds |
| **Release assets** | Build produces `.deb`, `.rpm`, `.exe` — release job picks them up to attach to a GitHub Release |
| **Debugging** | Upload logs and core dumps when a job fails for post-mortem analysis |
| **Cross-job data** | One job scrapes data and saves a JSON file; another job processes it |

---

<img width="1891" height="903" alt="image" src="https://github.com/user-attachments/assets/60fcf1cd-8045-486c-a46e-462b62c42efb" />

## Task 5: Run Real Tests in CI

### Example — Python Test Script

Add this to your repo as `scripts/test_math.py`:

```python
# scripts/test_math.py
def add(a, b):
    return a + b

def multiply(a, b):
    return a * b

def test_add():
    assert add(2, 3) == 5, "add(2,3) should be 5"
    assert add(0, 0) == 0, "add(0,0) should be 0"
    assert add(-1, 1) == 0, "add(-1,1) should be 0"
    print("✅ test_add passed")

def test_multiply():
    assert multiply(3, 4) == 12, "multiply(3,4) should be 12"
    assert multiply(0, 5) == 0,  "multiply(0,5) should be 0"
    print("✅ test_multiply passed")

if __name__ == "__main__":
    test_add()
    test_multiply()
    print("All tests passed!")
```

### CI Workflow

```yaml
# .github/workflows/run-tests.yml
name: Run Python Tests

on:
  push:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install pytest

      - name: Run tests
        run: python scripts/test_math.py
        # exits non-zero on AssertionError → pipeline goes red

      - name: Upload test results
        if: always()    # upload even if tests fail
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: scripts/
```

### Break/Fix Cycle

**Break it** — change `assert add(2, 3) == 5` to `== 99`. Push → pipeline goes 🔴.

**Fix it** — revert to `== 5`. Push → pipeline goes 🟢.

This confirms your pipeline correctly catches regressions.

---

<img width="1887" height="886" alt="image" src="https://github.com/user-attachments/assets/00c8ff41-1df2-4b10-89dc-b8d91a6375b6" />

## Task 6: Caching Dependencies

Caching saves time by reusing downloaded packages across runs instead of re-downloading them every time.

```yaml
# .github/workflows/cache-demo.yml
name: Cache Dependencies

on:
  push:
  workflow_dispatch:

jobs:
  install-with-cache:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip           # what to cache
          key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            pip-${{ runner.os }}-
            # fallback key if exact match not found

      - name: Install dependencies
        run: pip install pytest requests flask  # installs from cache on 2nd run

      - name: Run tests
        run: python scripts/test_math.py
```

### Cache Key Strategy

| Key Component | Purpose |
| --- | --- |
| `runner.os` | Separate caches per OS (Linux vs macOS vs Windows) |
| `hashFiles('requirements.txt')` | Invalidate cache when dependencies change |
| `restore-keys:` fallback | Use a partial cache if exact key not found (still faster than nothing) |

### First Run vs Second Run

- **Run 1:** Cache miss → packages downloaded fresh → slow (~30–60s for heavy deps)
- **Run 2:** Cache hit → packages restored from cache → fast (~2–5s)

### What Is Being Cached and Where?

- **What:** The pip package cache directory (`~/.cache/pip`) — the downloaded `.whl` files for all packages.
- **Where:** GitHub's managed cache storage (up to 10 GB per repo, per branch). The cache is stored on GitHub's infrastructure, not on the runner's disk. The runner downloads the cache archive at the start of the job via the `actions/cache` step.
- **Scope:** Cache keys are scoped to the branch by default. PRs can read from their base branch's cache but write to their own.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| Secrets | Never print values; use truthy checks only; inject as env vars |
| Secret masking | GitHub auto-masks in logs, but masking can fail on encoded values |
| Artifacts | Persist files from a run; downloaded via UI or `download-artifact` action |
| Artifact between jobs | Upload in Job 1, download in Job 2 — the only way to share files across jobs |
| Real tests in CI | Script must exit non-zero on failure; `actions/checkout` is always the first step |
| `if: always()` on upload | Ensures test reports are saved even when tests fail |
| Caching | Cache key = OS + hash of dependency file; restore-keys provide fallback |
| Cache storage | Managed by GitHub; up to 10 GB per repo; scoped to branch |

---

## Quick Reference

```yaml
# Secret
${{ secrets.SECRET_NAME }}

# Upload artifact
- uses: actions/upload-artifact@v4
  with:
    name: my-artifact
    path: dist/
    retention-days: 30

# Download artifact
- uses: actions/download-artifact@v4
  with:
    name: my-artifact
    path: downloaded/

# Cache
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
    restore-keys: pip-${{ runner.os }}-

# Checkout (always first!)
- uses: actions/checkout@v4

# Setup Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
```
