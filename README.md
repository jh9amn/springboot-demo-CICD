# Spring Boot Demo – Task REST API

Java 17+ and Maven required.

    mvn spring-boot:run

Endpoints (`/api/tasks`): GET, GET /{id}, POST, PUT /{id}, DELETE /{id}

    curl -X POST localhost:8080/api/tasks -H "Content-Type: application/json" -d '{"title":"Learn Spring Boot"}'
    curl localhost:8080/api/tasks

H2 console: http://localhost:8080/h2-console (JDBC URL: jdbc:h2:mem:demodb)

Local Host: http://localhost:8081/api/tasks
Host: http://54.252.211.57:8081/api/tasks

---


# Spring Boot CI/CD Deployment Guide (GitHub Actions + Docker + AWS EC2)

This guide documents the end-to-end setup and troubleshooting checklist for deploying a containerized Spring Boot application to an AWS EC2 instance using GitHub Actions.

---

## 1. Prerequisites & AWS EC2 Configuration

### Security Group Rules
Ensure your EC2 instance's Security Group allows inbound traffic on your exposed application port:
* **Type:** Custom TCP
* **Port Range:** `8081` (or your chosen host port)
* **Source:** `0.0.0.0/0` (Anywhere-IPv4)

### EC2 Instance Setup
1. Install Docker on your Ubuntu EC2 instance and add the `ubuntu` user to the `docker` group so commands run without `sudo`:
   ```bash
   sudo apt update && sudo apt install -y docker.io
   sudo usermod -aG docker ubuntu
   newgrp docker

```

2. Verify your EC2 CPU architecture (`x86_64` vs. `aarch64`/ARM64):
```bash
uname -m

```


*(Note: AWS Graviton instances such as `t4g` run on `aarch64` / `linux/arm64`.)*

---

## 2. GitHub Repository Secrets

Navigate to **Settings > Secrets and variables > Actions** in your GitHub repository and configure the following secrets:

| Secret Name | Description |
| --- | --- |
| `DOCKER_USERNAME` | Your Docker Hub username |
| `DOCKER_PASSWORD` | Your Docker Hub Personal Access Token (PAT) or password |
| `EC2_HOST` | Public IPv4 address or Public DNS of your EC2 instance |
| `EC2_USER` | SSH username (e.g., `ubuntu`) |
| `EC2_KEY` | Contents of your private `.pem` SSH key file |

---

## 3. GitHub Actions Workflow (`.github/workflows/main.yml`)

Use Docker Buildx with QEMU to build multi-architecture images (`linux/amd64` and `linux/arm64`). This prevents `no matching manifest` errors when deploying from standard x86 GitHub runners to ARM-based EC2 instances.

```yaml
name: Spring Boot CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  IMAGE_NAME: spring-boot-app

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'

      - name: Build JAR
        run: mvn clean package -DskipTests

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ${{ secrets.DOCKER_USERNAME }}/${{ env.IMAGE_NAME }}:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_KEY }}
          script: |
            docker pull ${{ secrets.DOCKER_USERNAME }}/${{ env.IMAGE_NAME }}:latest
            docker stop spring-boot-app || true
            docker rm spring-boot-app || true
            docker run -d \
              --name spring-boot-app \
              --restart unless-stopped \
              -p 8081:8080 \
              ${{ secrets.DOCKER_USERNAME }}/${{ env.IMAGE_NAME }}:latest
            docker image prune -f

```

---

## 4. Common Pitfalls & Quick Fixes

| Issue / Error | Root Cause | Solution |
| --- | --- | --- |
| **`no matching manifest for linux/arm64/v8`** | GitHub Actions builds `linux/amd64` images by default, which fail on AWS Graviton (`arm64`) instances. | Use `docker/setup-qemu-action`, `docker/setup-buildx-action`, and `platforms: linux/amd64,linux/arm64` in `build-push-action`. |
| **Connection Refused / Timeout on Port `8081**` | Docker was mapped as `-p 8081:8081`, but Spring Boot listens on port `8080` inside the container by default. | Map host port to container port as `-p 8081:8080` (or set `server.port=8081` in `application.properties`). |
| **`docker ps` is empty after deploy** | The container crashed immediately on startup or failed to pull/start during the SSH script. | Run `docker ps -a` and `docker logs spring-boot-app` on EC2, or run `docker run --rm -p 8081:8080 <image>` interactively to view live errors. |
| **Maven cache not working in CI** | Typo in workflow property (`chache: 'maven'`). | Ensure exact spelling: `cache: 'maven'`. |

---

## 5. Backend Debugging Checklist (EC2 SSH)

If the application is unreachable after a deployment, SSH into the EC2 instance and run these checks in order:

1. **Verify running containers and port bindings:**
```bash
docker ps

```


Confirm `PORTS` displays `0.0.0.0:8081->8080/tcp`.
2. **Inspect stopped/crashed containers and logs:**
```bash
docker ps -a
docker logs spring-boot-app --tail 50

```


3. **Test endpoint locally on the EC2 host:**
```bash
curl -I http://localhost:8081

```


*(If this responds with an HTTP status code, the container is healthy and the issue is with the AWS Security Group or OS firewall.)*
4. **Check Ubuntu UFW firewall status:**
```bash
sudo ufw status

```


If active, allow the port:
```bash
sudo ufw allow 8081/tcp

```



```
