# Day 30 – Docker Images & Container Lifecycle

> **Date:** March 2, 2026  
> **Topic:** Understanding Docker Images, Layers, and the Container Lifecycle

---

## 🧠 Core Concepts

### Images vs Containers

| Concept | Description |
|--------|-------------|
| **Image** | A read-only template used to create containers. Like a class in OOP. |
| **Container** | A running (or stopped) instance of an image. Like an object instantiated from a class. |
| **Registry** | A storage system for images (e.g., Docker Hub). |

**Analogy:** An image is a recipe. A container is the dish you made from it. You can make many dishes from the same recipe, and changing the dish doesn't alter the recipe.

---

## Task 1: Docker Images

### Pull Images from Docker Hub

```bash
docker pull nginx
docker pull ubuntu
docker pull alpine
```

**Expected output (nginx example):**
```
Using default tag: latest
latest: Pulling from library/nginx
2d429b9e73a6: Pull complete
...
Status: Downloaded newer image for nginx:latest
```

### List All Images

```bash
docker images
# or
docker image ls
```

**Example output:**
```
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    a6bd71f48f68   2 weeks ago    187MB
ubuntu       latest    bf3dc08bfed0   4 weeks ago    77.9MB
alpine       latest    1d34ffeaf190   3 weeks ago    7.79MB
```

### Ubuntu vs Alpine — Why the Size Difference?

| Image | Size | Reason |
|-------|------|--------|
| `ubuntu` | ~78MB | Based on full Ubuntu GNU/Linux — includes bash, apt, glibc, many system utilities |
| `alpine` | ~8MB | Based on musl libc + BusyBox — a minimal Linux distro designed for small footprint |

**Key differences:**
- Alpine uses `musl libc` instead of `glibc` — much smaller C library
- Alpine uses `BusyBox` which combines many Unix tools into one small binary
- Ubuntu includes many preinstalled packages and a full package manager ecosystem
- Alpine's package manager is `apk` (Alpine Package Keeper)

**Use Alpine when:** You need a small base for production containers  
**Use Ubuntu when:** You need compatibility with many GNU tools or apt packages

### Inspect an Image

```bash
docker image inspect nginx
```

**Information you can see:**
- `Id` — full SHA256 image ID
- `RepoTags` — name and tag (e.g., `nginx:latest`)
- `Created` — when the image was built
- `Architecture` — e.g., `amd64` or `arm64`
- `Os` — operating system (e.g., `linux`)
- `RootFS.Layers` — SHA256 hashes of all image layers
- `Config.Env` — environment variables set in the image
- `Config.Cmd` / `Config.Entrypoint` — default command when container starts
- `Config.ExposedPorts` — ports declared in the Dockerfile
- `Config.Volumes` — declared volumes
- `GraphDriver` — storage driver info (e.g., overlay2)

```bash
# Filter for specific fields using --format
docker image inspect nginx --format='{{.Architecture}}'
docker image inspect nginx --format='{{json .Config.Env}}'
```

### Remove an Image

```bash
# Remove by name
docker rmi alpine

# Remove by image ID
docker rmi 1d34ffeaf190

# Force remove (even if tagged multiple times)
docker rmi -f alpine
```

> ⚠️ You cannot remove an image if a container (even stopped) is using it. Remove the container first.

---

## Task 2: Image Layers

### View Image History

```bash
docker image history nginx
```

**Example output:**
```
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
a6bd71f48f68   2 weeks ago    CMD ["nginx" "-g" "daemon off;"]                0B        buildkit...
<missing>      2 weeks ago    STOPSIGNAL SIGQUIT                              0B
<missing>      2 weeks ago    EXPOSE 80                                       0B
<missing>      2 weeks ago    COPY /etc/nginx /etc/nginx                      35.4kB
<missing>      2 weeks ago    RUN /bin/sh -c apt-get update && apt-get...     97.8MB
<missing>      2 weeks ago    ENV NGINX_VERSION=1.25.3                        0B
<missing>      2 weeks ago    FROM debian:bookworm-slim                        74.8MB
```

### Understanding Layers

**What are layers?**

A Docker image is made up of a stack of read-only **layers**. Each instruction in a `Dockerfile` that modifies the filesystem creates a new layer:

| Dockerfile Instruction | Creates a Layer? | Size |
|----------------------|------------------|------|
| `FROM` | ✅ Yes | Base OS size |
| `RUN apt-get install ...` | ✅ Yes | Size of installed packages |
| `COPY ./app /app` | ✅ Yes | Size of copied files |
| `CMD` | ✅ Yes (metadata only) | 0B |
| `ENV` | ✅ Yes (metadata only) | 0B |
| `EXPOSE` | ✅ Yes (metadata only) | 0B |

**Why 0B layers?** Instructions like `CMD`, `ENV`, `EXPOSE`, `LABEL` only change image metadata — they don't write files to disk, so their layer has no storage size.

**Why does Docker use layers?**

1. **Caching** — If a layer hasn't changed, Docker reuses the cached version. This makes builds dramatically faster.
2. **Sharing** — Multiple images that share the same base (e.g., all using `debian:bookworm-slim`) only store that base layer once on disk.
3. **Efficiency** — When you `docker pull` an image, Docker only downloads layers you don't already have.
4. **Immutability** — Read-only layers make images safe and reproducible.

**Layer caching in practice:**
```dockerfile
# ✅ Good — dependencies layer is cached unless package.json changes
COPY package.json .
RUN npm install
COPY . .

# ❌ Bad — any source change invalidates the install cache
COPY . .
RUN npm install
```

