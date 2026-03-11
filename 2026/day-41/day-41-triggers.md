# Day 41 – Triggers & Matrix Builds

Create a markdown file:

`day-41-triggers.md`

Document each task.

---

# Task 1 – Trigger on Pull Request

Create:

```
.github/workflows/pr-check.yml
```

### Workflow

```yaml
name: PR Check

on:
  pull_request:
    branches:
      - main

jobs:
  pr-check:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Print Branch Name
        run: echo "PR check running for branch: ${{ github.head_ref }}"
```

### Steps to Test

1. Create new branch

```
git checkout -b feature-test
```

1. Make change and commit

```
git add .
git commit -m "Testing PR workflow"
git push origin feature-test
```

1. Open **Pull Request → main**

Verify:

- Go to **PR page**
- See **Checks running**

---
<img width="1918" height="657" alt="image" src="https://github.com/user-attachments/assets/b5a040bb-0198-48eb-81ed-27ad77f309d2" />


# Task 2 – Scheduled Trigger

Add this to any workflow.

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
```

### Meaning

```
0 0 * * *
│ │ │ │ │
│ │ │ │ └ Day of week
│ │ │ └ Month
│ │ └ Day of month
│ └ Hour
└ Minute
```

### Runs

**Every day at midnight UTC**

---

### Question

**Cron for every Monday at 9 AM**

Answer:

```
0 9 * * 1
```

---

# Task 3 – Manual Trigger

Create file:

```
.github/workflows/manual.yml
```

### Workflow

```yaml
name: Manual Workflow

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Choose environment"
        required: true
        default: "staging"

jobs:
  manual-job:
    runs-on: ubuntu-latest

    steps:
      - name: Print Environment
        run: echo "Deploying to ${{ github.event.inputs.environment }}"
```

### Run Manually

1. Go to **GitHub → Actions**
2. Select **Manual Workflow**
3. Click **Run workflow**
4. Enter:

```
staging
or
production
```

Verify output shows:

```
Deploying to staging
```

---
<img width="1903" height="595" alt="image" src="https://github.com/user-attachments/assets/2c7daaab-9e31-4a20-b3e0-04f95f08f328" />


# Task 4 – Matrix Builds

Create:

```
.github/workflows/matrix.yml
```

### Workflow

```yaml
name: Matrix Build

on:
  push:

jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os: [ubuntu-latest]
        python-version: [3.10, 3.11, 3.12]

    steps:
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Print Python Version
        run: python --version
```

### Result

3 jobs run in **parallel**.

```
Python 3.10
Python 3.11
Python 3.12
```

---

# Extend Matrix with OS

Update matrix:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: [3.10, 3.11, 3.12]
```

### Total Jobs

```
2 OS × 3 Python versions = 6 jobs
```

---
<img width="1918" height="617" alt="image" src="https://github.com/user-attachments/assets/1b6828f4-b001-4e96-9b87-f3dc7d19250e" />

# Task 5 – Exclude & Fail Fast

Update workflow:

```yaml
strategy:
  fail-fast: false

  matrix:
    os: [ubuntu-latest, windows-latest]
    python-version: [3.10, 3.11, 3.12]

    exclude:
      - os: windows-latest
        python-version: "3.10"
```

### Jobs Now

Original:

```
6
```

Excluded:

```
1
```

Final:

```
5 jobs
```

---

# Fail-Fast Explanation

### Default (fail-fast: true)

If **one matrix job fails**

➡ GitHub **cancels remaining jobs**

Example

```
3.10 fails
3.11 cancelled
3.12 cancelled
```

---

### fail-fast: false

If **one job fails**

➡ Other jobs **continue running**

Example

```
3.10 fails
3.11 runs
3.12 runs
```

This is useful for **testing multiple environments independently**.

---
