# Linux Troubleshooting Runbook

## Target Service / Process

**Service:** `sshd`

**Why chosen:** Critical for remote access and commonly troubleshot

---

## Environment Basics

### Command 1: `uname -a`

```bash
uname -a
```

**Observation:**

- Kernel version and architecture 

📸 Output:

<img width="1512" height="43" alt="image" src="https://github.com/user-attachments/assets/87135b5c-6014-43cf-8778-eded87d14e29" />


---

### Command 2: `cat /etc/os-release`

```bash
cat /etc/os-release
```

**Observation:**

- Confirms OS distribution and version

📸 Output:

<img width="866" height="308" alt="image" src="https://github.com/user-attachments/assets/c0b0c3ae-c098-4dc0-98f5-f40f174a87d9" />

---

## Filesystem Sanity Check

### Command 3: Create test directory & file

```bash
mkdir /tmp/runbook-demo
cp /etc/hosts /tmp/runbook-demo/hosts-copy
ls -l /tmp/runbook-demo

```

**Observation:**

- Filesystem is writable and responding normally

📸 Output:

<img width="602" height="105" alt="image" src="https://github.com/user-attachments/assets/b906e396-e3bd-429f-874f-3553c8a7924b" />

---

## Snapshot: CPU & Memory

### Command 4: `top`

```bash
top

```

**Observation:**

- CPU and memory usage within normal range
- No abnormal spikes observed

📸 Output:

<img width="1037" height="410" alt="image" src="https://github.com/user-attachments/assets/28e84ab1-1bea-4831-88cf-0d437b61e878" />


---

### Command 5: `free -h`

```bash
free -h

```

**Observation:**

- Sufficient free memory available
- No memory pressure

📸 Output:

<img width="927" height="91" alt="image" src="https://github.com/user-attachments/assets/860fae3e-f0c1-48ca-a4ae-e0e469e1b3d9" />


---

## Snapshot: Disk & IO

### Command 6: `df -h`

```bash
df -h

```

**Observation:**

- Disk usage under safe limits
- No partitions nearing full capacity

📸 Output:

<img width="805" height="180" alt="image" src="https://github.com/user-attachments/assets/53f9d5b5-7a02-4a08-86e8-426183f2d818" />


---

### Command 7: `du -sh /var/log`

```bash
du -sh /var/log

```

**Observation:**

- Log directory size reasonable
- No unexpected growth

📸 Output:
<img width="766" height="141" alt="image" src="https://github.com/user-attachments/assets/9ed8da02-4226-417d-8f49-4f4d4da23893" />


---

## Snapshot: Network

### Command 8: Check listening ports

```bash
ss -tulpn | grep sshd
```

**Observation:**

- SSH is listening on expected port
- No port conflicts detected

📸 Output:

<img width="1210" height="67" alt="image" src="https://github.com/user-attachments/assets/a0f505c8-e10f-4f88-a310-a2149a81ec5b" />


---

## Logs Reviewed

### Command 9: Service logs

```bash
journalctl -u sshd -n 50

```

**Observation:**

- No recent errors
- Normal authentication activity

📸 Output:

<img width="716" height="66" alt="image" src="https://github.com/user-attachments/assets/3ef8e901-99da-4b72-8a36-6a25ac1f326d" />


---

### Command 10: Authentication logs

```bash
tail -n 50 /var/log/auth.log

```

**Observation:**

- Successful logins visible
- No repeated failures

📸 Output:

<img width="1587" height="588" alt="image" src="https://github.com/user-attachments/assets/ba819311-06ec-4ad2-b481-813d8ab9ff0f" />



---

## Quick Findings

- Service is **active and stable**
- Resource usage is **normal**
- No disk, memory, or network bottlenecks
- Logs show **no critical errors**

---

## If This Worsens (Next Steps)

1. Restart service and monitor logs:
    
    ```bash
    systemctl restart sshd
    
    ```
    
2. Increase logging verbosity for SSH
3. Capture deeper diagnostics (`strace`, `lsof`, or enable debug logs)
