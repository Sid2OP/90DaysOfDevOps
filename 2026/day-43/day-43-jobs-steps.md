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
<img width="1918" height="900" alt="image" src="https://github.com/user-attachments/assets/84541a7e-8a18-49ac-9831-f748e688162e" />

## Task 2: Environment Variables

Env vars can be scoped at 3 levels. Inner scopes override outer ones.

```yaml
# .github/workflows/env-vars.yml
name: Environment Variables Demo

on:
  workflow_dispatch:

env:
  APP_NAME: myapp          # Workflow level — available to all jobs and steps

jobs:
  print-vars:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: staging   # Job level — available to all steps in this job

    steps:
      - name: Print all env vars
        env:
          VERSION: 1.0.0     # Step level — only available in this step
        run: |
          echo "App Name (workflow):  $APP_NAME"
          echo "Environment (job):    $ENVIRONMENT"
          echo "Version (step):       $VERSION"

      - name: Print GitHub context variables
        run: |
          echo "Commit SHA:  ${{ github.sha }}"
          echo "Triggered by: ${{ github.actor }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "Event: ${{ github.event_name }}"
```

### Variable Scope Summary

| Level | Defined under | Available to |
| --- | --- | --- |
| Workflow | Top-level `env:` | All jobs and steps |
| Job | `jobs.<job>.env:` | All steps in that job |
| Step | `jobs.<job>.steps[*].env:` | That step only |
| GitHub context | `${{ github.* }}` | Anywhere in the workflow |

---
<img width="1918" height="870" alt="image" src="https://github.com/user-attachments/assets/155d20fd-6005-41a4-9fc0-faccd6a0586e" />

## Task 3: Job Outputs

Jobs are isolated — they run on separate VMs. To pass data between jobs, use **outputs**.

```yaml
# .github/workflows/job-outputs.yml
name: Job Outputs Demo

on:
  workflow_dispatch:

jobs:
  generate:
    runs-on: ubuntu-latest
    outputs:
      build_date: ${{ steps.set-date.outputs.date }}   # expose step output as job output

    steps:
      - name: Set today's date
        id: set-date                                   # id required to reference output
        run: echo "date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT

      - name: Confirm it was set
        run: echo "Date set to: ${{ steps.set-date.outputs.date }}"

  consume:
    runs-on: ubuntu-latest
    needs: generate           # must depend on the producing job

    steps:
      - name: Use the date from the previous job
        run: |
          echo "Build date from 'generate' job: ${{ needs.generate.outputs.build_date }}"
```

### Why Pass Outputs Between Jobs?

- **Version tagging** — a `build` job computes a version string (e.g., `1.4.2-abc1234`), and downstream `deploy` and `release` jobs use that exact same string.
- **Dynamic matrix** — a `setup` job generates a list of services to test; a matrix `test` job uses that list to spin up parallel runners.
- **Artifact paths** — a `build` job outputs the path to a compiled binary; a `test` job knows exactly where to find it.
- **Conditional routing** — a job outputs a flag like `changed=true`; downstream jobs run only if that flag is set.

Without outputs, each job would have to recompute or re-fetch the same information independently, causing inconsistency and wasted time.

---
<img width="1918" height="905" alt="image" src="https://github.com/user-attachments/assets/afa94554-2711-4ae2-9536-bcbc78c24b1c" />

## Task 4: Conditionals

```yaml
# .github/workflows/conditionals.yml
name: Conditionals Demo

on:
  push:
  pull_request:

jobs:
  # Job-level conditional: only runs on push events, not PRs
  push-only-job:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      # Step only runs when branch is main
      - name: Main branch step
        if: github.ref == 'refs/heads/main'
        run: echo "This is a push to main!"

      # Always runs (no condition)
      - name: Risky step
        id: risky
        run: |
          echo "Attempting something that might fail..."
          exit 1
        continue-on-error: true   # pipeline continues even if this step fails

      # Only runs if the previous step failed
      - name: Handle failure
        if: steps.risky.outcome == 'failure'
        run: echo "Risky step failed, but we caught it and continued!"

      # Always runs regardless of earlier failures
      - name: Cleanup
        if: always()
        run: echo "Cleanup always runs."
```

### Conditional Reference

| Expression | When it runs |
| --- | --- |
| `if: github.ref == 'refs/heads/main'` | Only on the `main` branch |
| `if: github.event_name == 'push'` | Only on push events |
| `if: steps.<id>.outcome == 'failure'` | Only if a specific step failed |
| `if: failure()` | If any previous step in the job failed |
| `if: success()` | Default behavior — only if all previous steps passed |
| `if: always()` | Regardless of success or failure |
| `continue-on-error: true` | Step failure is ignored; job continues and marks the step yellow, not red |

> **`continue-on-error: true`** — the step is allowed to fail without failing the entire job. The step shows a warning icon in the UI. Useful for non-critical steps like coverage uploads or optional linters.
> 

---
<img width="1888" height="888" alt="image" src="https://github.com/user-attachments/assets/73e21539-907e-4243-a2c7-3bc5e27dac7a" />

## Task 5: Smart Pipeline

```yaml
# .github/workflows/smart-pipeline.yml
name: Smart Pipeline

on:
  push:               # triggers on any branch

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Run Linter
        run: |
          echo "Running lint checks..."
          echo "Branch: ${{ github.ref_name }}"

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run Tests
        run: |
          echo "Running test suite..."
          echo "Commit: ${{ github.sha }}"

  summary:
    runs-on: ubuntu-latest
    needs: [lint, test]       # runs after BOTH lint and test complete

    steps:
      - name: Main branch summary
        if: github.ref == 'refs/heads/main'
        run: |
          echo "✅ Production push to main branch"
          echo "Commit message: ${{ github.event.commits[0].message }}"
          echo "Triggered by: ${{ github.actor }}"
          echo "SHA: ${{ github.sha }}"

      - name: Feature branch summary
        if: github.ref != 'refs/heads/main'
        run: |
          echo "🔀 Feature branch push: ${{ github.ref_name }}"
          echo "Commit message: ${{ github.event.commits[0].message }}"
          echo "Triggered by: ${{ github.actor }}"
          echo "SHA: ${{ github.sha }}"
```

**Workflow graph:**

```
lint ──┐
       ├──► summary
test ──┘
```

`lint` and `test` run **in parallel**. `summary` waits for both and then runs with branch-aware output.

---

## Key Concepts Summary

| Concept | Syntax | Purpose |
| --- | --- | --- |
| Job dependency | `needs: job-name` | Run jobs in sequence |
| Workflow env var | top-level `env:` | Shared across all jobs |
| Job env var | `jobs.<job>.env:` | Scoped to one job |
| Step env var | `steps[*].env:` | Scoped to one step |
| Set output | `echo "key=val" >> $GITHUB_OUTPUT` | Pass data from a step |
| Expose job output | `outputs: name: ${{ steps.id.outputs.key }}` | Make step output available to other jobs |
| Read job output | `${{ needs.job.outputs.name }}` | Consume another job's output |
| Branch conditional | `if: github.ref == 'refs/heads/main'` | Run only on a specific branch |
| Failure handler | `if: steps.id.outcome == 'failure'` | React to a specific step failing |
| Non-fatal step | `continue-on-error: true` | Let a step fail without breaking the job |
| Always run | `if: always()` | Run regardless of earlier failures |

---

