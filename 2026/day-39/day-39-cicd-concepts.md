# day-39-cicd-concepts.md

## Day 39 – What is CI/CD

CI/CD is a software development practice that automates **building, testing, and deploying applications** whenever code changes are made.

Popular tools that implement CI/CD include **GitHub Actions**, **Jenkins**, **GitLab CI**, and **CircleCI**.

The goal is to deliver software **faster, safer, and with fewer manual steps**.

---

# Task 1 – The Problem

### Scenario

A team of **5 developers** pushes code to the same repository and manually deploys to production.

### What can go wrong?

- Code conflicts between developers
- Bugs introduced without testing
- One developer overwrites another's changes
- Deployment mistakes (wrong version deployed)
- Different environments causing failures

---

### What does **"It works on my machine"** mean?

This means the code **runs correctly on the developer's computer but fails in another environment** such as staging or production.

Common reasons:

- Different OS
- Different dependency versions
- Missing environment variables
- Configuration differences

CI/CD solves this by running code in **consistent automated environments**.

---

### How many times a day can a team safely deploy manually?

Usually **1–2 times per day** because manual deployment is slow and risky.

With CI/CD pipelines, teams can deploy **multiple times per day safely**.

---

# Task 2 – CI vs CD

## Continuous Integration (CI)

Continuous Integration is the practice of **automatically building and testing code whenever developers push changes to a shared repository**.

It helps detect:

- build failures
- test failures
- integration issues

Example:

A developer pushes code → automated tests run using **GitHub Actions**.

---

## Continuous Delivery (CD)

Continuous Delivery means the application is **automatically built, tested, and prepared for release**, but **deployment to production requires manual approval**.

Example:

Code passes CI → deployment is ready → team manually approves release.

---

## Continuous Deployment

Continuous Deployment goes one step further.

Every successful build **is automatically deployed to production without human approval**.

Example:

Code pushed → tests pass → Docker image built → automatically deployed.

Companies like **Netflix** and **Amazon** use similar automated deployment practices.

---

# Task 3 – Pipeline Anatomy

### Trigger

An event that **starts the pipeline**.

Examples:

- Git push
- Pull request
- Schedule
- Manual trigger

---

### Stage

A **logical phase of the pipeline**.

Examples:

- Build
- Test
- Deploy

---

### Job

A **group of steps executed on the same machine** inside a stage.

Example:

A job that installs dependencies and runs tests.

---

### Step

A **single command or action** inside a job.

Example:

```
npm install
npm test
docker build .
```

---

### Runner

The **machine that executes pipeline jobs**.

Example:

A virtual machine provided by **GitHub Actions** or a self-hosted runner.

---

### Artifact

Files produced during the pipeline.

Examples:

- compiled application
- Docker image
- test reports
- build logs

Artifacts can be used in later stages.

---

# Task 4 – Pipeline Diagram

Example pipeline for:

Developer pushes code → test → build Docker image → deploy to staging.

```
Developer
   │
   │ git push
   ▼
GitHub Repository
   │
   ▼
CI/CD Pipeline Triggered
   │
   ├── Stage 1: Build
   │      Install dependencies
   │      Compile application
   │
   ├── Stage 2: Test
   │      Run unit tests
   │      Run integration tests
   │
   ├── Stage 3: Docker Build
   │      Build Docker image
   │      Push to container registry
   │
   └── Stage 4: Deploy
          Deploy container to staging server
```

Tools used in this pipeline may include **Docker**, **GitHub Actions**, and **Kubernetes**.

---

# Task 5 – Explore in the Wild

Example repository explored: **React**

Repository:

https://github.com/facebook/react

Folder checked:

```
.github/workflows/
```

Example workflow file: `ci.yml`

### What triggers it?

Triggered on:

- push
- pull_request

---

### How many jobs does it have?

Multiple jobs such as:

- lint
- test
- build

---

### What does it do?

The workflow:

- installs dependencies
- runs lint checks
- runs automated tests
- verifies builds

This ensures that **new code does not break the project**.

---
