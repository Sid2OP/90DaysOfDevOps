# Day 28 – Revision Day

## 🧠 Self-Assessment Checklist

### Linux
- [✓] Navigate file system (ls, cd, cp, mv, rm, mkdir)
- [✓] Manage processes (ps, top, kill, &, fg, bg)
- [✓] systemd (systemctl start/stop/enable/status)
- [✓] Text editing (nano, vim)
- [✓] Monitoring (top, free, df, du)
- [✓] File system hierarchy (/ /etc /var /home /tmp)
- [✓] Users & groups (useradd, userdel, groupadd, passwd)
- [✓] Permissions (chmod numeric & symbolic)
- [✓] Ownership (chown, chgrp)
- [✓] LVM (PV, VG, LV)
- [✓] Networking (ping, curl, ss, netstat, dig, nslookup)
- [✓] DNS, IP, subnets, ports

### Shell Scripting
- [✓] Variables, arguments, input
- [✓] if/elif/else, case
- [✓] for, while, until loops
- [✓] Functions
- [✓] Text tools (grep, awk, sed, sort, uniq)
- [✓] Error handling (set -euo pipefail, trap)
- [✓] Cron jobs

### Git & GitHub
- [✓] Init, add, commit, log
- [✓] Branching
- [✓] Push/Pull
- [✓] Clone vs Fork
- [✓] Merge (FF vs merge commit)
- [✓] Rebase
- [✓] Stash
- [✓] Cherry-pick
- [✓] Squash merge
- [✓] Reset vs Revert
- [✓] GitFlow / GitHub Flow / Trunk-based
- [✓] GitHub CLI


---

# ⚡ Quick-Fire Answers (Clean Version)

Here they are again cleanly for memory practice:

- `chmod 755 script.sh` → rwx r-x r-x
- Process vs Service → Process = running program, Service = managed background process
- Port 8080 → `ss -tulnp | grep 8080`
- `set -euo pipefail` → fail fast, safe scripts
- `reset --hard` vs `revert` → destructive vs safe
- Branching strategy (5 devs, weekly) → GitHub Flow / Trunk-based
- `git stash` → temporary save of uncommitted work
- Cron 3 AM → `0 3 * * * script.sh`
- `fetch` vs `pull` → download vs download+merge
- LVM → flexible dynamic storage management

---

## 📘 Teach It Back Section

### Example: Explaining Git Branching

Git branching allows multiple developers to work on different features at the same time without affecting the main codebase.
Each branch is like a separate timeline of the project. Developers create feature branches, work independently, 
and then merge their changes back into the main branch. This prevents conflicts, keeps production code stable, and enables parallel development. 
Branching is essential for team collaboration, CI/CD pipelines, and release management.

---
