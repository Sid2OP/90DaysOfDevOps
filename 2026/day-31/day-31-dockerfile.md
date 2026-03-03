# 🐳 Day 31 – Dockerfile: Build Your Own Images

---

# Task 1: Your First Dockerfile

### 📁 Setup

```bash
mkdir my-first-image
cd my-first-image
```

### 📄 Dockerfile

```
FROM ubuntu:latest

RUN apt-get update && apt-get install -y curl

CMD ["echo", "Hello from my custom image!"]
```

### 🏗 Build image

```bash
docker build -t my-ubuntu:v1 .
```

### ▶ Run container

```bash
docker run my-ubuntu:v1
```

### ✅ Output
<img width="777" height="486" alt="image" src="https://github.com/user-attachments/assets/c55a822c-48cb-4d2e-b1a2-3f26ad1c73a3" />

# Task 2: Dockerfile Instructions

### 📄 Dockerfile

```
FROM ubuntu:latest

WORKDIR /app

COPY test.txt .

RUN apt-get update && apt-get install -y curl

EXPOSE 3000

CMD ["bash"]
```

### 📁 Create file

```bash
echo "Hello from container file system" > test.txt
```

### 🏗 Build

```bash
docker build -t instruction-demo:v1 .
```

### ▶ Run

```bash
docker run -it instruction-demo:v1
```

<img width="732" height="178" alt="image" src="https://github.com/user-attachments/assets/a3f61a99-8ea5-4a3c-b95b-e7762f70c956" />

### 🧠 What each does

| Instruction | Meaning |
| --- | --- |
| FROM | Base image |
| RUN | Executes commands at build time |
| COPY | Copies host files into image |
| WORKDIR | Sets default working directory |
| EXPOSE | Documents container port |
| CMD | Default runtime command |

---

# Task 3: CMD vs ENTRYPOINT

### CMD Image

```
FROM ubuntu
CMD ["echo", "hello"]
```

Build:

```bash
docker build -t cmd-test .
```

Run:

```bash
docker run cmd-test
# Output: hello
```

Override:

```bash
docker run cmd-test echo hi
# Output: hi
```

---

### ENTRYPOINT Image

```
FROM ubuntu
ENTRYPOINT ["echo"]
```

Build:

```bash
docker build -t entry-test .
```

Run:

```bash
docker run entry-test hello
# Output: hello
```

Run:

```bash
docker run entry-test devops
# Output: devops
```

---

### 🧠 Notes Answer

**CMD**

- Default command
- Can be fully overridden
- Best for default behavior

**ENTRYPOINT**

- Fixed executable
- Arguments can change
- Best for tools/utilities

✅ Use CMD → flexible containers

✅ Use ENTRYPOINT → fixed-purpose containers (like `kubectl`, `terraform`, `nginx`)

---
<img width="916" height="352" alt="image" src="https://github.com/user-attachments/assets/4eecc827-63c9-4553-a046-a906ebcfe7b4" />
<img width="975" height="417" alt="image" src="https://github.com/user-attachments/assets/f633b78f-e4ce-4344-9b99-8386c62f99ea" />

# Task 4: Simple Web App Image

### 📄 index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title>My Docker Website</title>
</head>
<body>
  <h1>Hello from Docker!</h1>
  <p>My first custom Nginx image</p>
</body>
</html>
```

### 📄 Dockerfile

```
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
```

### 🏗 Build

```bash
docker build -t my-website:v1 .
```

### ▶ Run

```bash
docker run -d -p 8080:80 my-website:v1
```

### 🌐 Browser

```
http://localhost:8080
```
<img width="631" height="325" alt="image" src="https://github.com/user-attachments/assets/d31b0c88-3936-4c3b-8e7c-b50e3df23fb8" />

---

# Task 5: .dockerignore

### 📄 .dockerignore

```
node_modules
.git
*.md
.env
```

### 🏗 Build

```bash
docker build -t ignore-test:v1 .
```

### 🧠 Why this matters

- Smaller images
- Faster builds
- Better security
- Clean production images

---

# Task 6: Build Optimization (Caching)

### ❌ Bad order

```
COPY . .
RUN npm install
```

### ✅ Optimized order

```
COPY package.json .
RUN npm install
COPY . .
```

### 🧠 Why layer order matters

Docker builds in **layers**:

- If a layer changes → all layers after it rebuild
- Stable layers first = cache reuse
- Faster builds
- Faster CI/CD
- Less bandwidth
- Lower cloud costs

---

# 🧠 Notes Answers Section

### Why does layer order matter?

Because Docker caches layers.

If an early layer changes → everything after rebuilds.

Optimized order = faster builds + better CI/CD performance.

---
