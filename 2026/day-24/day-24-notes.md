# ✅ Day 24 — Advanced Git (Commands Only)

---

## 🔀 Task 1: Git Merge — Hands-On

### Create `feature-login` and add commits

```bash
git switch main
git checkout -b feature-login

echo "login v1" > login.txt
git add login.txt
git commit -m "Add login feature v1"

echo "login v2" >> login.txt
git add login.txt
git commit -m "Update login feature"
```

### Merge into `main`

```bash
git switch main
git merge feature-login
```
<img width="1033" height="651" alt="image" src="https://github.com/user-attachments/assets/19187107-15ce-432b-b9a4-8e0f0875f490" />

---

### Create `feature-signup` + commit on main before merge

```bash
git checkout -b feature-signup

echo "signup v1" > signup.txt
git add signup.txt
git commit -m "Add signup feature"

git switch main
echo "main change" > main.txt
git add main.txt
git commit -m "Main branch update"
```

### Merge feature-signup

```bash
git merge feature-signup
```

### View history

```bash
git log --oneline --graph --all
```

---
<img width="1110" height="728" alt="image" src="https://github.com/user-attachments/assets/82321cbc-b1af-49c1-9006-618893a4f9fc" />

---

## 🔁 Task 2: Git Rebase — Hands-On

### Create feature branch + commits

```bash
git switch main
git checkout -b feature-dashboard

echo "dash v1" > dashboard.txt
git add dashboard.txt
git commit -m "Dashboard v1"

echo "dash v2" >> dashboard.txt
git add dashboard.txt
git commit -m "Dashboard v2"
```

### Move main ahead

```bash
git switch main
echo "main update" >> main.txt
git add main.txt
git commit -m "Main moved ahead"
```

### Rebase

```bash
git switch feature-dashboard
git rebase main
```

### View history

```bash
git log --oneline --graph --all
```

---
<img width="1312" height="791" alt="image" src="https://github.com/user-attachments/assets/a81cfe61-20f1-4ba5-b866-02ae514d66df" />

## 🧬 Task 3: Squash vs Regular Merge

### Squash merge

```bash
git checkout -b feature-profile

echo "p1" > profile.txt
git add profile.txt
git commit -m "Profile commit 1"

echo "p2" >> profile.txt
git add profile.txt
git commit -m "Profile commit 2"

echo "p3" >> profile.txt
git add profile.txt
git commit -m "Profile commit 3"

git switch main
git merge --squash feature-profile
git commit -m "Add profile feature (squashed)"
```
<img width="1247" height="790" alt="image" src="https://github.com/user-attachments/assets/72e46e07-7a8a-432c-bd6f-87337f6efaf3" />
<img width="1007" height="487" alt="image" src="https://github.com/user-attachments/assets/7e755366-a576-4865-b4b4-85e2bf5f18d2" />

### Regular merge

```bash
git checkout -b feature-settings

echo "s1" > settings.txt
git add settings.txt
git commit -m "Settings v1"

echo "s2" >> settings.txt
git add settings.txt
git commit -m "Settings v2"

git switch main
git merge feature-settings
```

### Compare history

```bash
git log --oneline --graph --all
```

---
<img width="1112" height="773" alt="image" src="https://github.com/user-attachments/assets/555b3755-eb25-4921-84c8-86322b42f0da" />

## 📦 Task 4: Git Stash — Hands-On

### Create uncommitted changes

```bash
echo "WIP change" >> stash.txt
```

### Try switching branch

```bash
git switch feature-login
```

### Stash changes

```bash
git stash
```

### Switch, work, and return

```bash
git switch main
echo "urgent fix" > urgent.txt
git add urgent.txt
git commit -m "Urgent fix"

git switch feature-login
```

### Apply stash

```bash
git stash pop
```
<img width="1072" height="132" alt="image" src="https://github.com/user-attachments/assets/e6e0bce0-91d1-4b7d-ae04-426d6ffd2207" />
<img width="936" height="220" alt="image" src="https://github.com/user-attachments/assets/d09c9510-827c-4359-844f-2edbb8209caa" />

---

## 🍒 Task 5: Cherry Picking

### Create feature-hotfix + commits

```bash
git checkout -b feature-hotfix

echo "fix1" > hotfix.txt
git add hotfix.txt
git commit -m "Hotfix commit 1"

echo "fix2" >> hotfix.txt
git add hotfix.txt
git commit -m "Hotfix commit 2"

echo "fix3" >> hotfix.txt
git add hotfix.txt
git commit -m "Hotfix commit 3"
```

### View commits

```bash
git log --oneline
```

### Cherry-pick only second commit

```bash
git switch main
git cherry-pick <commit-hash-of-second-commit>
```

### Verify

```bash
git log --oneline --graph
```

---
<img width="978" height="731" alt="image" src="https://github.com/user-attachments/assets/14a52561-b81f-4744-ba66-4aa227a28b9a" />
<img width="1352" height="617" alt="image" src="https://github.com/user-attachments/assets/f5a91499-57a6-4a42-b69d-e0cc9f6d59e7" />
