# day-34-compose-advanced.md

## Day 34 – Docker Compose: Real-World Multi-Container Apps

## Overview

In this task, I built a production-like multi-container application using Docker Compose. The stack includes:

- Web Application (Python Flask)
- PostgreSQL Database
- Redis Cache

The goal was to understand service dependencies, healthchecks, restart policies, custom Dockerfiles, networks, volumes, and scaling.

---

# Project Structure

```
day34-compose/
│
├── docker-compose.yml
│
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

---

# Task 1 – Build Your Own App Stack

## Flask App

### app/app.py

```python
from flask import Flask
import psycopg2
import redis
import os

app = Flask(__name__)

@app.route("/")
def hello():
    db_host = os.getenv("DB_HOST")
    redis_host = os.getenv("REDIS_HOST")

    return f"Hello from Flask! DB Host: {db_host}, Redis Host: {redis_host}"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

### app/requirements.txt

```
flask
psycopg2-binary
redis
```

---

### app/Dockerfile

```
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

---

# docker-compose.yml

```yaml
version: "3.9"

services:

  web:
    build: ./app
    container_name: flask-app
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DB_HOST: db
      REDIS_HOST: redis
    networks:
      - app-network
    labels:
      project: "day34"
      service: "web"

  db:
    image: postgres:14
    container_name: postgres-db
    restart: always
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: devdb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devuser -d devdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    labels:
      project: "day34"
      service: "database"

  redis:
    image: redis:7
    container_name: redis-cache
    networks:
      - app-network
    labels:
      project: "day34"
      service: "cache"

volumes:
  postgres-data:

networks:
  app-network:
```

---

# Task 2 – depends_on & Healthchecks

### depends_on

The `web` service depends on the database service.

```
depends_on:
  db:
    condition: service_healthy
```

This ensures:

1. Database container starts
2. Healthcheck verifies database readiness
3. Only then the web container starts

### Database Healthcheck

```
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U devuser -d devdb"]
  interval: 10s
  timeout: 5s
  retries: 5
```

This checks if PostgreSQL is ready to accept connections.

---

# Task 3 – Restart Policies

### restart: always

```
restart: always
```

If the container stops for any reason, Docker automatically restarts it.

### Test

Kill the database container:

```
docker kill postgres-db
```

Docker automatically recreates it.

---

### restart: on-failure

```
restart: on-failure
```

This only restarts the container if it exits with an error.

---

### When to use restart policies

| Policy | Use Case |
| --- | --- |
| always | Critical services like databases |
| on-failure | Applications that might crash due to temporary errors |
| unless-stopped | Services that should run unless manually stopped |

---

# Task 4 – Custom Dockerfiles in Compose

Instead of using a prebuilt image, Docker Compose builds the image using:

```
build: ./app
```

To rebuild after code changes:

```
docker compose up --build
```

This rebuilds the image and restarts the containers.

---

# Task 5 – Named Networks & Volumes

### Named Volume

```
volumes:
  postgres-data:
```

This ensures database data persists even if containers are removed.

### Named Network

```
networks:
  app-network:
```

All services communicate within this private network.

---

# Task 6 – Scaling (Bonus)

Command used:

```
docker compose up --scale web=3
```

### Result

Docker created three instances of the web service:

```
flask-app-1
flask-app-2
flask-app-3
```

### Issue with Port Mapping

Port mapping like:

```
5000:5000
```

can only bind once on the host machine.

Therefore, scaling multiple containers causes a **port conflict**.

---

### Why Simple Scaling Breaks

When multiple containers try to bind to the same host port:

```
Host:5000 → Container:5000
```

only one container can use it.

### Real-world Solution

Production systems solve this using:

- Load balancers
- Reverse proxies (NGINX / Traefik)
- Container orchestration (Kubernetes / Docker Swarm)

---

# Commands Used

Start services

```
docker compose up
```

Build and start

```
docker compose up --build
```

Run in background

```
docker compose up -d
```

Stop services

```
docker compose down
```

Scale service

```
docker compose up --scale web=3
```

Check containers

```
docker ps
```

---

<img width="1592" height="112" alt="image" src="https://github.com/user-attachments/assets/180118ef-3c1c-4ada-a49b-7779b25f137c" />

----

<img width="798" height="247" alt="image" src="https://github.com/user-attachments/assets/6131f6b0-a090-4fe9-a7dd-6d4688c8eab9" />
