# Day 09 – Linux User & Group Management Challenge

---

## Objective

Practice real-world Linux user and group management used in DevOps and production environments.

---

# Day 09 Challenge

## Users & Groups Created

### Users:

- tokyo
- berlin
- professor
- nairobi

### Groups:

- developers
- admins
- project-team

---

## Task 1: Create Users

```bash
sudo useradd -m tokyo
sudo passwd tokyo

sudo useradd -m berlin
sudo passwd berlin

sudo useradd -m professor
sudo passwd professor

```

### Verification:

```bash
cat /etc/passwd | grep -E "tokyo|berlin|professor"
ls /home


```
screenshot-
<img width="970" height="152" alt="useradd" src="https://github.com/user-attachments/assets/d7c79cec-3d0a-41bb-ac81-7239cb6bbd5f" />

---

## Task 2: Create Groups

```bash
sudo groupadd developers
sudo groupadd admins

```

### Verification:

```bash
cat /etc/group | grep -E "developers|admins"


```
screenshot-
<img width="912" height="137" alt="groupadd" src="https://github.com/user-attachments/assets/6070c795-01d9-4aa8-8833-955ad9f46819" />

---

## Task 3: Assign Users to Groups

```bash
sudo usermod -aG developers tokyo
sudo usermod -aG developers,admins berlin
sudo usermod -aG admins professor

```

### Verification:

```bash
groups tokyo
groups berlin
groups professor


```
screenshot-
<img width="981" height="231" alt="usermod" src="https://github.com/user-attachments/assets/c2f37744-b04a-4641-b41c-11c9faebfe7f" />

---

## Task 4: Shared Directory

### Create directory:

```bash
sudo mkdir -p /opt/dev-project

```

### Set group owner:

```bash
sudo chgrp developers /opt/dev-project

```

### Set permissions:

```bash
sudo chmod 775 /opt/dev-project

```

### Verification:

```bash
ls -ld /opt/dev-project

```

### Test file creation:

```bash
sudo -u tokyo touch /opt/dev-project/tokyo-file.txt
sudo -u berlin touch /opt/dev-project/berlin-file.txt


```
screenshot-
<img width="1017" height="257" alt="chgrp" src="https://github.com/user-attachments/assets/838fb757-12de-44d9-b36c-466bcc27ebd4" />

---

## Task 5: Team Workspace

### Create user:

```bash
sudo useradd -m nairobi
sudo passwd nairobi

```

### Create group:

```bash
sudo groupadd project-team

```

### Add users to group:

```bash
sudo usermod -aG project-team nairobi
sudo usermod -aG project-team tokyo

```

### Create workspace:

```bash
sudo mkdir -p /opt/team-workspace
sudo chgrp project-team /opt/team-workspace
sudo chmod 775 /opt/team-workspace

```

### Verification:

```bash
ls -ld /opt/team-workspace
groups nairobi

```

### Test:

```bash
sudo -u nairobi touch /opt/team-workspace/nairobi-file.txt


```
screenshot-
<img width="1071" height="373" alt="last" src="https://github.com/user-attachments/assets/c269bc38-036b-4f42-af04-e3ae52d835b7" />

---

## Group Assignments

- tokyo → developers, project-team
- berlin → developers, admins
- professor → admins
- nairobi → project-team

---

## Directories Created

| Directory | Group Owner | Permissions |
| --- | --- | --- |
| /opt/dev-project | developers | 775 |
| /opt/team-workspace | project-team | 775 |

---

## Commands Used

- useradd
- passwd
- groupadd
- usermod -aG
- groups
- mkdir
- chgrp
- chmod
- ls
- cat
- sudo -u

---

## What I Learned

- Linux access control is based on users, groups, and permissions
- Group-based access is essential for team collaboration
- Proper permission design prevents unauthorized access
