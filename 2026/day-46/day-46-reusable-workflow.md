# Day 46 – Reusable Workflows & Composite Actions

## Overview

Instead of copy-pasting workflow YAML across repos, teams build **reusable workflows** (called like functions with `workflow_call`) and **composite actions** (reusable step bundles). Today you build both and understand when to use each.

---

## Task 1: Understanding `workflow_call`

### What is a Reusable Workflow?

A reusable workflow is a regular GitHub Actions workflow file that exposes itself as callable by other workflows using the `workflow_call` trigger. Instead of duplicating YAML across repos, you define the logic once and call it from anywhere — like importing a function.

### What is the `workflow_call` Trigger?

`workflow_call` is an event trigger (like `push` or `pull_request`) that makes a workflow file invokable by other workflows. It supports typed `inputs:`, `secrets:`, and `outputs:` — exactly like function parameters and return values.

```yaml
on:
  workflow_call:
    inputs:
      app_name:
        type: string
        required: true
    secrets:
      docker_token:
        required: true
    outputs:
      build_version:
        value: ${{ jobs.build.outputs.version }}
```

### Reusable Workflow vs Regular Action (`uses:`)

|  | Reusable Workflow | Regular Action (`uses:`) |
| --- | --- | --- |
| **Unit** | An entire workflow (can contain multiple jobs) | A single step |
| **Syntax** | `uses:` at the **job** level | `uses:` at the **step** level |
| **Can have jobs** | Yes — full job orchestration | No — just steps |
| **Can accept secrets** | Yes — via `secrets:` block | No — secrets passed as inputs |
| **Lives in** | `.github/workflows/` | `.github/actions/<name>/` or a public repo |

### Where Must a Reusable Workflow Live?

- **Same repo:** `.github/workflows/filename.yml` → called with `uses: ./.github/workflows/filename.yml`
- **External repo:** `org/repo/.github/workflows/filename.yml@ref` → called with `uses: org/repo/.github/workflows/filename.yml@main`
- The file **must** be in `.github/workflows/` — not in subdirectories.

---

## Task 2: Create Your First Reusable Workflow

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build Workflow

on:
  workflow_call:
    inputs:
      app_name:
        description: Name of the application to build
        type: string
        required: true
      environment:
        description: Target environment
        type: string
        required: false
        default: staging
    secrets:
      docker_token:
        required: true
    outputs:
      build_version:
        description: The generated build version string
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.gen-version.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print build info
        run: |
          echo "Building ${{ inputs.app_name }} for ${{ inputs.environment }}"

      - name: Confirm secret is set
        run: |
          if [ -n "${{ secrets.docker_token }}" ]; then
            echo "Docker token is set: true"
          else
            echo "Docker token is set: false"
          fi

      - name: Generate build version
        id: gen-version
        run: |
          SHORT_SHA=$(echo ${{ github.sha }} | cut -c1-7)
          echo "version=v1.0-${SHORT_SHA}" >> $GITHUB_OUTPUT
          echo "Generated version: v1.0-${SHORT_SHA}"
```

> This file alone won't run — it needs a caller workflow to invoke it.
> 

---

## Task 3: Create a Caller Workflow

```yaml
# .github/workflows/call-build.yml
name: Caller Workflow

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      app_name: "my-web-app"
      environment: "production"
    secrets:
      docker_token: ${{ secrets.DOCKER_TOKEN }}

  print-version:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Print version from reusable workflow
        run: |
          echo "Build version: ${{ needs.build.outputs.build_version }}"
```

**What you see in the Actions tab:**

- The caller workflow (`call-build.yml`) shows as the top-level run
- Clicking into it shows the `build` job — but its steps are actually running inside `reusable-build.yml`


- GitHub renders a nested workflow graph: `Caller → reusable-build → build job → steps`

---
<img width="1887" height="892" alt="image" src="https://github.com/user-attachments/assets/5737d163-c1c3-4f7c-aa23-5c35bff27087" />

## Task 4: Outputs from a Reusable Workflow

Outputs flow upward through 3 levels:

```
Step output  →  Job output  →  Workflow output  →  Caller job reads it
```

```yaml
# Inside reusable-build.yml — already wired above:
on:
  workflow_call:
    outputs:
      build_version:
        value: ${{ jobs.build.outputs.version }}   # 3. workflow exposes job output

jobs:
  build:
    outputs:
      version: ${{ steps.gen-version.outputs.version }}  # 2. job exposes step output
    steps:
      - id: gen-version
        run: echo "version=v1.0-abc1234" >> $GITHUB_OUTPUT  # 1. step sets output
```

```yaml
# Inside call-build.yml — caller reads it:
  print-version:
    needs: build
    steps:
      - run: echo "${{ needs.build.outputs.build_version }}"
```

---

## Task 5: Create a Composite Action

Composite actions bundle multiple steps into a single reusable `uses:` call.

```yaml
# .github/actions/setup-and-greet/action.yml
name: Setup and Greet
description: Greets the user and prints environment info

