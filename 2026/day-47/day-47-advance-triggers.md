# Day 47 – Advanced Triggers: PR Events, Cron Schedules & Event-Driven Pipelines

## Overview

GitHub Actions supports dozens of event types beyond basic `push`. Today you go deep into **PR lifecycle events**, **scheduled cron jobs**, **path/branch filters**, and **workflow chaining** — the patterns that make pipelines truly intelligent and event-driven.

---

## Task 1: Pull Request Event Types

```yaml
# .github/workflows/pr-lifecycle.yml
name: PR Lifecycle Events

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  pr-event-logger:
    runs-on: ubuntu-latest
    steps:
      - name: Log PR event details
        run: |
          echo "Event type:     ${{ github.event.action }}"
          echo "PR title:       ${{ github.event.pull_request.title }}"
          echo "PR author:      ${{ github.event.pull_request.user.login }}"
          echo "Source branch:  ${{ github.head_ref }}"
          echo "Target branch:  ${{ github.base_ref }}"
          echo "PR number:      #${{ github.event.pull_request.number }}"

      - name: Merged step (only on merge)
        if: github.event.action == 'closed' && github.event.pull_request.merged == true
        run: |
          echo "PR was MERGED into ${{ github.base_ref }}!"
          echo "Merged by: ${{ github.event.pull_request.merged_by.login }}"
          echo "Merge commit: ${{ github.event.pull_request.merge_commit_sha }}"
```
<img width="1912" height="902" alt="image" src="https://github.com/user-attachments/assets/eafe7eb0-4a98-454a-8c1c-485d587f5308" />


### PR Event Types Explained

| Event type | When it fires |
| --- | --- |
| `opened` | PR is first created |
| `synchronize` | New commit pushed to the PR branch |
| `reopened` | A closed PR is re-opened |
| `closed` | PR is closed (merged OR abandoned — check `.merged`) |
| `ready_for_review` | Draft PR marked as ready |
| `review_requested` | Reviewer is requested |

**Test it:** Create a PR → push an update to it → merge it. Each action fires the workflow with a different `github.event.action` value.

---

## Task 2: PR Validation Workflow

```yaml
# .github/workflows/pr-checks.yml
name: PR Validation Checks

on:
  pull_request:
    branches: [main]

jobs:
  file-size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # full history needed to diff

      - name: Check no file exceeds 1 MB
        run: |
          git diff --name-only origin/${{ github.base_ref }}...HEAD | while read file; do
            if [ -f "$file" ]; then
              size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file")
              if [ "$size" -gt 1048576 ]; then
                echo "ERROR: $file is $(($size / 1024))KB — exceeds 1 MB limit"
                exit 1
              fi
            fi
          done
          echo "All files are within the 1 MB limit."

  branch-name-check:
    runs-on: ubuntu-latest
    steps:
      - name: Validate branch naming convention
        run: |
          BRANCH="${{ github.head_ref }}"
          echo "Branch: $BRANCH"
          if [[ "$BRANCH" =~ ^(feature|fix|docs)/.+ ]]; then
            echo "Branch name is valid."
          else
            echo "ERROR: Branch '$BRANCH' does not follow naming convention."
            echo "Expected: feature/*, fix/*, or docs/*"
            exit 1
          fi

  pr-body-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check PR description
        run: |
          BODY="${{ github.event.pull_request.body }}"
          if [ -z "$BODY" ]; then
            echo "WARNING: PR description is empty. Please describe your changes."
            # continue-on-error is set below so this does not fail the job
          else
            echo "PR description found."
            echo "First 100 chars: ${BODY:0:100}"
          fi
        continue-on-error: true    # warns but doesn't block the PR
```

**Verify:** Open a PR from a branch named `my-random-branch` — the `branch-name-check` job fails immediately.

---

## Task 3: Scheduled Workflows (Cron Deep Dive)

