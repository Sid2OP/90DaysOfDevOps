**Task 1: Your First Script**

1. Create a file `hello.sh`
2. Add the shebang line `#!/bin/bash` at the top
3. Print `Hello, DevOps!` using `echo`
4. Make it executable and run it
<img width="387" height="132" alt="image" src="https://github.com/user-attachments/assets/6ed2801a-74fb-4b36-81f3-b05eee7a3058" />
output
<img width="648" height="38" alt="image" src="https://github.com/user-attachments/assets/3d82bbe3-de78-4262-a578-1dec6a03561c" />

**Task 2: Variables**

1. Create `variables.sh` with:
    - A variable for your `NAME`
    - A variable for your `ROLE` (e.g., "DevOps Engineer")
    - Print: `Hello, I am <NAME> and I am a <ROLE>`

<img width="697" height="231" alt="image" src="https://github.com/user-attachments/assets/ef5c1274-31d2-4d4c-9036-ae2f5f55d865" />
output
<img width="713" height="47" alt="image" src="https://github.com/user-attachments/assets/f66baa3b-ef46-45c5-9b74-a34ffc3d86f3" />

**Task 3: User Input with read**

1. Create `greet.sh` that:
    - Asks the user for their name using `read`
    - Asks for their favourite tool
    - Prints: `Hello <name>, your favourite tool is <tool>`
<img width="637" height="207" alt="image" src="https://github.com/user-attachments/assets/1865ea13-92f1-4774-82e1-be1acee2dc2f" />
  
output
<img width="695" height="92" alt="image" src="https://github.com/user-attachments/assets/d2663028-01e0-41b2-81e7-e4651e759fc6" />

**Task 4: If-Else Conditions**

1. Create `check_number.sh` that:
    - Takes a number using `read`
    - Prints whether it is **positive**, **negative**, or **zero**
  
   <img width="675" height="252" alt="image" src="https://github.com/user-attachments/assets/329f9def-3d81-4a1e-91d9-7ca87579befa" />
   output
   <img width="891" height="297" alt="image" src="https://github.com/user-attachments/assets/9c199301-943a-44c9-a944-cfd4781bba5d" />

2. Create `file_check.sh` that:
    - Asks for a filename
    - Checks if the file **exists** using `f`
    - Prints appropriate message
<img width="643" height="250" alt="image" src="https://github.com/user-attachments/assets/7906b6f7-eada-4c6b-b967-9fcea6c8b4f9" />
  
output
<img width="887" height="62" alt="image" src="https://github.com/user-attachments/assets/263be4d5-5502-4937-9b28-a1fb4f9a14d2" />

**Task 5: Combine It All**

Create `server_check.sh` that:

1. Stores a service name in a variable (e.g., `nginx`, `sshd`)
2. Asks the user: "Do you want to check the status? (y/n)"
3. If `y` — runs `systemctl status <service>` and prints whether it's **active** or **not**
4. If `n` — prints "Skipped."

<img width="962" height="267" alt="image" src="https://github.com/user-attachments/assets/8983f99e-6c99-4122-a4a4-f1cc54373264" />
output
<img width="1163" height="336" alt="image" src="https://github.com/user-attachments/assets/81ec2c8d-7203-4114-9a2b-62bbed8fa30e" />