inputs:
  name:
    description: Name to greet
    required: true
  language:
    description: Language for greeting (en, es, hi)
    required: false
    default: en

outputs:
  greeted:
    description: Whether the greeting was completed
    value: ${{ steps.greet.outputs.greeted }}

runs:
  using: composite
  steps:
    - name: Print greeting
      id: greet
      shell: bash
      run: |
        case "${{ inputs.language }}" in
          es) echo "Hola, ${{ inputs.name }}! 👋" ;;
          hi) echo "नमस्ते, ${{ inputs.name }}! 👋" ;;
          *)  echo "Hello, ${{ inputs.name }}! 👋" ;;
        esac
        echo "greeted=true" >> $GITHUB_OUTPUT

    - name: Print environment info
      shell: bash
      run: |
        echo "Date: $(date)"
        echo "Runner OS: ${{ runner.os }}"
        echo "Runner Arch: ${{ runner.arch }}"
```

### Workflow That Uses the Composite Action

```yaml
# .github/workflows/use-composite-action.yml
name: Use Composite Action

on:
  workflow_dispatch:

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (required to use local actions)
        uses: actions/checkout@v4

      - name: Run composite action (English)
        id: greet-en
        uses: ./.github/actions/setup-and-greet
        with:
          name: "DevOps Engineer"
          language: en

      - name: Run composite action (Spanish)
        uses: ./.github/actions/setup-and-greet
        with:
          name: "DevOps Engineer"
          language: es

      - name: Check output
        run: echo "Was greeted: ${{ steps.greet-en.outputs.greeted }}"
```

> **Important:** You must run `actions/checkout@v4` before using a local composite action. Without checkout, the runner doesn't have the `.github/actions/` directory.
> 

---
<img width="1890" height="896" alt="image" src="https://github.com/user-attachments/assets/305ea589-ff0d-4bcd-b08b-22b8b207c5b5" />

## Task 6: Reusable Workflow vs Composite Action

|  | Reusable Workflow | Composite Action |
| --- | --- | --- |
| **Triggered by** | `workflow_call` event | `uses:` in a step |
| **Can contain jobs** | ✅ Yes — full multi-job orchestration | ❌ No — steps only |
| **Can contain multiple steps** | ✅ Yes (within jobs) | ✅ Yes |
| **Lives where** | `.github/workflows/` | `.github/actions/<name>/action.yml` |
| **Can accept secrets directly** | ✅ Yes — via `secrets:` block | ❌ No — pass as inputs (treated as strings) |
| **Invoked at** | Job level (`jobs.<id>.uses:`) | Step level (`steps[*].uses:`) |
| **Outputs** | Via `on.workflow_call.outputs` | Via `outputs:`  • `$GITHUB_OUTPUT` |
| **Best for** | Full pipeline segments (build+test+deploy), cross-repo reuse, job-level parallelism | Grouping 2–10 steps that repeat across workflows, custom tooling, setup tasks |
| **Nesting limit** | 4 levels deep max | No limit |

### Mental Model

```
Reusable Workflow  =  a function that contains entire jobs
Composite Action   =  a function that contains steps within a job
```

Use a **reusable workflow** when you want to share multi-job pipeline logic across repos.

Use a **composite action** when you want to bundle a sequence of steps used repeatedly within jobs.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| `workflow_call` | Makes a workflow callable; supports typed inputs, secrets, and outputs |
| Caller syntax (same repo) | `uses: ./.github/workflows/file.yml` |
| Caller syntax (external repo) | `uses: org/repo/.github/workflows/file.yml@main` |
| Outputs chain | Step → Job → Workflow → Caller reads via `needs.<job>.outputs.<name>` |
| Composite action location | `.github/actions/<name>/action.yml` |
| `runs: using: composite` | Required declaration for composite actions |
| Checkout required | Must run `actions/checkout@v4` before any local `uses:` action |
| Max callers | A single reusable workflow can be called by up to 20 unique callers per run |
| Max nesting depth | Reusable workflows can be nested up to 4 levels deep |

---

## Quick Reference

```yaml
# Declare a reusable workflow
on:
  workflow_call:
    inputs:
      my_input:
        type: string
        required: true
    secrets:
      my_secret:
        required: true
    outputs:
      result:
        value: ${{ jobs.job1.outputs.result }}

# Call a reusable workflow (same repo)
jobs:
  call:
    uses: ./.github/workflows/reusable.yml
    with:
      my_input: "hello"
    secrets:
      my_secret: ${{ secrets.MY_SECRET }}

# Call a reusable workflow (external repo)
jobs:
  call:
    uses: org/repo/.github/workflows/reusable.yml@main
    with:
      my_input: "hello"
    secrets: inherit   # passes ALL caller secrets

# Composite action declaration
runs:
  using: composite
  steps:
    - shell: bash
      run: echo "hello"

# Use a local composite action
- uses: actions/checkout@v4
- uses: ./.github/actions/my-action
  with:
    name: value
```