```yaml
# .github/workflows/scheduled-tasks.yml
name: Scheduled Tasks

on:
  schedule:
    - cron: '30 2 * * 1'     # Every Monday at 2:30 AM UTC
    - cron: '0 */6 * * *'    # Every 6 hours
  workflow_dispatch:          # Manual trigger for testing

jobs:
  scheduled-health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Which schedule triggered this?
        run: |
          echo "Schedule: ${{ github.event.schedule }}"
          echo "Event: ${{ github.event_name }}"

      - name: Health check
        run: |
          STATUS=$(curl -o /dev/null -s -w "%{http_code}" https://github.com)
          echo "GitHub status code: $STATUS"
          if [ "$STATUS" != "200" ]; then
            echo "Health check failed!"
            exit 1
          fi
          echo "Health check passed."
```

### Cron Expressions

```
# Cron format: minute  hour  day-of-month  month  day-of-week
#                 *      *         *          *        *

'30 2 * * 1'     # Every Monday at 2:30 AM UTC
'0 */6 * * *'    # Every 6 hours
'30 3 * * 1-5'   # Every weekday at 9:00 AM IST (IST = UTC+5:30, so 9:00 IST = 3:30 UTC)
'0 0 1 * *'      # First day of every month at midnight UTC
'0 0 * * 0'      # Every Sunday at midnight
'*/15 * * * *'   # Every 15 minutes
```

### Cron Expressions for the Notes

| Goal | Cron (UTC) | Note |
| --- | --- | --- |
| Every weekday at 9 AM IST | `30 3 * * 1-5` | IST is UTC+5:30, so 9:00 IST = 03:30 UTC |
| First day of every month at midnight | `0 0 1 * *` | Midnight UTC |

### Why Scheduled Workflows May Be Delayed or Skipped

GitHub explicitly warns that scheduled workflows **may be delayed or not run** on inactive repos. Reasons:

- Scheduled jobs run on GitHub's shared scheduler. During high demand, jobs are queued and may start minutes or hours late.
- **GitHub disables scheduled workflows on repos with no activity for 60 days.** A push, PR, or manual trigger reactivates them.
- Scheduled workflows only run on the **default branch** — they cannot be triggered on feature branches.
- If your repo is public and inactive, GitHub may deprioritize it to allocate compute to active repos.

---

## Task 4: Path & Branch Filters

```yaml
# .github/workflows/smart-triggers.yml
name: Smart Path & Branch Triggers

on:
  push:
    branches:
      - main
      - 'release/**'
    paths:
      - 'src/**'
      - 'app/**'

---

# Separate workflow using paths-ignore
name: Skip Docs-Only Changes

on:
  push:
    branches: [main]
    paths-ignore:
      - '*.md'
      - 'docs/**'
      - '.github/**'
      - '*.txt'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build (only when real code changed)
        run: echo "Code changed — building..."
```

### `paths` vs `paths-ignore`

|  | `paths` | `paths-ignore` |
| --- | --- | --- |
| **Logic** | Whitelist — ONLY run if any listed path changed | Blacklist — SKIP if ALL changed files match |
| **Best for** | Monorepos (trigger only the service that changed) | Skipping docs/config-only changes |
| **Gotcha** | Both cannot be used together in the same trigger | If even one non-ignored file changed, it runs |

**When to use `paths`:** Monorepo with `services/api/`, `services/web/`, `services/worker/` — each service has its own workflow triggered only when its directory changes.

**When to use `paths-ignore`:** A full-stack repo where pushing a README fix shouldn't re-run your entire test + build + deploy pipeline.

**Test it:** Push a change only to `README.md` — the `paths-ignore` workflow is skipped entirely. ✅

---

## Task 5: `workflow_run` — Chain Workflows Together

```yaml
# .github/workflows/tests.yml
name: Run Tests

on:
  push:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: python scripts/test_math.py
```

```yaml
# .github/workflows/deploy-after-tests.yml
name: Deploy After Tests

on:
  workflow_run:
    workflows: ["Run Tests"]    # must match the 'name:' field exactly
    types: [completed]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Check if tests passed
        run: |
          CONCLUSION="${{ github.event.workflow_run.conclusion }}"
          echo "Triggering workflow conclusion: $CONCLUSION"

          if [ "$CONCLUSION" != "success" ]; then
            echo "Tests did not pass (conclusion: $CONCLUSION). Skipping deploy."
            exit 1
          fi

      - name: Deploy
        run: |
          echo "Tests passed! Deploying..."
          echo "Triggered by commit: ${{ github.event.workflow_run.head_sha }}"
          echo "Branch: ${{ github.event.workflow_run.head_branch }}"
```

