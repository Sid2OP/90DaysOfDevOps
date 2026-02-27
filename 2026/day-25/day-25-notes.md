# ✅ Day 25 — Git Reset vs Revert & Branching Strategies

(**Commands Only**)

---

## 🔁 Task 1: Git Reset — Hands-On

### Create commits A, B, C

```bash
echo "A" > reset.txt
git add reset.txt
git commit -m "Commit A"

echo "B" >> reset.txt
git add reset.txt
git commit -m "Commit B"

echo "C" >> reset.txt
git add reset.txt
git commit -m "Commit C"
```

### View history

```bash
git log --oneline
```

---

### `git reset --soft` (go back 1 commit)

```bash
git reset --soft HEAD~1
git status
```
<img width="907" height="612" alt="image" src="https://github.com/user-attachments/assets/fd3faef1-08c4-4ae1-a7a9-a4449975edc4" />
<img width="927" height="442" alt="image" src="https://github.com/user-attachments/assets/c45a8dc0-c74d-466e-bc08-42236be0525c" />

### Re-commit

```bash
git commit -m "Commit C again"
```

---

### `git reset --mixed` (go back 1 commit)

```bash
git reset --mixed HEAD~1
git status
```
<img width="885" height="246" alt="image" src="https://github.com/user-attachments/assets/2a09d994-00bb-4121-9df3-eb6d502c83b2" />

### Re-commit

```bash
git add reset.txt
git commit -m "Commit C again (mixed)"
```

---

### `git reset --hard` (go back 1 commit)

```bash
git reset --hard HEAD~1
git status
```

---
<img width="1117" height="417" alt="image" src="https://github.com/user-attachments/assets/ee235a58-a94d-4b1e-9251-bb5398d72588" />

## ↩️ Task 2: Git Revert — Hands-On

### Create commits X, Y, Z

```bash
echo "X" > revert.txt
git add revert.txt
git commit -m "Commit X"

echo "Y" >> revert.txt
git add revert.txt
git commit -m "Commit Y"

echo "Z" >> revert.txt
git add revert.txt
git commit -m "Commit Z"
```

### View commits

```bash
git log --oneline
```

### Revert middle commit (Y)

```bash
git revert <commit-hash-of-Y>
```

### Verify history

```bash
git log --oneline
```

---
<img width="885" height="238" alt="image" src="https://github.com/user-attachments/assets/95bb03cf-4104-4ce0-84d6-3509cf943a01" />

## 🔄 Task 3: Reset vs Revert — Practice Commands

```bash
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1

git revert HEAD
git revert <commit-hash>
```

---

## 🌿 Task 4: Branching Strategies — Practice Simulation Commands

### GitFlow style

```bash
git branch develop
git checkout -b feature-auth develop
git checkout -b release-v1 develop
git checkout -b hotfix-prod main
```

---

### GitHub Flow style

```bash
git checkout -b feature-payment main
git commit -am "Feature work"
git checkout main
git merge feature-payment
```

---

### Trunk-Based Development style

```bash
git checkout main
git checkout -b short-fix
git commit -am "Quick fix"
git checkout main
git merge short-fix
```

---
# 📄 `day-25-notes.md` — Answers Section

---

## 🧠 Task 1: Git Reset — Answers

### Difference between `-soft`, `-mixed`, and `-hard`

**`git reset --soft`**

- Moves HEAD to previous commit
- Changes stay in **staging area**
- Nothing is lost

**`git reset --mixed` (default)**

- Moves HEAD to previous commit
- Changes stay in **working directory**
- Staging area is cleared

**`git reset --hard`**

- Moves HEAD to previous commit
- **Deletes changes from staging and working directory**
- Data is lost permanently

---

### Which one is destructive and why?

**`git reset --hard` is destructive**

Because it permanently deletes:

- file changes
- staged changes
- commit history reference

---

### When would you use each one?

- **-soft**
- When you want to edit the last commit message or combine commits
- **-mixed**
- When you want to redo staging properly
- **-hard**
- When you want to completely discard changes and go back clean

---

### Should you ever use `git reset` on pushed commits?

❌ **No**

Because it rewrites history and breaks other developers’ repositories, causing sync conflicts.

---

## ↩️ Task 2: Git Revert — Answers

### What happens when you revert the middle commit?

Git creates a **new commit** that reverses the changes of that commit.

---

### Is the reverted commit still in history?

✅ **Yes**

The original commit remains, plus a new "revert commit" is added.

---

### How is `git revert` different from `git reset`?

**git reset**

- Rewrites history
- Removes commits from branch history
- Dangerous on shared branches

**git revert**

- Does not rewrite history
- Adds a new commit
- Safe for shared branches

---

### Why is revert safer for shared branches?

Because it preserves history and does not change commit hashes, so collaborators stay in sync.

---

### When would you use revert vs reset?

**Use revert:**

- On shared branches
- On production/main branches
- In team environments

**Use reset:**

- On local branches
- For unpushed commits
- For personal cleanup

---

## 🔄 Task 3: Reset vs Revert — Comparison

| Feature | git reset | git revert |
| --- | --- | --- |
| What it does | Moves HEAD backward | Creates reverse commit |
| Removes commit from history | ✅ Yes | ❌ No |
| Safe for shared branches | ❌ No | ✅ Yes |
| History rewriting | ✅ Yes | ❌ No |
| When to use | Local cleanup | Production safety |

---

## 🌿 Task 4: Branching Strategies — Answers

---

### 🌱 GitFlow

**How it works:**

Multiple long-lived branches (`main`, `develop`) + support branches (`feature`, `release`, `hotfix`)

**Flow:**

```
main
  |
develop
  |—— feature/*
  |—— release/*
  |—— hotfix/*
```

**Used in:**

Large teams, enterprise systems, scheduled releases

**Pros:**

- Structured
- Stable releases
- Clear workflows

**Cons:**

- Complex
- Heavy process
- Slow for fast startups

---

### 🌿 GitHub Flow

**How it works:**

Single `main` branch + short-lived feature branches

**Flow:**

```
main
  |
feature-branch → PR → merge
```

**Used in:**

Startups, SaaS, fast delivery teams

**Pros:**

- Simple
- Fast shipping
- Easy collaboration

**Cons:**

- Needs strong CI/CD
- Less structure for big releases

---

### 🌳 Trunk-Based Development

**How it works:**

Everyone commits to `main` (trunk)

Branches are very short-lived

**Flow:**

```
main (trunk)
 | short-fix
 | quick-feature
```

**Used in:**

Big tech companies, high automation environments

**Pros:**

- Continuous integration
- Fast feedback
- No long-lived branches

**Cons:**

- Requires discipline
- Requires CI/CD maturity

---

## 🎯 Strategy Selection Answers

### Startup shipping fast:

✅ **GitHub Flow**

---

### Large team with scheduled releases:

✅ **GitFlow**

---

### Open-source projects:

Mostly use:

✅ **GitHub Flow**

(Example: most popular OSS repos use PR → main model)

---

## 🧠 Real DevOps Rule Summary

- `reset` = rewrite history (local only)
- `revert` = safe undo (shared branches)
- `merge` = preserve history
- `rebase` = rewrite history cleanly
- `stash` = context switching
- `cherry-pick` = selective commit transfer
- `squash` = clean history
- branching strategy = team scale decision

---
