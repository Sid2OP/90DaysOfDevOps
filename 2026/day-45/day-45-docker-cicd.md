# Day 45 – Docker Build & Push in GitHub Actions

## Overview

Today you build a **complete CI/CD pipeline** — a `git push` to GitHub automatically builds a Docker image and ships it to Docker Hub. No manual `docker build` or `docker push` steps ever again. This is exactly how production pipelines work.

---

## Task 1: Prepare

### Minimal Dockerfile

If you don't have one from Day 36, add this to your repo root:

```docker
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY scripts/ ./scripts/

RUN pip install --no-cache-dir pytest

CMD ["python", "scripts/test_math.py"]
```

### Secrets Required (from Day 44)

Confirm these are set in `Settings → Secrets and Variables → Actions`:

| Secret | Value |
| --- | --- |
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_TOKEN` | Docker Hub access token |

> Generate a token at: [hub.docker.com](http://hub.docker.com) → Account Settings → Security → New Access Token
> 

---

## Task 2: Build the Docker Image in CI

```yaml
# .github/workflows/docker-publish.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/github-actions-practice

jobs:
  docker:
	    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image (no push yet)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: ${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Verify:** In the Actions tab, check the build step logs. You should see Docker layers being pulled and built, ending with a successful image creation.

---

## Task 3: Push to Docker Hub

Expand the workflow to log in and push with two tags:

```yaml
# .github/workflows/docker-publish.yml
name: Docker build & push

on:
    workflow_dispatch

jobs:
    build-and-push:
        runs-on: ubuntu-latest
        
        steps:
          - name: Code checkout
            uses: actions/checkout@v4

          - name: Login to docker hub
            uses: docker/login-action@v3
            with:
                username: ${{ vars.DOCKERHUB_USER }}
                password: ${{ secrets.DOCKERHUB_PASS }}
          
          - name: Build and push to docker hub
            uses: docker/build-push-action@v6
            with:
                context: .
                push: true
                tags: ${{ vars.DOCKERHUB_USER }}/github-actions-app:${{ github.ref_name}}

        
```

**Verify:** After a push to `main`, go to [hub.docker.com](http://hub.docker.com) → your repository. You should see two tags:

- `latest`
- `sha-abc1234` (7-char commit hash)

---

## Task 4: Only Push on Main

The `push:` condition in the `build-push-action` step handles this:

```yaml
push: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
```

**Behavior by branch:**

| Event | Branch | Build? | Push? |
| --- | --- | --- | --- |
| `push` | `main` | ✅ | ✅ |
| `push` | `feature/xyz` | ✅ | ❌ |
| `pull_request` | any | ✅ | ❌ |

**Test it:** Create a feature branch, push a commit, and check the Actions tab. The workflow runs and builds successfully but the push step is skipped — no new tag appears on Docker Hub.

---

## Task 5: Add a Status Badge

### Get the Badge URL

```
https://github.com/YOUR_USERNAME/github-actions-practice/actions/workflows/docker-publish.yml/badge.svg
```

### Add to [README.md](http://README.md)

```markdown
# github-actions-practice

![Docker Build & Push](https://github.com/YOUR_USERNAME/github-actions-practice/actions/workflows/docker-publish.yml/badge.svg)

Practicing GitHub Actions from Day 42 onwards.
```

Push the updated README — the badge immediately reflects the latest workflow status: green (✅) or red (❌).

---

## Task 6: Pull and Run Locally

```bash
# Pull the image you just pushed
docker pull YOUR_DOCKERHUB_USERNAME/github-actions-practice:latest

# Run it
docker run --rm YOUR_DOCKERHUB_USERNAME/github-actions-practice:latest

# Or run a specific SHA-tagged version (pinned, reproducible)
docker run --rm YOUR_DOCKERHUB_USERNAME/github-actions-practice:sha-abc1234
```

You should see the test output from `test_math.py` running inside the container.

### The Full Journey: `git push` to Running Container

```
git push origin main
       ↓
GitHub detects push event
       ↓
Actions runner provisioned (ubuntu-latest VM)
       ↓
actions/checkout@v4  →  code is cloned onto the runner
       ↓
docker/setup-buildx-action  →  BuildKit enabled for efficient layer caching
       ↓
docker/login-action  →  runner authenticates to Docker Hub with your secrets
       ↓
docker/build-push-action  →  Dockerfile is read, image is built layer by layer
       ↓
Image tagged as :latest and :sha-<hash>  →  pushed to Docker Hub registry
       ↓
Runner VM is destroyed (ephemeral)
       ↓
docker pull username/repo:latest  (anywhere in the world)
       ↓
docker run  →  container running your app
```

Total time: typically **60–120 seconds** from `git push` to image available on Docker Hub.

---

## Complete Final Workflow

```yaml
# .github/workflows/docker-publish.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/github-actions-practice

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Get short SHA
        id: sha
        run: echo "short=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:sha-${{ steps.sha.outputs.short }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Image digest
        run: echo "Pushed image digest: ${{ steps.build.outputs.digest }}"
```

---

## Key Concepts Summary

| Concept | Detail |
| --- | --- |
| `docker/setup-buildx-action` | Enables BuildKit — faster builds, layer caching, multi-platform support |
| `docker/login-action` | Authenticates to Docker Hub using secrets; never hardcode credentials |
| `docker/build-push-action` | Builds image from Dockerfile and pushes to registry in one step |
| Dual tagging | `:latest` for convenience; `:sha-<hash>` for traceability and rollback |
| `cache-from/to: type=gha` | Uses GitHub Actions cache for Docker layers — dramatically speeds up rebuilds |
| Conditional push | `push: ${{ condition }}` — evaluates to `true`/`false`; clean way to gate pushes |
| Status badge | Live indicator in README; reflects last workflow run status instantly |

---

## Quick Reference

```yaml
# Docker login
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_TOKEN }}

# Build and push
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: username/repo:latest

# Short SHA
- id: sha
  run: echo "short=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT
- run: echo ${{ steps.sha.outputs.short }}

# Badge URL
https://github.com/<user>/<repo>/actions/workflows/<file>.yml/badge.svg

# Pull and run
docker pull username/repo:latest
docker run --rm username/repo:latest
```

<img width="1881" height="888" alt="image" src="https://github.com/user-attachments/assets/7df0fe80-a566-4009-8157-3a1ba2f0bfa8" />
