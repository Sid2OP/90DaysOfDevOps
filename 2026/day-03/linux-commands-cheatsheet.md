day 3

---

## ⚙️ Process Management Commands

| Command | Purpose | Example |
| --- | --- | --- |
| `ps aux` | Show all running processes | `ps aux | grep firefox` |
| `top` | Real-time process monitoring | `top` |
| `htop` | Interactive process viewer (better than `top`) | `htop` |
| `kill <PID>` | Terminate a process by PID | `kill -9 1234` |
| `nice` / `renice` | Set or change process priority | `renice -n 5 -p 1234` |
| `jobs` | List background jobs in current shell | `jobs` |
| `fg %1` | Bring job to foreground | `fg %1` |

---

## 📂 File System Commands

| Command | Purpose | Example |
| --- | --- | --- |
| `ls -l` | List files with details | `ls -lh` |
| `cd` | Change directory | `cd /var/log` |
| `pwd` | Show current directory | `pwd` |
| `cp` | Copy files/directories | `cp file.txt /backup/` |
| `mv` | Move or rename files | `mv old.txt new.txt` |
| `rm` | Remove files | `rm file.txt` |
| `find` | Search for files | `find /home -name "*.txt"` |
| `du -sh` | Show disk usage of directory | `du -sh /var/log` |
| `df -h` | Show free disk space | `df -h` |

---

## 🌐 Networking Troubleshooting Commands

| Command | Purpose | Example |
| --- | --- | --- |
| `ping <host>` | Test connectivity | `ping google.com` |
| `traceroute <host>` | Show route packets take | `traceroute google.com` |
| `netstat -tulnp` | Show active connections & listening ports | `netstat -tulnp` |
| `ss -tulwn` | Modern replacement for netstat | `ss -tulwn` |
| `ifconfig` / `ip addr` | Show network interfaces | `ip addr show` |
| `nslookup <domain>` | DNS lookup | `nslookup example.com` |
| `dig <domain>` | Detailed DNS query | `dig example.com` |
| `curl -I <url>` | Fetch HTTP headers | `curl -I https://example.com` |
| `wget <url>` | Download files from web | `wget https://example.com/file.zip` |
