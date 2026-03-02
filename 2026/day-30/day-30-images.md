## 🧠 Core Concepts

### Images vs Containers

An **image** is a read-only template used to create containers — like a class in OOP. A **container** is a running (or stopped) instance of an image — like an object instantiated from that class. A **registry** is a storage system for images (e.g., Docker Hub).

**Analogy:** An image is a recipe. A container is the dish you made from it. You can make many dishes from the same recipe, and changing the dish doesn't alter the recipe.

---

## Task 1: Docker Images

### Pull Images from Docker Hub

```bash
docker pull nginx
docker pull ubuntu
docker pull alpine
```

### List All Images

```bash
docker images
# or
docker image ls
```

<img width="1082" height="782" alt="image" src="https://github.com/user-attachments/assets/5b2fbcd7-db14-4229-92f9-fec941189b8b" />

### Ubuntu vs Alpine — Why the Size Difference?

| Image | Size | Reason |
| --- | --- | --- |
| `ubuntu` | ~78MB | Full Ubuntu GNU/Linux — includes bash, apt, glibc, many system utilities |
| `alpine` | ~8MB | musl libc + BusyBox — minimal Linux distro designed for small footprint |

**Key differences:**

- Alpine uses `musl libc` instead of `glibc` — much smaller C library
- Alpine uses `BusyBox` which combines many Unix tools into one small binary
- Ubuntu includes many preinstalled packages and a full package manager ecosystem
- Alpine's package manager is `apk` (Alpine Package Keeper)

> Use Alpine when you need a small base for production containers. Use Ubuntu when you need compatibility with many GNU tools or apt packages.
> 

### Inspect an Image

```bash
docker image inspect nginx
```

Information you can see:

- `Id` — full SHA256 image ID
- `RepoTags` — name and tag (e.g., `nginx:latest`)
- `Created` — when the image was built
- `Architecture` — e.g., `amd64` or `arm64`
- `Os` — operating system (e.g., `linux`)
- `RootFS.Layers` — SHA256 hashes of all image layers
- `Config.Env` — environment variables set in the image
- `Config.Cmd` / `Config.Entrypoint` — default command when container starts
- `Config.ExposedPorts` — ports declared in the Dockerfile
- `GraphDriver` — storage driver info (e.g., overlay2)

```bash
# Filter for specific fields
docker image inspect nginx --format='{{.Architecture}}'
docker image inspect nginx --format='{{json .Config.Env}}'
```

### Remove an Image

```bash
# Remove by name
docker rmi alpine

# Force remove
docker rmi -f alpine
```

> ⚠️ You cannot remove an image if a container (even stopped) is using it. Remove the container first.
> 

---

<img width="1587" height="690" alt="image" src="https://github.com/user-attachments/assets/d330d20d-9aac-4362-a5aa-be00190f8c03" />

<img width="1057" height="222" alt="image" src="https://github.com/user-attachments/assets/38cbf941-05fe-4e99-83ae-d4be7b33b5f0" />

## Task 2: Image Layers

### View Image History

```bash
docker image history nginx
```

<img width="1225" height="417" alt="image" src="https://github.com/user-attachments/assets/d2595504-6503-4cb7-b235-412ffdcb267d" />

### What Are Layers and Why Does Docker Use Them?

A Docker image is made up of a stack of read-only **layers**. Each Dockerfile instruction that modifies the filesystem creates a new layer.

| Dockerfile Instruction | Creates a Layer? | Size |
| --- | --- | --- |
| `FROM` | ✅ Yes | Base OS size |
| `RUN apt-get install ...` | ✅ Yes | Size of installed packages |
| `COPY ./app /app` | ✅ Yes | Size of copied files |
| `CMD` | ✅ Yes (metadata only) | 0B |
| `ENV` | ✅ Yes (metadata only) | 0B |
| `EXPOSE` | ✅ Yes (metadata only) | 0B |

**Why 0B layers?** Instructions like `CMD`, `ENV`, `EXPOSE` only change image metadata — they don't write files to disk.

**Why does Docker use layers?**

1. **Caching** — Unchanged layers are reused, making builds dramatically faster
2. **Sharing** — Multiple images sharing the same base only store it once on disk
3. **Efficiency** — `docker pull` only downloads layers you don't already have
4. **Immutability** — Read-only layers make images safe and reproducible

