# Day 11 – File Ownership Challenge (chown & chgrp)

## Task

Master file and directory ownership in Linux.

- Understand file ownership (user and group)
- Change file owner using chown
- Change file group using chgrp
- Apply ownership changes recursively

---

## Task 1: Understanding Ownership

Command:

ls -l

Format:

- rw-r--r-- 1 owner group size date filename

**Owner:** User who owns the file

**Group:** Group associated with the file

**Difference:**

Owner controls the file directly. Group defines which users share group-level permissions.

---

## Task 2: Basic chown Operations

Commands:

```bash
touch devops-file.txt
ls -l devops-file.txt
sudo chown tokyo devops-file.txt
sudo chown berlin devops-file.txt
ls -l devops-file.txt

```

---

screenshot -<img width="853" height="247" alt="task_2" src="https://github.com/user-attachments/assets/bc417f0f-fb30-415d-9efa-377cb50c7533" />


## Task 3: Basic chgrp Operations

Commands:

```bash
touch team-notes.txt
ls -l team-notes.txt
sudo groupadd heist-team
sudo chgrp heist-team team-notes.txt
ls -l team-notes.txt

```

---

screenshot-<img width="823" height="185" alt="task_3" src="https://github.com/user-attachments/assets/7352778d-1244-4c4b-8b53-fe265d9728a1" />


## Task 4: Combined Owner & Group Change

Commands:

```bash
touch project-config.yaml
sudo chown professor:heist-team project-config.yaml

mkdir app-logs
sudo chown berlin:heist-team app-logs
ls -l project-config.yaml app-logs

```

---

screenshot-<img width="957" height="227" alt="task_4" src="https://github.com/user-attachments/assets/446d1dd3-91ff-4df3-84a5-96579398078d" />


## Task 5: Recursive Ownership

Commands:

```bash
mkdir -p heist-project/vault
mkdir -p heist-project/plans
touch heist-project/vault/gold.txt
touch heist-project/plans/strategy.conf
sudo groupadd planners
sudo chown -R professor:planners heist-project/
ls -lR heist-project/

```

---

screenshot-<img width="941" height="462" alt="task_5" src="https://github.com/user-attachments/assets/82614afb-6a06-4c46-948a-4949a46381d9" />


## Task 6: Practice Challenge

Commands:

```bash
sudo useradd tokyo
sudo useradd berlin
sudo useradd nairobi

sudo groupadd vault-team
sudo groupadd tech-team

mkdir bank-heist
touch bank-heist/access-codes.txt
touch bank-heist/blueprints.pdf
touch bank-heist/escape-plan.txt

sudo chown tokyo:vault-team bank-heist/access-codes.txt
sudo chown berlin:tech-team bank-heist/blueprints.pdf
sudo chown nairobi:vault-team bank-heist/escape-plan.txt

ls -l bank-heist/

```

---

screenshot-<img width="1057" height="490" alt="task_6" src="https://github.com/user-attachments/assets/1ee3ef32-42b9-4d84-b73b-b897d98f487c" />


# Documentation Section

## Files & Directories Created

- devops-file.txt
- team-notes.txt
- project-config.yaml
- app-logs/
- heist-project/
- bank-heist/

## Ownership Changes

Example:

- devops-file.txt → user:user → tokyo:heist-team

## Commands Used

- ls -l
- chown
- chgrp
- chown -R
- useradd
- groupadd

## What I Learned

- Ownership controls access security
- Group ownership enables team collaboration
- Recursive ownership is critical for project directories

---

## Troubleshooting

Permission denied? → Use sudo

Group missing? → sudo groupadd groupname

User missing? → sudo useradd username
