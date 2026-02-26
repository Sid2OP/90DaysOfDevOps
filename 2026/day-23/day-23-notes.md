# ✅ Day 23 — Task-wise Git Commands

---

## 🌱 Task 1: Understanding Branches

**Commands to run:**

```bash
git branch
git status
git log --oneline
```

---

## 🔀 Task 2: Branching Commands — Hands-On

### List all branches

```bash
git branch
```

### Create a new branch `feature-1`

```bash
git branch feature-1
```

### Switch to `feature-1`

```bash
git checkout feature-1
# OR (modern)
git switch feature-1
```

### Create and switch in one command (`feature-2`)

```bash
git checkout -b feature-2
# OR (modern)
git switch -c feature-2
```

### Switch between branches using `git switch`

```bash
git switch main
git switch feature-1
git switch feature-2
```

### Make a commit on `feature-1`

```bash
git switch feature-1
echo "feature work" >> test.txt
git add test.txt
git commit -m "Add feature work on feature-1"
```

### Switch to `main` and verify commit is not there

```bash
git switch main
git log --oneline
```

### Delete a branch

```bash
git branch -d feature-2
```

---
<img width="1197" height="705" alt="image" src="https://github.com/user-attachments/assets/af1f5c16-2883-4da7-8b95-f9be23cf0298" />

## 🌐 Task 3: Push to GitHub

### Connect local repo to GitHub

```bash
git remote add origin https://github.com/<username>/devops-git-practice.git
```

### Verify remote

```bash
git remote -v
```

### Push main branch

```bash
git push -u origin main
```

### Push feature-1 branch

```bash
git push origin feature-1
```

### Check branches

```bash
git branch -a
```

---
<img width="932" height="357" alt="image" src="https://github.com/user-attachments/assets/30b6b671-9a40-43b6-8e90-6d7983c6024c" />


## ⬇️ Task 4: Pull from GitHub

### Pull changes made on GitHub

```bash
git pull
```

### Compare fetch vs pull

```bash
git fetch
git status
git pull
```

---
<img width="906" height="282" alt="image" src="https://github.com/user-attachments/assets/7dc71425-3aa7-46dc-9779-605832e5f01f" />


## 🔁 Task 5: Clone vs Fork

### Clone a public repo

```bash
git clone https://github.com/<owner>/<repo>.git
```

### Clone your fork

```bash
git clone https://github.com/<your-username>/<repo>.git
```

### Add upstream (original repo)

```bash
git remote add upstream https://github.com/<owner>/<repo>.git
```

### Sync fork with original repo

```bash
git fetch upstream
git merge upstream/main
```

---