### Key `workflow_run` Facts

- The deploy workflow runs on the **default branch**, even if the triggering push was to another branch.
- Access the triggering run's artifacts using `actions/download-artifact` with the `run-id` from `github.event.workflow_run.id`.
- `github.event.workflow_run.conclusion` values: `success`, `failure`, `cancelled`, `skipped`, `timed_out`.
- The `name:` field in the caller must **exactly match** the `name:` of the target workflow.

---

## Task 6: `repository_dispatch` — External Event Triggers

```yaml
# .github/workflows/external-trigger.yml
name: External Deploy Trigger

on:
  repository_dispatch:
    types: [deploy-request]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Print dispatch payload
        run: |
          echo "Triggered by external event: deploy-request"
          echo "Environment: ${{ github.event.client_payload.environment }}"
          echo "Version:     ${{ github.event.client_payload.version }}"
          echo "Requester:   ${{ github.event.client_payload.requester }}"

      - name: Run deployment
        run: |
          echo "Deploying to ${{ github.event.client_payload.environment }}..."
```

### Triggering with `gh` CLI

```bash
# Trigger a repository_dispatch event
gh api repos/<owner>/<repo>/dispatches \
  -f event_type=deploy-request \
  -f client_payload='{"environment":"production","version":"v1.4.2","requester":"slack-bot"}'

# Or with curl (requires PAT with repo scope)
curl -X POST \
  -H "Authorization: token YOUR_PAT" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/<owner>/<repo>/dispatches \
  -d '{"event_type":"deploy-request","client_payload":{"environment":"staging"}}'
```

### When Would an External System Trigger a Pipeline?

| External System | Use Case |
| --- | --- |
| **Slack bot** | `/deploy production` command in Slack triggers a deploy workflow |
| **Monitoring tool** | Alerting system detects a rollback condition and triggers `rollback-deploy` |
| **Another CI system** | Jenkins or CircleCI completes a build and hands off to GitHub Actions for deployment |
| **Webhook from SaaS** | Stripe, PagerDuty, or Datadog fires an event that triggers a remediation workflow |
| **Scheduled external job** | A cron job on a server triggers a data pipeline at an exact time with custom payload |
| **Approval system** | A custom approval portal POSTs to GitHub after a human approves a release |

`repository_dispatch` is essentially a **webhook you control** — any system that can make an HTTPS POST request can trigger your pipeline.

---

## Key Concepts Summary

| Concept | Key Point |
| --- | --- |
| PR event types | `types: [opened, synchronize, reopened, closed]` — closed ≠ merged, check `.merged` |
| PR merge check | `if: github.event.action == 'closed' && github.event.pull_request.merged == true` |
| Cron triggers | Only run on the default branch; may be delayed or disabled on inactive repos |
| `paths:` | Whitelist — workflow runs only if a matching path changed |
| `paths-ignore:` | Blacklist — workflow skips if ALL changed files match |
| `workflow_run` | Chains workflows; `name:` must match exactly; runs on default branch |
| `repository_dispatch` | External HTTP trigger; payload available via `github.event.client_payload.*` |
| IST → UTC | Subtract 5h 30m (9:00 AM IST = 3:30 AM UTC) |

---

## Quick Reference

```yaml
# PR with activity types
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

# PR merge conditional
if: github.event.action == 'closed' && github.event.pull_request.merged == true

# Cron schedule
on:
  schedule:
    - cron: '30 3 * * 1-5'   # weekdays 9 AM IST
    - cron: '0 0 1 * *'       # 1st of month midnight
  workflow_dispatch:           # always add this for manual testing

# Path filters
on:
  push:
    paths: ['src/**', 'app/**']
    paths-ignore: ['*.md', 'docs/**']
    branches: [main, 'release/**']

# Chain with workflow_run
on:
  workflow_run:
    workflows: ["Run Tests"]  # exact name match
    types: [completed]

# Conclusion check
if: github.event.workflow_run.conclusion == 'success'

# Repository dispatch trigger
on:
  repository_dispatch:
    types: [deploy-request]

# Read client payload
${{ github.event.client_payload.environment }}
```
