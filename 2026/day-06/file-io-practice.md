# Day 06 – Linux Fundamentals: Read and Write Text Files

## Objective

Practice basic file read/write operations using fundamental Linux commands.

---

## Step 1: Create a file

```bash
touch notes.txt

```

**What it does:**

Creates an empty file named `notes.txt`.

---

## Step 2: Write text using redirection (`>`)

```bash
echo"Line 1: Linux file I/O practice" > notes.txt

```

**What it does:**

Writes the first line to the file (overwrites if file exists).

---

## Step 3: Append text using redirection (`>>`)

```bash
echo"Line 2: Using output redirection" >> notes.txt

```

**What it does:**

Appends a new line without overwriting existing content.

---

## Step 4: Write and display using `tee`

```bash
echo"Line 3: Writing with tee command" |tee -a notes.txt

```

**What it does:**

Displays output on screen and appends it to the file.

---

## Step 5: Read the full file

```bash
cat notes.txt

```

**What it does:**

Displays the entire contents of the file.

---

## Step 6: Read first part of the file

```bash
head -n 2 notes.txt

```

**What it does:**

Shows the first 2 lines of the file.

---

## Step 7: Read last part of the file

```bash
tail -n 2 notes.txt

```

**What it does:**

Shows the last 2 lines of the file.

---

## Final Notes

- Practiced creating, writing, appending, and reading files
- Used `>`, `>>`, `cat`, `head`, `tail`, and `tee`
- Commands are simple and reusable for daily Linux work
  
  OUTPUT
  
  <img width="1236" height="388" alt="image" src="https://github.com/user-attachments/assets/f61c9a6f-f365-4977-b1f4-d32e33658bf3" />
