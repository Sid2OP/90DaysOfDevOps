# 🐳 Day 32 – Docker Volumes & Networking

---

# Task 1: The Problem (Ephemeral Containers)

### MySQL

```
docker run-d \
--name mysql-volume \
-eMYSQL_ROOT_PASSWORD=secret \
-v mysqldata:/var/lib/mysql \
  mysql:8
```

## 🔍 Enter container

```bash
docker exec -it pg-test psql -U postgres
```
Inside Postgres:

```sql
CREATE DATABASE devops;
\c devops
CREATE TABLE users(id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO users(name) VALUES ('Sid');
SELECT * FROM users;
```
<img width="777" height="242" alt="image" src="https://github.com/user-attachments/assets/e9c89b5c-eaa8-4e9f-9eee-e02d0131dd57" />

## 🛑 Remove container

```bash
docker stop pg-test
docker rm pg-test
```

---

## 🚀 Run new container

```bash
docker run -d \
  --name pg-test \
  -e POSTGRES_PASSWORD=secret \
  postgres
```

Enter again:

```bash
docker exec -it pg-test psql -U postgres
```

Check database:

```sql
\l
```

❌ Your `devops` database is gone.

---
## 🧠 Why?

Containers are **ephemeral**.

Data was stored inside container’s writable layer.

When container was removed → data deleted.

---

# Task 2: Named Volumes

## 📦 Create volume

```bash
docker volume create pgdata
```

Verify:

```bash
docker volume ls
```

---

## 🐳 Run Postgres with volume

```bash
docker run -d \
  --name pg-volume \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres
```

---

## ➕ Add data again

```bash
docker exec -it pg-volume psql -U postgres
```

Inside:

```sql
CREATE DATABASE devops;
\q
```

---

## 🛑 Remove container

```bash
docker stop pg-volume
docker rm pg-volume
```

---

## 🔁 Run new container with SAME volume

```bash
docker run -d \
  --name pg-volume2 \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres
```

Check again:

```bash
docker exec -it pg-volume2 psql -U postgres
```

```sql
\l
```

✅ `devops` database is still there.

---
<img width="1170" height="607" alt="image" src="https://github.com/user-attachments/assets/9a8154fb-fada-4db5-9ae1-8da278053d5a" />

---

# Task 3: Bind Mounts

## 📁 Create host folder

```bash
mkdir my-html
echo "<h1>Hello from Host!</h1>" > my-html/index.html
```

---

## 🐳 Run Nginx with bind mount

```bash
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  -v $(pwd)/my-html:/usr/share/nginx/html \
  nginx
```

Open:

```
http://localhost:8080
```

---

## ✏ Edit host file

```bash
echo "<h1>Updated Live!</h1>" > my-html/index.html
```

Refresh browser.

✅ Page updates instantly.

---

## 🧠 Named Volume vs Bind Mount

| Named Volume | Bind Mount |
| --- | --- |
| Managed by Docker | Managed by host |
| Safer | Direct host access |
| Good for DB data | Good for dev |
| Portable | Less portable |

---
<img width="902" height="217" alt="image" src="https://github.com/user-attachments/assets/fb39a903-65ce-4798-af33-c52f3455b3d3" />

# Task 4: Docker Networking Basics

## 📡 List networks

```bash
docker network ls
```

You’ll see:

- bridge
- host
- none

---

## 🔍 Inspect bridge

```bash
docker network inspect bridge
```

---

## 🐳 Run two containers on default bridge

```bash
docker run -dit --name c1 ubuntu bash
docker run -dit --name c2 ubuntu bash
```

---

## ❓ Can they ping by name?

Enter c1:

```bash
docker exec -it c1 bash
```

Install ping:

```bash
apt update && apt install -y iputils-ping
```

Try:

```bash
ping c2
```

❌ Fails (default bridge doesn’t support name resolution automatically)

---
<img width="571" height="42" alt="image" src="https://github.com/user-attachments/assets/fab906b3-8ee6-40f4-aea8-6c799796396f" />

## ✅ Ping by IP

Get c2 IP:

```bash
docker inspect c2 | grep IPAddress
```

Then:

```bash
ping <IP>
```

✔ Works

---

<img width="1202" height="327" alt="image" src="https://github.com/user-attachments/assets/dfee4165-f688-4844-9757-2f2cbc9139fb" />

# Task 5: Custom Network

## 🌐 Create custom bridge

```bash
docker network create my-app-net
```

---

## 🐳 Run containers on custom network

```bash
docker run -dit --name c3 --network my-app-net ubuntu bash
docker run -dit --name c4 --network my-app-net ubuntu bash
```

---

## 🧪 Test ping by name

Enter c3:

```bash
docker exec -it c3 bash
apt update && apt install -y iputils-ping
ping c4
```

✅ Works!

---

<img width="856" height="157" alt="image" src="https://github.com/user-attachments/assets/f7030daf-6c86-4860-93f1-f98a4c6aea11" />

## 🧠 Why custom network works?

Custom bridge has:

- Built-in DNS
- Automatic name resolution
- Better isolation

Default bridge:

- Legacy
- No automatic DNS

---

# Task 6: Put It Together (Real App Architecture)

## 🌐 Create network

```bash
docker network create app-net
```

---

## 📦 Run database with volume

```bash
docker volume create mysqldata
```

```bash
docker run -d \
  --name mysql-db \
  --network app-net \
  -e MYSQL_ROOT_PASSWORD=secret \
  -v mysqldata:/var/lib/mysql \
  mysql
```

---

## 🚀 Run app container

```bash
docker run -dit \
  --name app \
  --network app-net \
  ubuntu bash
```

Enter app container:

```bash
docker exec -it app bash
```

Install ping:

```bash
apt update && apt install -y iputils-ping
```

Test:

```bash
ping mysql-db
```

✅ Works by container name.

---

<img width="931" height="203" alt="image" src="https://github.com/user-attachments/assets/b28ef37d-f917-4cdf-8603-bb33743c6741" />

# 🧠 Final Understanding

### Volumes solve:

- Data persistence
- Database durability
- Production storage

### Networking solves:

- Service communication
- Microservices architecture
- Real production design

---

# 🔥 Real-World Architecture Now Looks Like:

```
[ App Container ]  →  talks to  →  [ Database Container ]
          |                              |
      Custom Network                  Named Volume
```

---
