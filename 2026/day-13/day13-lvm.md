# Day 13 – Linux Volume Management (LVM)

## 🎯 Goal

Learn Linux LVM to manage storage flexibly: create, extend, and mount logical volumes.

---

---

# 🧪 Challenge Tasks

## Task 1: Check Current Storage

lsblk

pvs

vgs

lvs

df -h

---

## Task 2: Create Physical Volume

pvcreate /dev/loop0

pvs

---

## Task 3: Create Volume Group

vgcreate devops-vg /dev/loop0

vgs

---

## Task 4: Create Logical Volume

lvcreate -L 500M -n app-data devops-vg

lvs

📸 Screenshot space here
<img width="1152" height="667" alt="1" src="https://github.com/user-attachments/assets/35d03a2f-5187-40ed-9a9d-ad332df1ec6e" />


---

## Task 5: Format and Mount

mkfs.ext4 /dev/devops-vg/app-data

mkdir -p /mnt/app-data

mount /dev/devops-vg/app-data /mnt/app-data

df -h /mnt/app-data

📸 Screenshot space here
<img width="1232" height="691" alt="2" src="https://github.com/user-attachments/assets/65ed2d41-151c-47e8-89c8-16aebf43157e" />
<img width="1260" height="595" alt="3" src="https://github.com/user-attachments/assets/a7c63b6b-6856-4cc9-8c04-48d0e1bf6474" />

---

## Task 6: Extend the Volume

lvextend -L +200M /dev/devops-vg/app-data

resize2fs /dev/devops-vg/app-data

df -h /mnt/app-data

📸 Screenshot space here
<img width="1460" height="76" alt="4" src="https://github.com/user-attachments/assets/5692d868-bbf6-42f7-a212-30d9b5dcbdfc" />

---

# 📚 What I Learned

1. LVM allows dynamic resizing of storage without downtime.
2. Storage is abstracted into PV → VG → LV layers.
3. Logical volumes behave like normal disks but are flexible.
