**Core components of linux** ➖DAY 2

![image.png](attachment:4c5ab5ed-dd0a-4543-bf6a-037379f167be:image.png)

**Applications**  -

- End-user programs: browsers, editors, IDEs, servers.
- Built on top of libraries/utilities, using kernel services indirectly.

**Shell** -

- The **shell** is the **user interface** in Linux architecture.
- It acts as a **bridge between the user and the kernel**.
- When you type a command, the shell interprets it and passes it to the operating system for execution.

 **Kernel (The Core)**

- **Definition:** The heart of Linux, responsible for interacting directly with hardware.
- **Functions:**
    - Process management (scheduling, multitasking).
    - Memory management (physical and virtual memory).
    - Device management (drivers for hardware).
    - File system management.

**Init System**

- Traditionally, Linux used **SysV init** (older system).
- Modern Linux distributions use **systemd** as the init system.
- **Init’s job:** Start and manage all other processes after the kernel is up.

**systemd**

- **systemd** is the default init system in most modern Linux distros (Ubuntu, Fedora, Arch, RHEL).
- It runs in **user space** and is responsible for:
- Bootstrapping the system after the kernel loads.
- Starting services (networking, logging, graphical environment).
- Managing daemons (background processes).
- Handling dependencies between services.
- Providing parallel startup (faster boot times).
- Offering tools like `systemctl` to control services

**How systemd Works**

1. **Kernel boots** and starts `systemd` as process **PID 1**.
2. `systemd` reads configuration files (unit files).
3. It starts required services in parallel (e.g., networking, login manager).
4. It keeps running to monitor and restart services if they fail.
5. Provides commands like:
- `systemctl start`
- `systemctl stop`
- `systemctl status`

🌀 Major Process States in Linux

1. **Running (R)**

- The process is either:
    - **Currently executing** on the CPU, or
    - **Ready to run**, waiting for CPU time.
- Only one process per CPU core can be truly running at a given instant.

2. **Sleeping (S)**

- The process is **waiting for an event** (like input/output).
- Two types:
- **Interruptible Sleep (S):** Can be woken up by signals (e.g., waiting for user input).
- **Uninterruptible Sleep (D):** Cannot be interrupted until the event finishes (e.g., waiting for disk I/O).

3. **Stopped (T)**

- The process has been **paused**.
- Usually happens when:
- A signal like `SIGSTOP` or `Ctrl+Z` is sent.
- Debuggers stop the process for inspection.

4. **Zombie (Z)**

- The process has **finished execution**, but its entry still exists in the process table.
- Occurs when:
    - The parent hasn’t read the child’s exit status yet.
- It’s a "dead" process waiting to be cleaned up.

5. **Idle / Waiting**

- Special state for kernel processes that wait for specific conditions.
- Example: waiting for hardware interrupts.

5 commands 

1. **`ps`**

- **Purpose:** Displays information about active processes.
- **Example:**

ps aux

- Shows all running processes with details like PID, CPU usage, and memory.

2. **`top`**

- **Purpose:** Real-time view of system processes and resource usage.
- **Example:**

top

- Interactive display of CPU, memory, and process activity.

3. **`kill`**

- **Purpose:** Terminates a process by its PID.
- **Example:**

kill -9 1234

- Forcefully kills process with PID 1234.

4. **`uptime`**

- **Purpose:** Shows how long the system has been running, along with load averages.
- **Example:**

uptime

- Output includes current time, uptime duration, and system load.

5. **`free`**

- **Purpose:** Displays memory usage (RAM and swap).
- **Example:**

free -h

- Shows memory statistics in human-readable format.
