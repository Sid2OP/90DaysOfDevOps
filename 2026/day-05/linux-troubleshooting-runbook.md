Linux Troubleshooting Runbook DAY-5 
Target Service / Process

Service: sshd
Why chosen: Critical for remote access and commonly troubleshot

Environment Basics
Command 1: uname -a
uname -a


Observation:

Kernel version and architecture
📸 Output:

<img width="1512" height="43" alt="image" src="https://github.com/user-attachments/assets/b2b06c5b-d276-4ecb-87b3-7693011ea94f" />


Command 2:
cat /etc/os-release
cat /etc/os-release


Observation:

Confirms OS distribution and version

📸 Output:
<img width="866" height="308" alt="image" src="https://github.com/user-attachments/assets/51094668-e557-4044-b150-7993dea99774" />

Filesystem Sanity Check
Command 3: 
Create test directory & file

mkdir /tmp/runbook-demo

cp /etc/hosts /tmp/runbook-demo/hosts-copy

ls -l /tmp/runbook-demo



Observation:

Filesystem is writable and responding normally

📸 Output:
<img width="602" height="105" alt="image" src="https://github.com/user-attachments/assets/2a8e78ae-d36d-48ec-9065-ec03c55bd900" />

Snapshot: CPU & Memory
Command 4: top
top


Observation:

CPU and memory usage within normal range

No abnormal spikes observed
📸 Output:
<img width="1037" height="410" alt="image" src="https://github.com/user-attachments/assets/5d48ab97-5b2a-4c78-9d05-ee2040c00850" />

Command 5: 
free -h
free -h


Observation:

Sufficient free memory available

No memory pressure

📸 Output:
<img width="927" height="91" alt="image" src="https://github.com/user-attachments/assets/184361d8-b028-4cbd-8507-602d63161ccf" />

Snapshot: Disk & IO
Command 6:
df -h
df -h


Observation:

Disk usage under safe limits

No partitions nearing full capacity

📸 Output:
<img width="805" height="180" alt="image" src="https://github.com/user-attachments/assets/4796ed70-a016-4b80-b4a1-8394cc887dd5" />

Command 7:
du -sh /var/log
du -sh /var/log


Observation:

Log directory size reasonable

No unexpected growth

📸 Output:
<img width="766" height="141" alt="image" src="https://github.com/user-attachments/assets/f0d08746-3973-41cc-b425-8cb4efdfe3d8" />

Snapshot: Network
Command 8:
Check listening ports
ss -tulpn | grep sshd


Observation:

SSH is listening on expected port

No port conflicts detected

📸 Output:
<img width="1210" height="67" alt="image" src="https://github.com/user-attachments/assets/4f86235a-7464-4e2c-8e4f-6559905c7b5d" />

Logs Reviewed
Command 9:Service logs
journalctl -u sshd -n 50


Observation:

No recent errors

📸 Output:
<img width="716" height="66" alt="image" src="https://github.com/user-attachments/assets/071ad144-e8a0-42ff-9593-83a58933accf" />

Command 10: Authentication logs

tail -n 50 /var/log/auth.log


Observation:

Successful logins visible

No repeated failures

📸 Output:
<img width="1587" height="588" alt="image" src="https://github.com/user-attachments/assets/26af62fc-31c5-4d55-9bd7-27d7903bf610" />

Quick Findings

Service is active and stable

Resource usage is normal

No disk, memory, or network bottlenecks

Logs show no critical errors

If This Worsens (Next Steps)

Restart service and monitor logs:

systemctl restart sshd


Increase logging verbosity for SSH

Capture deeper diagnostics (strace, lsof, or enable debug logs)







## Snapshot: CPU & Memory

### Command 4: `top`

```bash
top
```

**Observation:**

- CPU and memory usage within normal range
- No abnormal spikes observed

📸 Output:








