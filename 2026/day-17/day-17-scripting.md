Day 17 – Shell Scripting: Loops, Arguments & Error Handling

**Challenge Tasks**

**Task 1: For Loop**

1. Create `for_loop.sh` that:
    - Loops through a list of 5 fruits and prints each one
code
<img width="547" height="128" alt="image" src="https://github.com/user-attachments/assets/ed45266a-72bd-4319-9b29-a7cfd6b86bf4" />

output
<img width="632" height="128" alt="image" src="https://github.com/user-attachments/assets/1d2170d4-62a9-4106-931f-433b55c6a32b" />

2. Create `count.sh` that:
    - Prints numbers 1 to 10 using a for loop
code
<img width="447" height="122" alt="image" src="https://github.com/user-attachments/assets/4377c6a5-cb53-4304-9485-cfa482591d47" />

output

<img width="853" height="280" alt="image" src="https://github.com/user-attachments/assets/01ca2bc8-f7ac-4209-a12c-611f06011a7d" />


---

**Task 2: While Loop**

1. Create `countdown.sh` that:
    - Takes a number from the user
    - Counts down to 0 using a while loop
    - Prints "Done!" at the end
code
<img width="780" height="272" alt="image" src="https://github.com/user-attachments/assets/7a130bd7-30df-4104-8829-42b11216c3e6" />

output

<img width="852" height="308" alt="image" src="https://github.com/user-attachments/assets/97333c12-7d26-400a-ac25-0f38b1b5127d" />

---

**Task 3: Command-Line Arguments**

1. Create `greet.sh` that:
    - Accepts a name as `$1`
    - Prints `Hello, <name>!`
    - If no argument is passed, prints "Usage: ./greet.sh "
code
<img width="522" height="243" alt="image" src="https://github.com/user-attachments/assets/5873a46b-cb7a-4afd-8dd2-f7d4cd113aa1" />

output
<img width="808" height="51" alt="image" src="https://github.com/user-attachments/assets/3550600a-f491-4a08-a47e-b90ad8c082e2" />

2. Create `args_demo.sh` that:
    - Prints total number of arguments (`$#`)
    - Prints all arguments (`$@`)
    - Prints the script name (`$0`)
code
<img width="597" height="156" alt="image" src="https://github.com/user-attachments/assets/037e5393-1202-48bc-88fa-bd8fccd5e1c9" />

output
<img width="853" height="91" alt="image" src="https://github.com/user-attachments/assets/999c626f-6d46-498d-8061-01e7bce7f637" />

---

**Task 4: Install Packages via Script**

1. Create `install_packages.sh` that:
    - Defines a list of packages: `nginx`, `curl`, `wget`
    - Loops through the list
    - Checks if each package is installed (use `dpkg -s` or `rpm -q`)
    - Installs it if missing, skips if already present
    - Prints status for each package

code
<img width="693" height="302" alt="image" src="https://github.com/user-attachments/assets/53541f98-529c-4f8a-abc5-6ad0f830e666" />\

output
<img width="1041" height="380" alt="image" src="https://github.com/user-attachments/assets/1c324cf1-c9be-4e48-a077-76ce35c68138" />


---

**Task 5: Error Handling**

1. Create `safe_script.sh` that:
    - Uses `set -e` at the top (exit on error)
    - Tries to create a directory `/tmp/devops-test`
    - Tries to navigate into it
    - Creates a file inside
    - Uses `||` operator to print an error if any step fails

code
<img width="851" height="247" alt="image" src="https://github.com/user-attachments/assets/c59d7239-94c1-488c-b6b3-e9c02332a22a" />

output
<img width="755" height="51" alt="image" src="https://github.com/user-attachments/assets/f036d17b-e215-4847-8461-1a75108dd8cd" />

## What I Learned

- Loops automate repetitive tasks efficiently.
- Arguments make scripts dynamic and reusable.
- Error handling prevents broken automation.
- Root checks protect system integrity.


