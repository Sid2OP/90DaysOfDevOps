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

- Kernel version and architecture 확인

📸 Output:

<img width="1512" height="43" alt="image" src="https://github.com/user-attachments/assets/9ba089ce-b528-4580-b703-ea6fb832928f" />


---

### Command 2: `cat /etc/os-release`

```bash
cat /etc/os-release
```

**Observation:**

- Confirms OS distribution and version

📸 Output:

![image.png](attachment:8d8152fd-07eb-4c00-8bdd-5a12110ecfc3:image.png)

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

![image.png](attachment:09b9af73-b605-4833-a8c3-1849415232ca:image.png)

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

![image.png](attachment:ed6e02f8-50e0-4218-8888-2449a093b7d3:image.png)

---

### Command 5: `free -h`

```bash
free -h

```

**Observation:**

- Sufficient free memory available
- No memory pressure

📸 Output:

![image.png](attachment:ba0f99d1-680f-45f6-b7e1-c7e134461b0b:image.png)

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

![image.png](attachment:268a39d9-ff75-4135-8905-a6e8ae5b9df7:image.png)

---

### Command 7: `du -sh /var/log`

```bash
du -sh /var/log

```

**Observation:**

- Log directory size reasonable
- No unexpected growth

📸 Output:

![image.png](attachment:67e88995-7204-4cad-90f1-3cadc980296d:image.png)

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

![image.png](attachment:46a970cd-71dd-422f-bc5c-23e8c42e361c:image.png)

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

![image.png](attachment:b578f479-3930-4d39-b45d-c2b61a09609d:image.png)

---

### Command 10: Authentication logs

```bash
tail -n 50 /var/log/auth.log

```

**Observation:**

- Successful logins visible
- No repeated failures

📸 Output:

![image.png](attachment:65fd2d14-d21a-458e-b3eb-46bf113eaeba:image.png)

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