---

## Task 3: Container Lifecycle

### Full Lifecycle Walkthrough

```bash
# 1. CREATE — create container without starting it
docker create --name lifecycle-test nginx
# State: Created

docker ps -a
# STATUS: Created

# 2. START — start the created container
docker start lifecycle-test
# State: Running

docker ps -a
# STATUS: Up X seconds

# 3. PAUSE — suspend all processes (SIGSTOP)
docker pause lifecycle-test
# State: Paused

docker ps -a
# STATUS: Up X seconds (Paused)

# 4. UNPAUSE — resume processes
docker unpause lifecycle-test
# State: Running

docker ps -a
# STATUS: Up X seconds

# 5. STOP — graceful shutdown (SIGTERM, then SIGKILL after timeout)
docker stop lifecycle-test
# State: Exited (0)

docker ps -a
# STATUS: Exited (0) X seconds ago

# 6. RESTART — stop + start in one command
docker restart lifecycle-test
# State: Running

docker ps -a
# STATUS: Up X seconds

# 7. KILL — forceful shutdown (SIGKILL immediately)
docker kill lifecycle-test
# State: Exited (137)

docker ps -a
# STATUS: Exited (137) X seconds ago

# 8. REMOVE — delete the container
docker rm lifecycle-test

docker ps -a
# Container no longer appears
```

### State Diagram

```
        docker create
[Image] ──────────────► [Created]
                            │
                    docker start
                            │
                            ▼
                        [Running] ◄────────────┐
                         /    \                │
              docker pause    docker stop      │
                   /              \     docker restart
                  ▼               ▼            │
              [Paused]        [Exited] ────────┘
                  │               │
          docker unpause    docker rm
                  │               │
              [Running]       [Deleted]
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Clean exit (normal stop) |
| `137` | Killed with SIGKILL (docker kill or OOM) |
| `143` | Killed with SIGTERM (docker stop) |

---

## Task 4: Working with Running Containers

### Run Nginx in Detached Mode

```bash
docker run -d --name webserver -p 8080:80 nginx
```

- `-d` — detached mode (runs in background)
- `--name webserver` — give it a friendly name
- `-p 8080:80` — map host port 8080 to container port 80

### View Logs

```bash
# View current logs
docker logs webserver

# Follow/stream logs in real time (Ctrl+C to stop)
docker logs -f webserver

# Show last 50 lines + follow
docker logs --tail 50 -f webserver

# Add timestamps
docker logs -t webserver
```

### Exec Into the Container

```bash
# Open an interactive bash shell
docker exec -it webserver bash

# Once inside, look around:
ls /etc/nginx/          # nginx config files
ls /usr/share/nginx/html/  # default web root
cat /etc/nginx/nginx.conf
exit
```

### Run a Single Command Without Entering

```bash
# Run a command and get output, then exit
docker exec webserver ls /etc/nginx
docker exec webserver cat /etc/nginx/nginx.conf
docker exec webserver nginx -v
```

### Inspect the Container

```bash
docker inspect webserver
```

**Find specific info:**

```bash
# IP Address
docker inspect webserver --format='{{.NetworkSettings.IPAddress}}'
# Example output: 172.17.0.2

# Port mappings
docker inspect webserver --format='{{json .NetworkSettings.Ports}}'
# Example: {"80/tcp":[{"HostIp":"0.0.0.0","HostPort":"8080"}]}

# Mounts
docker inspect webserver --format='{{json .Mounts}}'

# All key network info
docker inspect webserver --format='
  IP: {{.NetworkSettings.IPAddress}}
  Gateway: {{.NetworkSettings.Gateway}}
  Ports: {{json .NetworkSettings.Ports}}'
```

---

## Task 5: Cleanup

### Stop All Running Containers

```bash
# List running container IDs and stop them all
docker stop $(docker ps -q)

# Or using newer syntax
docker container stop $(docker container ls -q)
```

### Remove All Stopped Containers

```bash
# Remove all stopped containers
docker container prune

# Or using subshell
docker rm $(docker ps -aq -f status=exited)
```

### Remove Unused Images

```bash
# Remove dangling images (untagged, not referenced by any container)
docker image prune

# Remove ALL unused images (not just dangling)
docker image prune -a
```

### Check Docker Disk Usage

```bash
docker system df
```

**Example output:**
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         2         452MB     234MB (51%)
Containers      3         1         1.2MB     800kB (66%)
Local Volumes   2         1         45MB      22MB (48%)
Build Cache     12        0         89MB      89MB
```

### Nuclear Cleanup (Remove Everything Unused)

```bash
# Remove all stopped containers, unused networks, dangling images, build cache
docker system prune

# Also remove unused images (not just dangling)
docker system prune -a

# Also remove unused volumes (careful — data loss!)
docker system prune -a --volumes
```

---

## 📝 Key Takeaways

1. **Images are immutable blueprints** — containers are ephemeral instances
2. **Layers enable caching** — order Dockerfile instructions from least to most frequently changing
3. **Alpine is tiny** (~8MB) because it uses musl libc + BusyBox instead of full GNU userland
4. **Container states:** Created → Running → Paused/Stopped → Removed
5. **`docker stop` is graceful** (SIGTERM timeout then SIGKILL); **`docker kill` is immediate** (SIGKILL)
6. **`docker exec`** lets you run commands in already-running containers
7. **`docker inspect`** is your best friend for debugging — IP, ports, mounts, config
8. **`docker system prune`** is the easiest way to reclaim disk space

---

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
