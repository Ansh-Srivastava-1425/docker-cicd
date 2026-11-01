# 🐳 Docker + CI/CD Notes (Beginner Friendly)

*My personal notes, written in simple words so future me can understand them quickly.*

## Contents

1. [Docker: what it is](#1-docker-what-it-is)
2. [When to use / when NOT to use Docker](#2-when-to-use--when-not-to-use-docker)
3. [Docker: steps to get started](#3-docker-steps-to-get-started)
4. [Docker boilerplate (copy and edit)](#4-docker-boilerplate-copy-and-edit)
5. [Useful Docker commands](#5-useful-docker-commands)
6. [CI/CD: what it is](#6-cicd-what-it-is)
7. [When to use / when NOT to use CI/CD](#7-when-to-use--when-not-to-use-cicd)
8. [CI/CD: steps to set up](#8-cicd-steps-to-set-up-github-actions)
9. [CI/CD boilerplate (copy and edit)](#9-cicd-boilerplate-copy-and-edit)
10. [Common errors and fixes](#10-common-errors-and-fixes)
11. [My project notes](#11-my-project-notes)
12. [To learn next](#12-to-learn-next)

---

## 1. Docker: what it is

### The problem

"It works on my laptop but crashes on yours."

Different Node versions, missing libraries, different operating systems...

### Docker's fix

Docker packs your app and everything it needs into one sealed box. That box runs the same way on every computer.

Think of it like a lunchbox: the food (your app) and the spoon, salt, and napkin (libraries and settings) are all inside. You carry it anywhere and it works.

### Words you will hear

| Word | Easy meaning |
|---|---|
| **Image** | The recipe or blueprint. Read-only; you build it once. |
| **Container** | A running copy made from an image — the "cooked meal". |
| **Dockerfile** | A text file containing the steps to build your image. |
| **Registry** | An online store for images, such as Docker Hub or GitHub Container Registry. |
| **Volume** | A safe folder that keeps data even after the container is deleted. |
| **Docker Compose** | Starts multiple containers together (for example, an app and a database) using one file. |

**Image vs. container:** An image is like a class; a container is an object made from it. One image can create many containers.

**Container vs. virtual machine:** A VM carries a whole operating system (heavy and slow). A container shares your computer's OS (lighter and starts in seconds).

---

## 2. When to use / when NOT to use Docker

### ✅ Use Docker when

- Your team says "works on my machine" a lot.
- Your app needs a database, cache, etc., and you want a one-command setup (`docker compose up`).
- You want to deploy to a server or cloud.
- You want the same clean environment for development, testing, and production.
- You are running CI/CD pipelines.
- You want to try a tool (MongoDB, Redis, PostgreSQL) without installing it on your laptop.

### ❌ Don't use Docker when

- It's a tiny script or a simple static page. Docker may be overkill.
- You need direct hardware access and are just starting out (USB, GPIO, Arduino serial). It is possible, but needs extra flags and can be more painful for beginners.
- You need heavy GUI apps or GPU features and don't know the setup yet. These can be harder to configure.
- Your laptop has very little RAM or disk space. Docker uses both.
- You are only learning basic coding. Learn the code first; Docker can come later.

---

## 3. Docker: steps to get started

### Step 1: Install Docker

Install Docker Desktop (Windows/macOS) or Docker Engine (Linux). Check that it works:

```bash
docker --version
# Shows the version. This means the installation is OK.

docker run hello-world
# Downloads a small test image and runs it.
```

### Step 2: Create a Dockerfile

Make a `Dockerfile` in your project root (see the boilerplate below). It tells Docker how to build your app.

### Step 3: Create a `.dockerignore`

This prevents unnecessary files from being copied into the image.

### Step 4: Build the image

```bash
docker build -t myapp .
```

- `-t myapp` gives the image a name.
- `.` means "use the Dockerfile in this folder".

### Step 5: Run the container

```bash
docker run -p 3000:3000 myapp
```

`-p 3000:3000` connects port 3000 on your laptop to port 3000 inside the container. Open `http://localhost:3000`.

### Step 6: Check and debug

```bash
docker ps
# Is it running?

docker logs <container>
# What is it printing? Are there any errors?
```

### Step 7: Stop and clean up

```bash
docker stop <container>
docker rm <container>
```

### Step 8 (optional): Push to Docker Hub

This lets others (or a server) pull your image.

```bash
docker login
docker tag myapp yourusername/myapp:1.0
docker push yourusername/myapp:1.0
```

---

## 4. Docker boilerplate (copy and edit)

### 4.1 Node.js / Express (MERN backend)

```dockerfile
# Start from a small image that already has Node installed
FROM node:20-alpine

# All following commands run inside /app in the container
WORKDIR /app

# Copy dependency files first so Docker can cache npm install
COPY package*.json ./

# Install dependencies (happens at build time)
RUN npm ci

# Copy the rest of the code
COPY . .

# Document the port used by the app
EXPOSE 3000

# Command that runs when the container starts
CMD ["node", "index.js"]
```

### 4.2 Python (Flask / FastAPI / scripts)

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### 4.3 React frontend (multi-stage; final image is small)

```dockerfile
# Stage 1: build the React app
FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: serve the built files with Nginx (no Node needed in final image)
FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Note:** Vite builds to `dist`; Create React App builds to `build`. Change the copied path if needed.

### 4.4 `.dockerignore`

```dockerignore
node_modules
.git
.env
*.log
dist
build
__pycache__
.DS_Store
```

### 4.5 `docker-compose.yml` (MERN: frontend + backend + MongoDB)

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      - MONGO_URL=mongodb://db:27017/mydb
    # "db" is the service name below
    depends_on:
      - db

  db:
    image: mongo:7
    volumes:
      - dbdata:/data/db
    # Keeps data after the container stops

volumes:
  dbdata:
```

Containers in the same Compose file communicate using service names (`db`, `backend`), not `localhost`.

Useful Compose commands:

```bash
docker compose up -d --build
# Build and start everything in the background

docker compose logs -f
# Watch logs live

docker compose down
# Stop and remove containers (data in the volume stays)
```

### 4.6 Good habits

- Copy dependency files first, then the code (faster rebuilds thanks to caching).
- Never put passwords or API keys in the Dockerfile. Use environment variables or secrets.
- Use small base images (`alpine`, `slim`).
- Pin versions (`node:20-alpine`, not `node:latest`).
- One container = one job.

---

## 5. Useful Docker commands

### Images

```bash
docker images                 # List images on my machine
docker build -t name:tag .    # Build an image
docker pull nginx             # Download an image
docker rmi <image>            # Delete an image
docker image prune            # Delete unused (dangling) images
docker history <image>        # See image layers and sizes
```

### Containers: run

```bash
docker run <image>                # Run once
docker run -d <image>             # -d = run in the background
docker run -p 8080:80 <image>     # Port mapping: HOST:CONTAINER
docker run --name mycont <image> # Give the container a name
docker run --rm <image>           # Automatically delete it when it stops
docker run -it ubuntu bash        # Open an interactive terminal inside
docker run -e KEY=value <image>   # Pass an environment variable
docker run -v mydata:/data <image> # Attach a volume
docker run -v $(pwd):/app <image> # Mount the current folder (for live code edits)
```

### Containers: manage

```bash
docker ps              # Running containers
docker ps -a           # All containers, including stopped ones
docker stop <name>     # Stop gracefully
docker start <name>    # Start a stopped container
docker restart <name>  # Restart a container
docker rm <name>       # Delete a container
docker rm -f <name>    # Force-delete a container, even if running
```

### Debug (most useful when something breaks)

```bash
docker logs <name>                 # See output
docker logs -f <name>              # Follow logs live
docker exec -it <name> sh          # Open a shell inside a running container
docker inspect <name>              # Full details (IP, ports, environment, mounts)
docker stats                       # Live CPU and RAM usage
docker cp <name>:/app/file.txt .   # Copy a file out of a container
```

### Volumes and networks

```bash
docker volume ls
docker volume rm <volume>
docker network ls
```

### Clean up (when disk is full)

```bash
docker system df       # Show how much space Docker uses
docker system prune    # Remove stopped containers and unused networks, etc.
docker system prune -a # Also remove ALL unused images (be careful!)
```

### Compose

```bash
docker compose up                 # Start all services
docker compose up -d --build      # Rebuild and start in the background
docker compose down               # Stop and remove containers
docker compose down -v            # Also delete volumes (data is gone!)
docker compose ps                 # Show service status
docker compose logs -f <svc>      # Follow logs for one service
docker compose exec <svc> sh      # Open a shell inside a service
```

---

## 6. CI/CD: what it is

**CI = Continuous Integration**

Every time I push code, a robot automatically builds and tests it. Think of it like a teacher who checks your homework as soon as you submit it.

**CD = Continuous Delivery / Continuous Deployment**

If tests pass, the robot prepares or ships the app to a server or Docker registry.

- **Continuous Delivery:** The app is ready to ship; a human presses the button.
- **Continuous Deployment:** The app ships automatically, without a human pressing the button.

### The pipeline (a conveyor belt)

```text
Push code → Build → Test → Build Docker image → Push image → Deploy
```

If any stage fails, the conveyor belt stops and I get a red ❌.

### GitHub Actions words

| Word | Easy meaning |
|---|---|
| **Workflow** | One automation file (YAML) in `.github/workflows/`. |
| **Trigger (`on`)** | What starts it: `push`, `pull_request`, or `schedule`. |
| **Job** | A group of steps that run on one machine. |
| **Step** | One task: run a command or use an action. |
| **Runner** | The machine that runs the job (for example, `ubuntu-latest`). |
| **Action** | A ready-made step created by someone else (for example, `actions/checkout`). |
| **Secrets** | Safe storage for passwords and tokens. |

---

## 7. When to use / when NOT to use CI/CD

### ✅ Use CI/CD when

- You work in a team and many people push code.
- You have tests and want them to run automatically.
- You deploy often and want the process to be automatic and repeatable.
- You want a green tick on your GitHub repository (it can also look professional on a résumé).
- You build Docker images regularly.

### ❌ Don't bother when

- It's a quick, throwaway experiment.
- It's a tiny solo project with no tests and no deployment.
- You have no tests yet. A pipeline with nothing to check is pointless, so write tests first.
- You haven't learned Git/GitHub basics yet.

**Rule of thumb:** If you find yourself repeating "run tests, build, upload, deploy" by hand, automate it.

---

## 8. CI/CD: steps to set up (GitHub Actions)

### Step 1: Put your project on GitHub

### Step 2: Create the workflow folder

Make `.github/workflows/` in the repository root. The name must be exact, including the dot.

### Step 3: Create a workflow file

Create a file inside the folder, such as `ci.yml`. Add the workflow (see the boilerplate below).

### Step 4: Add secrets if needed

For Docker Hub login or server keys:

**Repository → Settings → Secrets and variables → Actions → New repository secret**

### Step 5: Commit and push

```bash
git add .github/workflows/ci.yml
git commit -m "add CI pipeline"
git push
```

### Step 6: Open the Actions tab

Watch the workflow run:

- 🟡 Running
- ✅ Passed
- ❌ Failed

### Step 7: Debug failures

If it fails, click the run, then click the red step and read the log. The error is often near the end of the failing step's log.

### Step 8 (bonus): Add a badge to your README

```markdown
![CI](https://github.com/USERNAME/REPO/actions/workflows/ci.yml/badge.svg)
```

---

## 9. CI/CD boilerplate (copy and edit)

### 9.1 CI for Node.js (build + test)

File: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # Downloads my code onto the runner

      - uses: actions/setup-node@v4
        # Installs Node
        with:
          node-version: 20
          cache: npm
          # Caches npm dependencies for speed

      - run: npm ci
        # Clean install

      - run: npm run build --if-present
        # Build (skips if there is no build script)

      - run: npm test
        # Run tests; failure = red
```

### 9.2 CI for Python

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install -r requirements.txt
      - run: pytest
```

### 9.3 Build and push a Docker image (CD part)

File: `.github/workflows/docker.yml`

```yaml
name: Docker

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: yourusername/myapp:latest
```

Create a Docker Hub token at **hub.docker.com → Account Settings → Security → New Access Token**.

Save it as the repository secret `DOCKERHUB_TOKEN`.

### 9.4 Test first, then build the image (jobs in order)

This workflow runs the Docker job only if the test job succeeds.

```yaml
name: Docker

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npm test

  docker:
    needs: test
    # Runs only if the "test" job passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: yourusername/myapp:latest
```

---

## 10. Common errors and fixes

| Problem | Likely reason | Fix |
|---|---|---|
| App runs but won't open in the browser | Wrong port mapping | Check `-p HOST:CONTAINER`. The app must listen on `0.0.0.0`, not `localhost`. |
| Port is already in use | Something else is using that port | Stop the other process or use a different host port, such as `-p 3001:3000`. |
| Backend can't reach the database in Compose | The backend is using `localhost` | Use the service name (`db`) instead. |
| Data disappeared | No volume was configured | Add a volume. |
| `COPY` failed: file not found | File is outside the build context or excluded by `.dockerignore` | Check the file path, build context, and `.dockerignore`. |
| Image is huge | Large base image or `node_modules` copied into the image | Use `alpine` / `slim`, add a `.dockerignore`, and use multi-stage builds. |
| Changes are not showing | The old image is still being used | Rebuild with `docker build` or `docker compose up --build`. |
| Permission denied (Linux) | User is not in the Docker group | The original note suggests `sudo usermod -aG docker $USER`, then logging out and back in. **Security note:** membership in the Docker group grants powerful privileges; understand the implications before adding a user. |
| Pipeline is not starting | Wrong folder/file name or YAML indentation | Use `.github/workflows/*.yml` and spaces, not tabs. |
| `Error: Process completed with exit code 1` | A command failed | Open the red step and read the log above the failure. |
| Docker Hub login fails in CI | Secret name typo or invalid token | Check that the secret names match exactly and that the token is valid. |
| Disk is full | Old images and containers are piling up | Review Docker usage with `docker system df`; clean up carefully with `docker system prune`. |

---

## 11. My project notes

*Fill this in so future me remembers.*

- **Project name:**
- **What I put in Docker (frontend / backend / database):**
- **Which commands I used the most:**
- **What broke and how I fixed it:**
- **Did we have CI/CD? What did the pipeline do?**

---

## 12. To learn next

- Multi-stage builds in more depth
- Docker networks
- Docker secrets and environment files
- Deploy to a server / cloud (AWS, Render, etc.)
- Caching in GitHub Actions
- Development / staging / production deployments
- Docker with hardware / robotics (ROS 2 containers)
