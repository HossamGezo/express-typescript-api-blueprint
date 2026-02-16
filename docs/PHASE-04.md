# Phase 04: DevOps, Containerization & CI/CD Automation 🐳🤖

## Overview

This final phase is about "Packaging" and "Automating". We ensure the application runs identically in any environment and that every update is tested and deployed automatically.

---

## 1. The Multi-Stage Dockerfile 🏗️

We use a **Multi-stage Build** to create a small, secure, and production-ready image. It separates the "Building" environment from the "Running" environment.

#### ⚠️ The Husky / Docker Conflict

**_"Troubleshooting & Build Optimization"_**

> In our `package.json`, we use **Husky** to manage Git Hooks through the `prepare` script.
>
> During the Docker build process, the `.git` directory is excluded via `.dockerignore`.  
> Since Husky requires the `.git` folder to install hooks, the `prepare` script will fail and crash the build when `npm install` runs inside the container.
>
> To prevent this issue, we use the `--ignore-scripts` flag:
>
> ```bash
> npm install --ignore-scripts
> ```
>
> This ensures that Docker installs only the required dependencies without executing lifecycle scripts like `prepare`, resulting in a stable and successful production build.

### 📋 Dockerfile Implementation:

```dockerfile
# Stage 1: Build Stage (The Kitchen)
# We install everything needed to compile the code
FROM node:22.14.0-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --ignore-scripts
COPY . .
RUN npm run build

# Stage 2: Production Stage (The Dining Room)
# We only take the final results to keep the image small and secure
FROM node:22.14.0-alpine
WORKDIR /app
ENV NODE_ENV=production

# Install only production-needed libraries
COPY package*.json ./
RUN npm install --omit=dev --ignore-scripts

# Copy final artifacts from the builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/public ./public
COPY --from=builder /app/views ./views

# Copy Swagger docs and remove source TS files for security
COPY --from=builder /app/src/modules ./src/modules
RUN find ./src/modules -name "*.ts" -type f -delete

EXPOSE 5001
CMD ["node", "dist/index.js"]
```

---

## 2. Optimizing the Build Context (.dockerignore) 🚫

Before Docker starts building the image, it sends all files in your directory to the Docker daemon (the "Build Context"). The `.dockerignore` file ensures we don't send heavy or sensitive files that aren't needed for the build.

### 📋 .dockerignore Implementation:

```text
# Dependency directories - Prevent copying local OS-specific modules
node_modules/

# Build output - Ensure Docker builds a fresh version
dist/

# Environment variable files - CRITICAL for security
.env
.env.*
!.env.example

# Docker infrastructure files - Not needed inside the container
Dockerfile
.dockerignore
docker-compose*

# Version control history
.git
.github

# Logs and cache files
npm-debug.log*
*.tsbuildinfo

# Git hooks (Development only)
.husky
```

### 🧐 Why is this file essential? (The Pro Logic)

1.  **Build Speed:** Without this file, Docker would try to compress and send your entire `node_modules` folder (which can be 500MB+) to the daemon. By ignoring it, the build starts instantly.
2.  **Cross-Platform Stability:** It prevents copying `node_modules` built on your Mac into the Linux-based Docker image, which would cause "Module Not Found" or architecture mismatch errors.
3.  **Security:** It ensures your private `.env` file is never baked into the Docker Image, preventing potential credential leaks if the image is shared.

---

## 3. Docker CLI: Building & Inspecting 🛠️

To manage your images and containers, use these essential commands:

| Action          | Command                                                      | Purpose                                                                                                                    |
| :-------------- | :----------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------- |
| **Build**       | `docker build --platform linux/amd64 -t bookstore-api .`     | Builds the project into a Docker image. The `--platform` flag ensures compatibility with most cloud servers (linux/amd64). |
| **Run**         | `docker run --rm -p 5001:5001 --env-file .env bookstore-api` | Runs a container from the image to test it locally. Maps the container port to your local machine.                         |
| **Interactive** | `docker run -it --rm bookstore-api sh`                       | **Access the container:** Opens an interactive Linux shell inside the container to inspect files or debug.                 |
| **Push**        | `docker push username/repo:tag`                              | Pushes the image to Docker Hub so it can be deployed online.                                                               |

---

## 3. Orchestration with Docker Compose 🎼

Docker Compose manages multiple services (API + Database) as a single system.

### 📋 docker-compose.yml snippet:

```yaml
services:
  mongodb:
    image: mongo:latest
    container_name: bookstore_db_container
    ports: ["27017:27017"]
    volumes: ["mongo-data:/data/db"] # Persistence for database

  api:
    build: .
    container_name: bookstore_api_container
    ports: ["5001:5001"]
    env_file: [".env"]
    volumes: ["./public/images:/app/public/images"] # Persistence for uploads
    depends_on: ["mongodb"]

volumes:
  mongo-data:
```

