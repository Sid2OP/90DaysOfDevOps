# 📄 `shell_scripting_cheatsheet.md`

👉 Tells OS which interpreter to use to run the script.

# 🐚 Shell Scripting Cheat Sheet (DevOps Reference Guide)

---

# 📌 Quick Reference Table

| Topic | Key Syntax | Example | Explanation |
| --- | --- | --- | --- |
| Variable | VAR="value" | NAME="DevOps" | Stores data in memory |
| Argument | $1, $2 | ./script.sh arg1 | Access CLI arguments |
| If | if [ cond ]; then | if [ -f file ]; then | Conditional execution |
| For loop | for i in list; do | for i in 1 2 3; do | Loop over values |
| Function | name() { } | greet() { echo "Hi"; } | Reusable logic block |
| Grep | grep pattern file | grep -i "error" log.txt | Search text |
| Awk | awk '{print $1}' | awk -F: '{print $1}' file | Column processing |
| Sed | sed 's/a/b/' | sed -i 's/foo/bar/g' file | Text replacement |

---

# 🧱 Task 1: Basics

## Shebang

```bash
#!/bin/bash
```

## Running a script

```bash
chmod +x script.sh
./script.sh
bash script.sh
```

👉 Makes script executable and runs it.

## Comments

```bash
# single line comment
echo "Hi"  # inline comment
```

👉 Used for documentation and readability.

## Variables

```bash
VAR="value"
echo $VAR
echo "$VAR"
echo '$VAR'
```

👉 Store and access data; quotes control variable expansion.

## User input

```bash
read NAME
```

👉 Takes input from terminal into a variable.

## Command-line arguments

```bash
$0  # script name
$1  # first argument
$#  # argument count
$@  # all arguments
$?  # last command exit code
```

👉 Used to pass dynamic data into scripts.

---

# 🔀 Task 2: Operators & Conditionals

## String operators

```bash
=   !=   -z   -n
```

👉 Compare strings and check empty/non-empty values.

## Integer operators

```bash
-eq -ne -lt -gt -le -ge
```

👉 Compare numeric values safely.

## File test operators

```bash
-f file   # regular file
-d dir    # directory
-e file   # exists
-r file   # readable
-w file   # writable
-x file   # executable
-s file   # not empty
```

👉 Validate file system states before operations.

## if syntax

```bash
if [ condition ]; then
   command
elif [ condition ]; then
   command
else
   command
fi
```

👉 Decision-making logic in scripts.

## Logical operators

```bash
&&  ||  !
```

👉 Combine conditions and control flow.

## Case

```bash
case $VAR in
  start) echo "Start";;
  stop) echo "Stop";;
  *) echo "Unknown";;
esac
```

👉 Clean multi-condition branching.

---

# 🔁 Task 3: Loops

## for (list)

```bash
for i in 1 2 3; do echo $i; done
```

👉 Iterate over predefined values.

## for (C-style)

```bash
for ((i=0;i<5;i++)); do echo $i; done
```

👉 Counter-based looping.

## while

```bash
while read line; do echo $line; done < file.txt
```

👉 Loop while condition is true.

## until

```bash
until [ -f done.txt ]; do sleep 1; done
```

👉 Loop until condition becomes true.

## Loop control

```bash
break
continue
```

👉 Control loop execution.

## Loop files

```bash
for f in *.log; do echo $f; done
```

👉 Batch file operations.

## Loop command output

```bash
ps aux | while read line; do echo $line; done
```

👉 Process command output line-by-line.

---

# 🧩 Task 4: Functions

## Define

```bash
func() { echo "Hi"; }
```

👉 Create reusable logic blocks.

## Call

```bash
func
```

👉 Execute function.

## Arguments

```bash
add() { echo $(( $1 + $2 )); }
```

👉 Pass data into functions.

## Return values

```bash
return 0      # status code
echo "data"   # actual value
```

👉 `return` = status, `echo` = data output.

## Local variables

```bash
local VAR="value"
```

👉 Prevent variable leakage outside function.

---

# 🧪 Task 5: Text Processing

## grep

```bash
grep -i "error" file
grep -r "fail" /var/log
grep -n "text" file
grep -v "info" file
grep -E "err|fail" file
```

👉 Search and filter text.

## awk

```bash
awk '{print $1}' file
awk -F: '{print $1}' /etc/passwd
```

👉 Column-based data processing.

## sed

```bash
sed 's/old/new/g' file
sed -i 's/foo/bar/g' file
sed '5d' file
```

👉 Stream editing and transformations.

## cut

```bash
cut -d: -f1 /etc/passwd
```

👉 Extract columns by delimiter.

## sort

```bash
sort file
sort -n file
sort -r file
sort -u file
```

👉 Order and organize data.

## uniq

```bash
uniq
uniq -c
```

👉 Remove duplicates and count.

## tr

```bash
tr 'a-z' 'A-Z'
tr -d '0-9'
```

👉 Character translation/deletion.

## wc

```bash
wc -l
wc -w
wc -c
```

👉 Count lines, words, characters.

## head/tail

```bash
head -n 10 file
tail -n 10 file
tail -f logfile
```

👉 View file start/end and live logs.

---

# ⚙️ Task 6: Useful One-Liners

```bash
# Delete old files
find /logs -type f -mtime +7 -delete
# Auto cleanup

# Count lines in logs
wc -l *.log
# Log analysis

# Replace string
sed -i 's/old/new/g' *.conf
# Mass config updates

# Check service
systemctl is-active nginx
# Health check

# Disk alert
df -h | awk '$5+0 > 80 {print $0}'
# Disk monitoring

# Live error filter
tail -f app.log | grep -i error
# Real-time debugging

# CSV parsing
cut -d, -f1 data.csv
# Data extraction
```

---

# 🛡 Task 7: Error Handling & Debugging

## Exit codes

```bash
exit 0
exit 1
echo $?
```

👉 Control success/failure state.

## Strict mode

```bash
set -e
set -u
set -o pipefail
```

👉 Prevent silent failures.

## Debug mode

```bash
set -x
```

👉 Trace script execution line-by-line.

## Trap

```bash
trap 'echo "Cleanup"; rm temp.txt' EXIT
```

👉 Auto cleanup on script exit.

---
