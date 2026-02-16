# Day 15 – Networking Concepts: DNS, IP, Subnets & Ports

## Task 1: DNS – How Names Become IPs

**What happens when you type google.com in a browser?**

- Browser checks local DNS cache and /etc/hosts file.
- If not found, request goes to the configured DNS resolver (ISP / local router).
- Resolver queries DNS hierarchy (Root → TLD → Authoritative server).
- Final IP is returned and browser connects to that IP using TCP/HTTPS.

**DNS Record Types**

- A: Maps domain name to IPv4 address
- AAAA: Maps domain name to IPv6 address
- CNAME: Alias of one domain to another domain
- MX: Mail server for the domain
- NS: Name servers responsible for the domain

**Command Output**

```bash
dig google.com

```

<img width="912" height="457" alt="image" src="https://github.com/user-attachments/assets/eee04a4f-6042-4e00-8491-1d6323ff840b" />

## Task 2: IP Addressing

**What is an IPv4 address?**

- A 32-bit numeric address used to uniquely identify a device on a network, written in dotted-decimal format (e.g., 192.168.1.10).

**Public vs Private IP**

- Public IP example: 8.8.8.8 (Google DNS)
- Private IP example: 192.168.1.10

**Private IP Ranges**

- 10.0.0.0 – 10.255.255.255
- 172.16.0.0 – 172.31.255.255
- 192.168.0.0 – 192.168.255.255

**Command Output**

```bash
ip addr show

```

<img width="1072" height="312" alt="image" src="https://github.com/user-attachments/assets/5627cf18-870c-4e68-bd31-3b3c281b29d9" />

## Task 3: CIDR & Subnetting

**CIDR Meaning**

- /24 in 192.168.1.0/24 means 24 bits are used for the network part and 8 bits for hosts.

**Why we subnet:**

- To reduce broadcast traffic
- Improve security isolation
- Better IP management
- Efficient network segmentation

### CIDR Table

| CIDR | Subnet Mask | Total IPs | Usable Hosts |
| --- | --- | --- | --- |
| /24 | 255.255.255.0 | 256 | 254 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /28 | 255.255.255.240 | 16 | 14 |

---

## Task 4: Ports – The Doors to Services

**What is a port?**

- A logical communication endpoint that identifies a specific service on a system using an IP address.

### Common Ports

| Port | Service |
| --- | --- |
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 53 | DNS |
| 3306 | MySQL |
| 6379 | Redis |
| 27017 | MongoDB |

**Command Output**

```bash
ss -tulpn

```
<img width="1758" height="317" alt="image" src="https://github.com/user-attachments/assets/2f685c6e-d412-4128-9473-62718606ac76" />

Listening ports & services:

- 22 → sshd
- 631 → cups (printing service) *(example)*

## Task 5: Putting It Together

**curl [http://myapp.com:8080](http://myapp.com:8080/) involves:**

- DNS (resolve myapp.com)
- IP routing
- TCP connection
- Port 8080
- HTTP application protocol

**App can’t reach DB at 10.0.1.50:3306 – first checks:**

- Network reachability (ping / traceroute)
- Port access (nc / telnet)
- Firewall rules (ufw / security groups)
- DB service status

---

## What I Learned (3 Key Points)

- DNS, IP, ports, and protocols work together as a full communication stack.
- Subnetting is essential for security, scalability, and performance.
- Troubleshooting networking always follows layers: DNS → IP → Port → Service.
