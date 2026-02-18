# **Day 18 – Shell Scripting: Functions & Slightly Advanced Concepts**

**Challenge Tasks**

**Task 1: Basic Functions**

1. Create `functions.sh` with:
    - A function `greet` that takes a name as argument and prints `Hello, <name>!`
    - A function `add` that takes two numbers and prints their sum
    - Call both functions from the script

  code
  <img width="605" height="301" alt="image" src="https://github.com/user-attachments/assets/1ed921be-40c7-43c2-a6b9-fba951699123" />

  output
  <img width="753" height="72" alt="image" src="https://github.com/user-attachments/assets/93108398-e884-40ac-9ec9-0f801c9c1ebd" />

---

**Task 2: Functions with Return Values**

1. Create `disk_check.sh` with:
    - A function `check_disk` that checks disk usage of `/` using `df -h`
    - A function `check_memory` that checks free memory using `free -h`
    - A main section that calls both and prints the results

code
<img width="633" height="412" alt="image" src="https://github.com/user-attachments/assets/09c17b0a-f578-4014-85b4-a2ba48b07a86" />

output
<img width="1147" height="205" alt="image" src="https://github.com/user-attachments/assets/1bcf1016-3818-483c-a9e1-77f4721459be" />

---

**Task 3: Strict Mode — `set -euo pipefail`**

1. Create `strict_demo.sh` with `set -euo pipefail` at the top
2. Try using an **undefined variable** — what happens with `set -u`?
3. Try a command that **fails** — what happens with `set -e`?
4. Try a **piped command** where one part fails — what happens with `set -o pipefail`?

**Document:** What does each flag do?

- `set -e` →
- `set -u` →
- `set -o pipefail` →

  code
  <img width="662" height="315" alt="image" src="https://github.com/user-attachments/assets/963d8bb9-e3d7-451d-b10d-0893e73fd614" />

  output
set -e → Exit script immediately if any command fails
set -u → Exit script if an undefined variable is used
set -o pipefail → Pipeline fails if any command in pipe fails

---

**Task 4: Local Variables**

1. Create `local_demo.sh` with:
    - A function that uses `local` keyword for variables
    - Show that `local` variables don't leak outside the function
    - Compare with a function that uses regular variables

code
<img width="787" height="397" alt="image" src="https://github.com/user-attachments/assets/d89acbf5-ff38-4cd7-b5a1-b1ac8c6faebc" />

output
<img width="802" height="111" alt="image" src="https://github.com/user-attachments/assets/08e576af-2e30-45b9-94bc-5ba57a23bcbe" />


---

**Task 5: Build a Script — System Info Reporter**

Create `system_info.sh` that uses functions for everything:

1. A function to print **hostname and OS info**
2. A function to print **uptime**
3. A function to print **disk usage** (top 5 by size)
4. A function to print **memory usage**
5. A function to print **top 5 CPU-consuming processes**
6. A `main` function that calls all of the above with section headers
7. Use `set -euo pipefail` at the top

code
<img width="967" height="797" alt="image" src="https://github.com/user-attachments/assets/93365ede-70b1-41e2-a091-c6c32d2d8d25" />
<img width="870" height="497" alt="image" src="https://github.com/user-attachments/assets/fbf97d18-9882-4aee-b8ee-a756c9ffc6ac" />

output
<img width="1127" height="778" alt="image" src="https://github.com/user-attachments/assets/5822775d-137d-463d-9131-9e7a0328627a" />


