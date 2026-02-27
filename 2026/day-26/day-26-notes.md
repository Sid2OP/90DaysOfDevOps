# 🗓️ Day 26 – GitHub CLI: Manage GitHub from Your Terminal

**(Commands + Practical Understanding)**

> Tool used: **GitHub CLI (`gh`)**
> 

---

# 🔐 Task 1: Install and Authenticate

## Install GitHub CLI

### Ubuntu/Linux

```bash
sudo apt update
sudo apt install gh -y
```

### Verify install

```bash
gh --version
```

---

## Authenticate

```bash
gh auth login
```

Follow prompts:

- GitHub.com
- HTTPS
- Browser login
- Authenticate

---

## Verify login

```bash
gh auth status
```

---

<img width="1356" height="326" alt="image" src="https://github.com/user-attachments/assets/7548a847-3832-4eeb-acad-c237501a7e27" />

### 🧠 What authentication methods does `gh` support?

- Browser-based login
- Personal Access Token (PAT)
- SSH authentication
- OAuth token
- Device code login

➡ This makes `gh` usable in:

- servers
- CI/CD pipelines
- automation scripts
- headless environments

---

# 📦 Task 2: Working with Repositories

## Create repo from terminal

```bash
gh repo create devops-gh-test --public --add-readme
```

### What this does:

- Creates repo on GitHub
- Public repo
- Adds README
- No browser needed

---

## Clone repo using gh

```bash
gh repo clone <username>/devops-gh-test
```

➡ Automatically uses auth + correct protocol

---

## View repo details

```bash
gh repo view
```

---

## List all your repos

```bash
gh repo list
```

---

## Open repo in browser

```bash
gh repo view --web
```

---

## Delete test repo

```bash
gh repo delete devops-gh-test --yes
```

⚠️ Permanent deletion — no recycle bin

---

<img width="1833" height="643" alt="image" src="https://github.com/user-attachments/assets/912d9699-f7a9-4db6-b7df-23a7b97bfb67" />


# 🧾 Task 3: Issues

## Create issue

```bash
gh issue create \
  --title "Login bug" \
  --body "Login fails on invalid token" \
  --label bug
```

---

## List issues

```bash
gh issue list
```

---

## View specific issue

```bash
gh issue view 1
```

---

## Close issue

```bash
gh issue close 1
```

---

<img width="1198" height="351" alt="image" src="https://github.com/user-attachments/assets/e65cf017-e3f5-4418-878d-87507902b5c9" />

# 🔀 Task 4: Pull Requests

## Create branch + commit

```bash
git checkout -b gh-feature
echo "gh test" > gh.txt
git add gh.txt
git commit -m "Add gh test file"
git push -u origin gh-feature
```

---

## Create PR from terminal

```bash
gh pr create --fill
```

➡ Auto-fills title + body from commits

---

## List PRs

```bash
gh pr list
```

---

## View PR details

```bash
gh pr view
```

---

## Merge PR

```bash
gh pr merge --merge
```

---

### 🧠 What merge methods does `gh pr merge` support?

```bash
gh pr merge --merge     # merge commit
gh pr merge --squash    # squash merge
gh pr merge --rebase    # rebase merge
```

---

### 🧠 How would you review someone else’s PR using `gh`?

```bash
gh pr list
gh pr view <number>
gh pr checkout <number>
```

➡ Review code locally

➡ Test locally

➡ Comment:

```bash
gh pr comment <number> --body "Looks good"
```

---

<img width="1371" height="426" alt="image" src="https://github.com/user-attachments/assets/8127968b-dd90-4643-b564-3e4cd06955ff" />

# ⚙️ Task 5: GitHub Actions & Workflows (Preview)

## List workflow runs

```bash
gh run list --repo owner/repo
```

---

## View specific run

```bash
gh run view <run-id> --repo owner/repo
```

---

### 🧠 How `gh run` and `gh workflow` help in CI/CD:

- Monitor pipelines from terminal
- Script pipeline checks
- Auto-fail deployments
- Trigger workflows
- DevOps automation
- Incident pipelines
- GitOps workflows

➡ Example automation idea:

```bash
if gh run list | grep failed; then alert; fi
```

---

# 🧰 Task 6: Useful `gh` Tricks

## Raw GitHub API

```bash
gh api repos/<owner>/<repo>
```

➡ DevOps scripting, automation, monitoring

---

## Gists

```bash
gh gist create file.txt
gh gist list
```

➡ Share logs, configs, scripts

---

## Releases

```bash
gh release create v1.0.0
gh release list
```

➡ CI/CD release automation

---

## Aliases

```bash
gh alias set prs "pr list"
gh prs
```

➡ Productivity boost

---

## Search repos

```bash
gh search repos terraform
```

➡ Discover tools/libs from terminal

---
