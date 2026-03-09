Day 37 – Docker Revision & Cheat Sheet

# 📄 docker-cheatsheet.md

```markdown
# Docker Cheat Sheet

## Container Commands
docker run IMAGE                # Run container from image
docker run -it IMAGE            # Interactive container
docker run -d IMAGE             # Detached container
docker ps                       # List running containers
docker ps -a                    # List all containers
docker stop CONTAINER_ID        # Stop container
docker start CONTAINER_ID       # Start container
docker rm CONTAINER_ID          # Remove container
docker exec -it CONTAINER bash  # Access running container
docker logs CONTAINER_ID        # View container logs

## Image Commands
docker build -t IMAGE_NAME .    # Build image from Dockerfile
docker images                   # List images
docker pull IMAGE_NAME          # Download image from Docker Hub
docker push IMAGE_NAME          # Push image to Docker Hub
docker tag IMAGE NEW_NAME       # Tag image
docker rmi IMAGE_ID             # Remove image

## Volume Commands
docker volume create NAME       # Create named volume
docker volume ls                # List volumes
docker volume inspect NAME      # Inspect volume
docker volume rm NAME           # Remove volume

## Network Commands
docker network create NAME      # Create network
docker network ls               # List networks
docker network inspect NAME     # Inspect network
docker network connect NET CONT # Connect container to network

## Docker Compose Commands
docker compose up               # Start services
docker compose up -d            # Start in background
docker compose down             # Stop services
docker compose ps               # List compose containers
docker compose logs             # View logs
docker compose build            # Build images

## Cleanup Commands
docker system prune             # Remove unused data
docker container prune          # Remove stopped containers
docker image prune              # Remove unused images
docker volume prune             # Remove unused volumes
docker system df                # Check Docker disk usage

## Dockerfile Instructions
FROM IMAGE                      # Base image
RUN command                     # Execute command during build
COPY src dest                   # Copy files
WORKDIR path                    # Set working directory
EXPOSE PORT                     # Document container port
CMD command                     # Default command
ENTRYPOINT command              # Main container command
```

```markdown
# Docker Cheat Sheet

## Container Commands
docker run IMAGE                # Run container from image
docker run -it IMAGE            # Interactive container
docker run -d IMAGE             # Detached container
docker ps                       # List running containers
docker ps -a                    # List all containers
docker stop CONTAINER_ID        # Stop container
docker start CONTAINER_ID       # Start container
docker rm CONTAINER_ID          # Remove container
docker exec -it CONTAINER bash  # Access running container
docker logs CONTAINER_ID        # View container logs

## Image Commands
docker build -t IMAGE_NAME .    # Build image from Dockerfile
docker images                   # List images
docker pull IMAGE_NAME          # Download image from Docker Hub
docker push IMAGE_NAME          # Push image to Docker Hub
docker tag IMAGE NEW_NAME       # Tag image
docker rmi IMAGE_ID             # Remove image

## Volume Commands
docker volume create NAME       # Create named volume
docker volume ls                # List volumes
docker volume inspect NAME      # Inspect volume
docker volume rm NAME           # Remove volume

## Network Commands
docker network create NAME      # Create network
docker network ls               # List networks
docker network inspect NAME     # Inspect network
docker network connect NET CONT # Connect container to network

## Docker Compose Commands
docker compose up               # Start services
docker compose up -d            # Start in background
docker compose down             # Stop services
docker compose ps               # List compose containers
docker compose logs             # View logs
docker compose build            # Build images

## Cleanup Commands
docker system prune             # Remove unused data
docker container prune          # Remove stopped containers
docker image prune              # Remove unused images
docker volume prune             # Remove unused volumes
docker system df                # Check Docker disk usage

## Dockerfile Instructions
FROM IMAGE                      # Base image
RUN command                     # Execute command during build
COPY src dest                   # Copy files
WORKDIR path                    # Set working directory
EXPOSE PORT                     # Document container port
CMD command                     # Default command
ENTRYPOINT command              # Main container command
```

---

# 📄 day-37-revision.md

```markdown
# Day 37 – Docker Revision

## Self Assessment

| Skill | Status |
|------|------|
Run a container from Docker Hub | Can Do |
List, stop, remove containers/images | Can Do |
Explain image layers and caching | Shaky |
Write Dockerfile from scratch | Can Do |
Explain CMD vs ENTRYPOINT | can do |
Build and tag image | Can Do |
Create and use volumes | Can Do |
Use bind mounts | Can Do |
Create custom networks | Can Do |
Write docker-compose.yml | Can Do |
Use environment variables in compose | can do |
Write multi-stage Dockerfile | can do |
Push image to Docker Hub | Can Do |
Use healthchecks and depends_on | can do |

---

# Quick Fire Answers

## 1. Image vs Container
An image is a read-only template.
A container is a running instance of that image.

## 2. What happens to data when container is removed?
Data stored inside the container filesystem is deleted unless stored in a volume or bind mount.

## 3. How do containers communicate on same network?
They communicate using container names as hostnames via Docker’s internal DNS.

## 4. docker compose down vs docker compose down -v
docker compose down stops and removes containers and networks.
docker compose down -v also removes volumes.

## 5. Why multi-stage builds?
They reduce image size by separating build environment from runtime environment.

## 6. COPY vs ADD
COPY copies files from host to container.
ADD can also extract archives and download URLs.

## 7. What does -p 8080:80 mean?
Maps port 80 inside container to port 8080 on the host.

## 8. How to check Docker disk usage?
docker system df
```