```yaml
# This configuration can also be written using the multi-line (block) style
# for better readability and easier scalability.

services:
  mongodb:
    image: mongo:latest
    container_name: bookstore_db_container

    # Expose MongoDB default port
    ports:
      - "27017:27017"

    # Persistence for database data
    volumes:
      - "mongo-data:/data/db"

  api:
    build: .
    container_name: bookstore_api_container

    # Expose API application port
    ports:
      - "5001:5001"

    # Load environment variables from .env file
    env_file:
      - ".env"

    # Persistence for uploaded images
    volumes:
      - "./public/images:/app/public/images"

    # Ensure MongoDB starts before the API
    depends_on:
      - "mongodb"

volumes:
  # Named volume for MongoDB data persistence
  mongo-data:
```

---

## 4. Full Automation with GitHub Actions (CI/CD) 🤖✨

This is the most advanced part of our infrastructure. We have programmed a "Robot" (Workflow) that monitors our code 24/7. Every time you push code to GitHub, this robot automatically verifies quality, builds the Docker image, and pushes it to Docker Hub.

### 📋 The Ultimate CI/CD Workflow (`.github/workflows/deploy.yml`)

```yaml
name: CI/CD Pipeline - Build and Push

# 1. THE TRIGGER: When should the robot start?
on:
  push:
    branches: ["main", "release/**"] # Runs on main or any branch starting with release/

jobs:
  # JOB 1: The Inspector (Quality Control)
  test-and-quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4 # Pulls your code into the virtual server

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22.14.0
          cache: "npm" # Speeds up future runs by caching dependencies

      - run: npm install
      - run: npm run lint        # Ensures code follows style guidelines
      - run: npm run check-types # Ensures there are zero TypeScript errors

  # JOB 2: The Factory (Build & Deploy)
  build-and-push:
    needs: test-and-quality # ⚠️ CRITICAL: Only run if Job 1 passes perfectly
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }} # Hidden credentials
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3 # Enables multi-platform building

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64 # Standard architecture for Cloud servers (Render/AWS)
          tags: hossamgezo/bookstore-api:v2

      - name: Update Docker Hub Description
        uses: peter-evans/dockerhub-description@v4 # Syncs your GitHub README to Docker Hub
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
          repository: hossamgezo/bookstore-api
          short-description: "Professional Bookstore API built with Node.js, TypeScript, and Clean Architecture."
          readme-filepath: ./README.md
```

---

### 🧐 Deep Dive: Why this Workflow is "S-Tier"?

#### 1. The Trigger Strategy (`on`) 🎯

- **`release/**`**: Utilizing the double wildcard (`**`) means the automation monitors any branch prefixed with `release/` (e.g., `release/v2` or `release/prod-ready`). This provides immense flexibility for version management and staging deployments.

#### 2. The Quality Gate (`test-and-quality`) 🛡️

- We don't just deploy code; we enforce excellence. If a single unused variable or a type mismatch is detected, the workflow terminates immediately (`Job 1` fails), and the image build is cancelled. This prevents "broken" or "dirty" code from ever reaching your production registry.

#### 3. The Guard Condition (`needs`) ⛓️

- The **`needs: test-and-quality`** directive acts as a mandatory safety lock. It creates a strict dependency between the two jobs. The "Factory" (Docker Build) will not commence operation until the "Inspector" (Quality Check) approves the integrity of the source code.

#### 4. Environment Parity & Buildx 🐳

- **`platforms: linux/amd64`**: This solves the Apple Silicon (ARM64) architecture conflict permanently. The runner builds the image specifically for the cloud's architecture (AMD64), ensuring it runs flawlessly on Render without "Exec format" or "Invalid Platform" errors.

#### 5. Documentation Sync (`dockerhub-description`) 📚

- A rare professional touch: the workflow automatically synchronizes your local **README.md** with your **Docker Hub** repository overview. This ensures your documentation is always consistent and up-to-date across all platforms with zero manual effort.

---

### 🚀 How to Execute this Masterpiece?

1.  **Generate Token**: Obtain a Personal Access Token (PAT) from Docker Hub settings with `Read, Write, Delete` permissions.
2.  **Set Secrets**: Add `DOCKER_USERNAME` and `DOCKER_PASSWORD` (the token) to your GitHub Repository (Settings -> Secrets and variables -> Actions).
3.  **Push Code**: Once you `git push` to the main or any release branch, navigate to the **Actions** tab on GitHub to watch the automation build your empire!

---