---

## Task 3: Container Lifecycle

### Full Lifecycle Walkthrough

```bash
# 1. CREATE — create container without starting it
docker create --name lifecycle-test nginx
# State: Created

# 2. START
docker start lifecycle-test
# State: Running

# 3. PAUSE
docker pause lifecycle-test
# Status: Up X seconds (Paused)

# 4. UNPAUSE
docker unpause lifecycle-test
# State: Running

# 5. STOP — graceful (SIGTERM, then SIGKILL after timeout)
docker stop lifecycle-test
# Status: Exited (0)

# 6. RESTART
docker restart lifecycle-test
# State: Running

# 7. KILL — forceful (SIGKILL immediately)
docker kill lifecycle-test
# Status: Exited (137)

# 8. REMOVE
docker rm lifecycle-test
```
<img width="1333" height="761" alt="image" src="https://github.com/user-attachments/assets/a338fd93-c5d7-464b-be24-3bb73482267e" />

---

## Task 4: Working with Running Containers

### Run Nginx in Detached Mode

```bash
docker run -d --name webserver -p 8080:80 nginx
```

### View Logs

```bash
# View current logs
docker logs webserver

# Follow/stream logs in real time
docker logs -f webserver

# Last 50 lines + follow
docker logs --tail 50 -f webserver

# Add timestamps
docker logs -t webserver
```
<img width="1058" height="247" alt="image" src="https://github.com/user-attachments/assets/4bc791a8-d6c4-4186-8c61-46e76fcfd414" />

### Exec Into the Container

```bash
# Open interactive shell
docker exec -it webserver bash

# Look around inside:
ls /etc/nginx/
ls /usr/share/nginx/html/
cat /etc/nginx/nginx.conf
exit
```
<img width="1592" height="192" alt="image" src="https://github.com/user-attachments/assets/7ffad251-9737-4fdf-89e8-5a69b3ff0cdb" />

### Run a Single Command Without Entering

```bash
docker exec webserver ls /etc/nginx
docker exec webserver nginx -v
```
<img width="1592" height="192" alt="image" src="https://github.com/user-attachments/assets/7ffad251-9737-4fdf-89e8-5a69b3ff0cdb" />

### Inspect the Container

```bash
# Full metadata
docker inspect webserver

# IP Address
docker inspect webserver --format='{{.NetworkSettings.IPAddress}}'
# Example: 172.17.0.2

# Port mappings
docker inspect webserver --format='{{json .NetworkSettings.Ports}}'

# Mounts
docker inspect webserver --format='{{json .Mounts}}'
```

---
<img width="1105" height="177" alt="image" src="https://github.com/user-attachments/assets/c45ef5d3-519c-4a00-9e34-d742dc4377c9" />

## Task 5: Cleanup

### Stop All Running Containers

```bash
docker stop $(docker ps -q)
```

### Remove All Stopped Containers

```bash
docker container prune
```

### Remove Unused Images

```bash
# Dangling images only
docker image prune

# ALL unused images
docker image prune -a
```

### Check Docker Disk Usage

```bash
docker system df
```

<img width="1082" height="727" alt="image" src="https://github.com/user-attachments/assets/1c2263ae-1c0c-45be-85ca-dbb4ec98fa07" />

## Quick Reference Cheat Sheet

```bash
# Images
docker pull <image>           # Download image
docker images                 # List images
docker image inspect <image>  # Inspect image metadata
docker image history <image>  # Show layers
docker rmi <image>            # Remove image
docker image prune -a         # Remove unused images

# Containers
docker create <image>         # Create (don't start)
docker start <container>      # Start
docker stop <container>       # Graceful stop
docker kill <container>       # Force stop
docker pause <container>      # Pause
docker unpause <container>    # Unpause
docker restart <container>    # Restart
docker rm <container>         # Remove

# Inspection
docker ps                     # Running containers
docker ps -a                  # All containers
docker logs -f <container>    # Follow logs
docker exec -it <c> bash      # Shell into container
docker inspect <container>    # Full metadata

# Cleanup
docker system df              # Disk usage
docker system prune -a        # Remove all unused resources
```
