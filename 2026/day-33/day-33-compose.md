# Day 33 – Docker Compose: Multi-Container Basics

## Task 1: Install & Verify

Check if Docker Compose is installed:

```bash
docker compose version
```

If not installed:

```bash
sudo apt install docker-compose-plugin
```

---

# Task 2: Your First Compose File

Create a project folder:

```bash
mkdir compose-basics
cd compose-basics
```

Create a **docker-compose.yml**

```yaml
version: "3"

services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

Start the container:

```bash
docker compose up
```

Run in background:

```bash
docker compose up -d
```

Access in browser:

```
http://localhost:8080
```

Stop containers:

```bash
docker compose down
```
<img width="1853" height="693" alt="image" src="https://github.com/user-attachments/assets/c18ebd72-5692-49c3-9c51-7305a11136b7" />

# Task 3: Two-Container Setup (WordPress + MySQL)

Create a new **docker-compose.yml**

```yaml
version: "3"

services:

  db:
    image: mysql:5.7
    container_name: wordpress-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress
    container_name: wordpress-site
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  db_data:
```

Start containers:

```bash
docker compose up -d
```

Access WordPress:

```
http://localhost:8080
```

Setup WordPress in browser.

📸 Screenshot: WordPress setup page
<img width="1855" height="845" alt="image" src="https://github.com/user-attachments/assets/69768640-052a-4391-ba67-c20871682786" />
<img width="838" height="198" alt="image" src="https://github.com/user-attachments/assets/99b0be47-b633-4213-857a-cf45a874b1ee" />


## Data Persistence Test

Stop containers:

```bash
docker compose down
```

Start again:

```bash
docker compose up -d
```

Result:

WordPress data **still exists** because MySQL uses a **named volume (`db_data`)**.

---


# Task 4: Docker Compose Commands

Start services in detached mode

```bash
docker compose up -d
```

View running services

```bash
docker compose ps
```

View logs of all services

```bash
docker compose logs
```

View logs of a specific service

```bash
docker compose logs wordpress
```

Stop services without removing

```bash
docker compose stop
```

Remove everything

```bash
docker compose down
```

Rebuild images

```bash
docker compose up --build
```

---

# Task 5: Environment Variables

Create a **.env file**

```
MYSQL_ROOT_PASSWORD=rootpass
MYSQL_DATABASE=wordpress
MYSQL_USER=wpuser
MYSQL_PASSWORD=wppass
```

Update **docker-compose.yml**

```yaml
version: "3"

services:

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
    depends_on:
      - db

volumes:
  db_data:
```

Start services again:

```bash
docker compose up -d
```

Docker Compose automatically loads variables from `.env`.

---

