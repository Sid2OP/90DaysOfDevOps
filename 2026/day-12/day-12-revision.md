# Day 12 – Breather & Revision (Days 01–11)

## 🎯 Goal

Consolidate Linux fundamentals from Days 01–11 and reinforce core concepts through quick revision + hands-on checks.

---

## 🧠 Mindset & Plan Review (Day 01)

- Initial goal: Build strong Linux fundamentals for DevOps/Cloud engineering
- Focus areas:
    - Command line mastery
    - File system navigation
    - Permissions & ownership
    - Process & service management
    - User/group administration

**Tweak / Update:**

- Add more automation with bash scripting
- Focus more on debugging & troubleshooting workflows
- Start thinking in "production mindset" instead of just commands

---

## ⚙️ Processes & Services (Day 04/05)

**Commands rerun:**

ps aux | head

systemctl status ssh

journalctl -u ssh -n 10

**Observations:**

- Multiple background system processes always running
- SSH service active and running
- Logs show successful login attempts and service activity

---

## 📁 File Skills Practice (Days 06–11)

**Commands used:**

echo "test log" >> demo.txt

chmod 644 demo.txt

chown professor:planners demo.txt

ls -l demo.txt

cp demo.txt backup_demo.txt

mkdir test-folder

**Verified:**

- File creation
- Permission changes
- Ownership changes
- Directory creation

---

## 🧾 Cheat Sheet Refresh (Day 03)

**Top 5 emergency/incident commands:**

- `ls -l` → check permissions & ownership
- `df -h` → disk usage check
- `free -h` → memory usage
- `ps aux` → process monitoring
- `systemctl status <service>` → service health

---

## 👤 User/Group Sanity Check (Day 09/11)

**Scenario recreated:**

sudo adduser testuser

sudo usermod -aG sudo testuser

id testuser

ls -l /home

**Verification:**

- User created successfully
- Group assigned
- Home directory created

---

# 🧪 Mini Self-Check

### 1️⃣ Which 3 commands save you the most time right now, and why?

- `ls -l` → instant permission + ownership visibility
- `systemctl status` → fast service health check
- `journalctl` → debugging + logs in one place

---

### 2️⃣ How do you check if a service is healthy?

**Commands:**

systemctl status <service>

journalctl -u <service> -n 20

ps aux | grep <service>

---

### 3️⃣ How do you safely change ownership and permissions without breaking access?

**Example:**

sudo chown -R professor:planners project/

chmod -R 775 project/

Principle: fix ownership first, then permissions.

---

### 4️⃣ What will you focus on improving in the next 3 days?

- Bash scripting
- Automation mindset
- Error handling
- Log analysis
- Service debugging
- Linux security basics

  <img width="1397" height="785" alt="image" src="https://github.com/user-attachments/assets/5c1dff6e-ced0-427d-9982-4f31173771d8" />
  <img width="1126" height="766" alt="image" src="https://github.com/user-attachments/assets/b7ac878b-db34-422b-9746-ac27c426e8e4" />

