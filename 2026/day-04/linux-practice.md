Day 04 – Linux Processes, Services & Logs (Practice Notes)
1️⃣ Process Commands
Command 1: ps aux

Purpose: View all running processes on the system.

ps aux


What I learned:

Shows PID, CPU, memory usage, and command name

Useful to quickly find running processes

📸 Screenshot / Output:
<img width="1316" height="722" alt="image" src="https://github.com/user-attachments/assets/6107ea2b-fcb9-452b-be43-bc6023ce35a9" />


Command 2: top

Purpose: Real-time monitoring of system processes.

top


What I learned:

Live CPU and memory usage

Helps identify high resource consumption

📸 Screenshot / Output:
<img width="1486" height="742" alt="image" src="https://github.com/user-attachments/assets/889f3bca-2291-4a4b-b42c-d24b3e8502e0" />

2️⃣ Service Commands (systemd)
Command 3: systemctl status sshd

Purpose: Check status of SSH service.

systemctl status ssh


What I observed:

Service state (active/running)

Main PID and recent logs

📸 Screenshot / Output:
<img width="1205" height="427" alt="image" src="https://github.com/user-attachments/assets/6dbd654a-e54c-4ba2-99ff-39f718874347" />

Command: systemctl list-units
Purpose

List all active units currently managed by systemd.

Command Used
systemctl list-units

What I learned

Displays active and loaded units

Includes services, sockets, mounts, and targets

Shows unit state (active, running, exited)

Why it is useful

Quickly see what is running on the system

Helps identify failed or unexpected services

Useful during system troubleshooting

📸 Screenshot / Output:
<img width="1612" height="752" alt="image" src="https://github.com/user-attachments/assets/da782c85-2e57-43ea-82e2-2094d3925926" />


3️⃣ Log Commands
Command 5: journalctl -u ssh

Purpose: View logs for a specific service.

journalctl -u ssh


What I learned:

Shows authentication and service-related logs

Useful when a service fails

📸 Screenshot / Output:
<img width="1102" height="116" alt="image" src="https://github.com/user-attachments/assets/65514b50-db07-4acf-adf3-a817db5262e2" />




