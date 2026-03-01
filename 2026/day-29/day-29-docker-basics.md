# Day 29 – Docker Basics 🐳

## Task 1: What is Docker?

### What is a container and why do we need them?

A container is a lightweight, portable, and isolated environment that packages an application with all its dependencies (libraries, configs, runtime). Containers solve the problem of "it works on my machine" by ensuring the app runs the same everywhere.

**Why we need containers:**

- Environment consistency
- Faster deployments
- Lightweight compared to VMs
- Easy scaling
- Better resource usage

---

### Containers vs Virtual Machines

| Feature | Containers | Virtual Machines |
| --- | --- | --- |
| OS | Share host OS kernel | Each VM has its own OS |
| Size | MBs | GBs |
| Speed | Start in seconds | Start in minutes |
| Performance | Near-native | Overhead due to hypervisor |
| Isolation | Process-level | Hardware-level |

**Simple analogy:**

VM = full house 🏠

Container = room inside a house 🛏️

---

### Docker Architecture

**Components:**

- Docker Client → CLI (docker commands)
- Docker Daemon → Core engine that builds, runs, manages containers
- Docker Image → Read-only template
- Docker Container → Running instance of image
- Docker Registry → Image storage (Docker Hub)

**Flow:**

Docker CLI → Docker Daemon → Pull Image from Registry → Create Container → Run Container

---

### Docker Architecture (in my words)

Docker works like a factory system. The client gives instructions, the daemon does the heavy work, images are blueprints, containers are running machines, and Docker Hub is the warehouse.

---

## Task 2: Install Docker

### Verify Installation

```bash
docker --version
```

### Test Docker
**What happens internally:**

1. Docker checks local image
2. Image not found
3. Pulls from Docker Hub
4. Creates container
5. Runs container
6. Shows output
7. Container exits

---

<img width="1075" height="542" alt="image" src="https://github.com/user-attachments/assets/8d65edbd-5c59-4e3f-bf60-070429bf7f30" />

## Task 3: Run Real Containers

### Run Nginx Container

```bash
docker run -d -p 8080:80 --name mynginx nginx
```

Access in browser:

```
http://localhost:8080
```

---

<img width="1596" height="480" alt="image" src="https://github.com/user-attachments/assets/c765e022-f29b-4cf7-810e-114e28cdc28b" />
<img width="1845" height="836" alt="image" src="https://github.com/user-attachments/assets/aeaaaba2-820a-4eeb-8046-388712cfe986" />

### Run Ubuntu Interactive Container

```bash
docker run -it ubuntu bash
```

Inside container:

```bash
ls
cd /
apt update
```

Exit:

```bash
exit
```

---

<img width="1207" height="177" alt="image" src="https://github.com/user-attachments/assets/933caae2-c6e0-4aaf-8aef-a3b25e677ce8" />

### List Containers

Running:

```bash
docker ps
```

All:

```bash
docker ps -a
```

---

### Stop and Remove

```bash
docker stop mynginx
docker rm mynginx
```

---
<img width="1453" height="512" alt="image" src="https://github.com/user-attachments/assets/b2eeb5b8-09b3-4742-a34e-987457972eda" />

---

## Task 4: Explore

### Detached Mode

```bash
docker run -d nginx
```

Runs in background

---

### Custom Name

```bash
docker run -d --name web nginx
```

---

### Port Mapping

```bash
docker run -d -p 3000:80 nginx
```

---

### Logs

```bash
docker logs web
```

---

### Exec into Running Container

```bash
docker exec -it web bash
```

---

<img width="1537" height="550" alt="image" src="https://github.com/user-attachments/assets/0c60e83b-5bc2-4ee1-8c27-b3afb85b8a00" />

<img width="1365" height="512" alt="image" src="https://github.com/user-attachments/assets/ffcee023-1fb4-4f31-9340-ddf0d6047fe3" />


# DevOps Understanding Summary

- Image = Blueprint
- Container = Running app
- Dockerfile = Build instructions
- Docker Hub = Image registry
- Docker = Container engine

---

# Interview Lines

> Docker enables application portability, consistency, and fast deployment using containerization instead of virtualization.
> 

> Containers share the host OS kernel, making them lightweight and fast compared to VMs.
> 

---

# Real-World Use Case

- Microservices deployment
- CI/CD pipelines
- Cloud-native apps
- Dev/Test environments
- Kubernetes orchestration

---

# Commands Cheat Sheet

```bash
docker --version
docker run
docker ps
docker ps -a
docker stop
docker rm
docker images
docker pull
docker exec
docker logs
docker inspect
```

---
```bash
docker run hello-world
```
