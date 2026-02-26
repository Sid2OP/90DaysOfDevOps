# **Day 22 – Introduction to Git: Your First Repository**

# 📄 `git-commands.md`

```markdown
# Git Commands Reference Guide
(DevOps Learning Journal)

---

## 🔧 Setup & Config

### git --version
**What it does:** Checks installed Git version
**Example:**
```bash
git --version
```

### git config --global user.name

**What it does:** Sets your Git username

**Example:**

```bash
git config --global user.name "Your Name"
```

### git config --global user.email

**What it does:** Sets your Git email

**Example:**

```bash
git config --global user.email "youremail@example.com"
```

### git config --list

**What it does:** Shows all Git configurations

**Example:**

```bash
git config --list
```

---

## 📁 Basic Workflow

### git init

**What it does:** Initializes a new Git repository

**Example:**

```bash
git init
```

### git status

**What it does:** Shows current repo state

**Example:**

```bash
git status
```

### git add

**What it does:** Adds files to staging area

**Example:**

```bash
git add git-commands.md
```

### git commit

**What it does:** Saves staged changes to repository history

**Example:**

```bash
git commit -m "Add initial git commands reference"
```

---

## 👀 Viewing Changes

### git log

**What it does:** Shows commit history

**Example:**

```bash
git log
```

### git log --oneline

**What it does:** Shows compact commit history

**Example:**

```bash
git log --oneline
```

### git diff

**What it does:** Shows unstaged changes

**Example:**

```bash
git diff
```

---

📸 **Task Screenshot Section**

---

✅ Living document — will grow daily as Git knowledge increases

```

---

# 📄 `day-22-notes.md`

```md
# Day 22 – Introduction to Git
(DevOps Learning Notes)

---

## 🧠 What is Git and Why It Matters

Git is a distributed version control system that tracks changes in code, enables collaboration, maintains history, and supports automation.
It is the backbone of DevOps workflows, CI/CD pipelines, infrastructure-as-code, and modern software delivery.

---

## 📂 Repository Setup

Project folder:
```bash
devops-git-practice
```

Initialized using:

```bash
git init
```

---

## 🔁 Git Workflow Understanding

### Difference between `git add` and `git commit`

`git add` moves changes to the staging area.

`git commit` permanently stores staged changes in the repository history.

---

### What does the staging area do?

It allows control over what gets committed, enabling clean, logical, and organized commits.

---

### Why doesn't Git commit directly?

Because developers need a review layer to organize, validate, and group changes before saving them permanently.

---

### What information does `git log` show?

Commit hash, author, date, and commit message — full project history.

---

### What is the `.git/` folder?

It contains all repository data (history, branches, config, commits).

Deleting it removes Git completely from the project.

---

### Difference between working directory, staging area, and repository

**Working Directory** → actual files

**Staging Area** → prepared changes

**Repository** → permanent history

Flow:

```
Working Directory → Staging Area → Repository
```

<img width="1087" height="752" alt="image" src="https://github.com/user-attachments/assets/2a54147f-a172-4fcc-ab22-54ade32bfa37" />
<img width="1037" height="333" alt="image" src="https://github.com/user-attachments/assets/5872c1b8-589b-408b-b4c8-911bf309112b" />

