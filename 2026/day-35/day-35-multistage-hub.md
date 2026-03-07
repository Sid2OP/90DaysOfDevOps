# Day 35 – Multi-Stage Builds & Docker Hub

## Overview

Today’s goal was to understand how to create **optimized Docker images** using **multi-stage builds** and distribute them using **Docker Hub**.

Large Docker images slow down builds, deployments, and consume more storage. Multi-stage builds help reduce image size by separating the **build environment** from the **runtime environment**.

---

# Task 1 – The Problem with Large Images

### Simple Node.js App

**app.js**

```jsx
console.log("Hello from Docker Multi-Stage Build!");
```

### package.json

```json
{
  "name": "multistage-demo",
  "version": "1.0.0",
  "main": "app.js"
}
```

---

## Single Stage Dockerfile (Large Image)

```
FROM node:18

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

CMD ["node", "app.js"]
```

---

## Build Image

```bash
docker build -t node-single-stage .
```

Check image size:

```bash
docker images
```

Example output:

| Repository | Tag | Size |
| --- | --- | --- |

<img width="725" height="72" alt="image" src="https://github.com/user-attachments/assets/b932d2b2-40f8-40bb-8bae-6704bd4c8294" />

## Problem

The image is large because it contains:

- Full Node.js runtime
- Build dependencies
- npm cache
- Source files

All these remain in the final image.

---

# Task 2 – Multi-Stage Build

Multi-stage builds allow separating **build stage** and **runtime stage**.

---

## Multi-Stage Dockerfile

```
# Stage 1: Build stage
FROM node:18 AS builder

WORKDIR /app

COPY package.json .
RUN npm install

COPY . .

# Stage 2: Production stage
FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app .

CMD ["node", "app.js"]
```

---

## Build Image

```bash
docker build -t node-multistage .
```

Check size:

```bash
docker images
```

Example:

| Repository | Tag | Size |
| --- | --- | --- |
| node-single-stage | latest | 1GB |
| node-multistage | latest | 120MB |

---

<img width="730" height="87" alt="image" src="https://github.com/user-attachments/assets/73189c18-9722-4bb8-aa2a-66fcc3ca79b1" />

## Why Multi-Stage Image is Smaller

Multi-stage builds reduce image size because:

1. Build tools are removed from the final image.
2. Only required application files are copied.
3. Minimal base images like **alpine** are used.
4. Temporary build dependencies are discarded.

This results in **faster deployments, lower storage usage, and improved security**.

---

# Task 3 – Push Image to Docker Hub

### Step 1 – Login to Docker Hub

```bash
docker login
```

Enter your Docker Hub username and password.

---

### Step 2 – Tag Image

```bash
docker tag node-multistage yourusername/node-multistage:v1
```

Example:

```bash
docker tag node-multistage sid2op/node-multistage:v1
```

---

### Step 3 – Push Image

```bash
docker push yourusername/node-multistage:v1
```

---

### Step 4 – Verify Push

Check Docker Hub repository to confirm the image was uploaded.

---

### Step 5 – Test Pull

Remove local image:

```bash
docker rmi yourusername/node-multistage:v1
```

Pull again:

```bash
docker pull yourusername/node-multistage:v1
```

---

<img width="1093" height="405" alt="image" src="https://github.com/user-attachments/assets/e7ab7678-adfc-4b93-b78e-947d52078e91" />

<img width="937" height="111" alt="image" src="https://github.com/user-attachments/assets/c72eeb10-d192-4592-a7e2-5bbaa50d478d" />

# Task 4 – Docker Hub Repository

After pushing:

1. Open Docker Hub.
2. Navigate to your repository.
3. Add description.

Example description:

> This repository contains a Node.js application built using Docker multi-stage builds to create smaller and more efficient container images.
> 

---

## Understanding Tags

Tags represent versions of images.

Example:

```
node-multistage:v1
node-multistage:v2
node-multistage:latest
```

### Pull Latest

```bash
docker pull yourusername/node-multistage
```

This automatically pulls **latest**.

### Pull Specific Version

```bash
docker pull yourusername/node-multistage:v1
```

This pulls the exact version.

---

# Task 5 – Docker Image Best Practices

### 1️⃣ Use Minimal Base Images

Instead of:

```
FROM ubuntu
```

Use:

```
FROM node:18-alpine
```

Alpine images are much smaller.

---

### 2️⃣ Use Non-Root User

Running containers as root is insecure.

Example:

```
RUN adduser -D appuser
USER appuser
```

---

### 3️⃣ Combine RUN Commands

Bad:

```
RUN apt update
RUN apt install curl
```

Better:

```
RUN apt update && apt install -y curl
```

This reduces image layers.

---

### 4️⃣ Use Specific Tags

Avoid:

```
FROM node:latest
```

Use:

```
FROM node:18-alpine
```

This ensures consistent builds.

---

# Size Comparison

| Build Type | Image Size |
| --- | --- |
| Single Stage | ~1GB |
| Multi-Stage | ~120MB |

---

| node-single-stage | latest | 1GB |

---
