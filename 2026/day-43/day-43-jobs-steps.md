# Day 43 – Jobs, Steps, Env Vars & Conditionals

## Overview

Today covers **pipeline flow control** — chaining jobs with dependencies, scoping environment variables, passing data between jobs via outputs, and running steps conditionally. These are the building blocks of any real-world CI/CD workflow.

---

## Task 1: Multi-Job Workflow with Dependencies

Jobs run in parallel by default. Use `needs:` to create a dependency chain.

```yaml
# .github/workflows/multi-job.yml
name: Multi-Job Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: echo "Building the app..."

  test:
    runs-on: ubuntu-latest
    needs: build          # waits for build to succeed
    steps:
      - name: Test
        run: echo "Running tests..."

  deploy:
    runs-on: ubuntu-latest
    needs: test           # waits for test to succeed
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

**Verify:** In the Actions tab, click the workflow run — the graph view shows `build → test → deploy` as a linear chain. If `build` fails, `test` and `deploy` are skipped automatically.

> **Key rule:** `needs:` accepts a single job name or a list — `needs: [build, lint]` means "wait for both".
> 

---
