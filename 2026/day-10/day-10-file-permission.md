# Day 10 Challenge – File Permissions & File Operations

---

## Files Created

- devops.txt
- notes.txt
- script.sh
- project/ (directory)

---

## Task 1: Create Files

### Commands

touch devops.txt

echo "Linux permissions practice" > notes.txt

vim script.sh

# inside vim add:

# echo "Hello DevOps"

ls -l

---
<img width="1527" height="646" alt="step 1" src="https://github.com/user-attachments/assets/d4ca7761-e807-4519-a464-ff067e4ba623" />

## Task 2: Read Files

cat notes.txt

vim -R script.sh

head -n 5 /etc/passwd

tail -n 5 /etc/passwd

---
<img width="682" height="330" alt="step 2" src="https://github.com/user-attachments/assets/a8169e10-572b-4c18-acca-eb94b2684cff" />

## Task 3: Permission Understanding

Permission format:

rwxrwxrwx

owner | group | others

Values:

- r = 4
- w = 2
- x = 1

Check:

ls -l devops.txt notes.txt script.sh

### Explanation Example

- rw-r--r-- → owner: read/write, group: read, others: read

---

## Task 4: Modify Permissions

chmod +x script.sh

./script.sh

chmod a-w devops.txt

chmod 640 notes.txt

mkdir project

chmod 755 project

ls -l

---
<img width="936" height="551" alt="image" src="https://github.com/user-attachments/assets/bd9b6368-ef7b-4b30-b5b5-3b0152cf2f94" />

## Task 5: Permission Testing

echo "test" >> devops.txt

# Output: Permission denied

chmod -x script.sh

./script.sh

# Output: Permission denied

---

## Permission Changes

| File | Before | After |
| --- | --- | --- |
| script.sh | rw-r--r-- | rwxr-xr-x |
| devops.txt | rw-r--r-- | r--r--r-- |
| notes.txt | rw-r--r-- | rw-r----- |
| project/ | default | rwxr-xr-x |

---

## Commands Used

- touch
- echo
- vim
- cat
- head
- tail
- chmod
- ls
- mkdir

---

## What I Learned

- Linux permissions control security at file level
- Execute permission is mandatory to run scripts
- Numeric chmod is faster and more precise
- Group permissions are critical for team access
- Most prod errors are permission-related
