# Day 14 – Networking Fundamentals & Hands-on Checks

## Goal

Build practical troubleshooting confidence with core Linux networking commands and concepts.

Focus: fast diagnosis, repeatable checks, real-world flow.

---

## Quick Concepts

### OSI vs TCP/IP (in my words)

- **OSI (7 layers):** Physical, Data Link, Network, Transport, Session, Presentation, Application
- **TCP/IP (4 layers):** Link, Internet, Transport, Application

**Mapping idea:**

- OSI L1–L2 → TCP/IP Link
- OSI L3 → TCP/IP Internet
- OSI L4 → TCP/IP Transport
- OSI L5–L7 → TCP/IP Application

---

### Protocol placement

- **IP** → Network / Internet layer
- **TCP/UDP** → Transport layer
- **HTTP/HTTPS** → Application layer
- **DNS** → Application layer

**Example:**

> curl https://example.com = Application (HTTP) → Transport (TCP) → Internet (IP) → Link
> 

---

## Hands-on Checklist

### Identity

Command:

```bash
hostname -I
# or
ip addr show

```
Observation:

- My IP: 192.168.171.128
<img width="516" height="46" alt="image" src="https://github.com/user-attachments/assets/916d83e8-37a5-4c2d-8266-a94d7d146eab" />

---

### Reachability

```bash
ping <target>

```

Observation:

- Latency: 243.786 ms
- Packet loss: 0% packet loss

<img width="981" height="508" alt="image" src="https://github.com/user-attachments/assets/482d4a70-a910-4351-b2fa-cb49b214528a" />

---

### Path

```bash
traceroute <target>
# or
tracepath <target>

```
<img width="777" height="72" alt="image" src="https://github.com/user-attachments/assets/a7c072d8-9d58-45b3-9aff-0e48e985d2db" />

---

### Ports / Services

```bash
ss -tulpn
# or
netstat -tulpn

```
<img width="1762" height="307" alt="image" src="https://github.com/user-attachments/assets/c7ecee2b-0a30-4642-afab-1761d1df8b5f" />

---

### Name Resolution

```bash
dig <domain>
# or
nslookup <domain>

```
<img width="707" height="207" alt="image" src="https://github.com/user-attachments/assets/22c26a29-5034-4eec-ba6b-aa2582f0d3dd" />

---

### HTTP Check

```bash
curl -I <http/https-url>

```
<img width="1252" height="245" alt="image" src="https://github.com/user-attachments/assets/dd85d1ae-ef8e-46c5-9ce3-5d33555c9f7e" />

---

### Connection Snapshot

```bash
netstat -an | head

```
<img width="888" height="236" alt="image" src="https://github.com/user-attachments/assets/1d398e55-e07b-4dda-85f6-d0a87ccff1d6" />

---

## Mini Task – Port Probe & Interpret

1. Identify one listening port:

```bash
ss -tulpn

```

Example:

- Service: tcp
- Port: 22
1. Test locally:

```bash
nc -zv localhost <port>
# or
curl -I http://localhost:<port>

```
<img width="753" height="48" alt="image" src="https://github.com/user-attachments/assets/e66ae11f-28b5-4ccf-a1c5-28bf01709bfe" />


---

## Reflection

- Fastest signal command when something is broken:
    
    → ping
    
- If **DNS fails**, inspect layer:
    
    → application layer
    
- If **HTTP 500**, inspect layer:
    
    → application layer
