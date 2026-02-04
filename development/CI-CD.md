> What is CI/CD?
> 
> 
> CI/CD stands for **Continuous Integration** and **Continuous Deployment/Delivery**. It's a set of practices that automate the process of building, testing, and deploying your code. Think of it as a robot that checks your homework, makes sure it's correct, and then submits it for you!
> 

---

## Understanding CI/CD - The Big Picture

### What Problem Does CI/CD Solve?

**Without CI/CD (The Old Way):**

```
Developer writes code → Manually tests → Manually builds → Manually deploys → 😰 Errors discovered in production!

```

**With CI/CD (The Smart Way):**

```
Developer writes code → Auto-tested → Auto-built → Auto-deployed → 🎉 Errors caught early!

```

### The Two Parts Explained

| Term | Full Form | What It Does | When It Runs |
| --- | --- | --- | --- |
| **CI** | Continuous Integration | Automatically tests and validates your code | Every time you push code or create a PR |
| **CD** | Continuous Deployment | Automatically deploys your code to servers | When you're ready to release (usually with a tag) |

<!-- 📷 IMAGE PLACEHOLDER: Add a diagram showing the CI/CD pipeline flow here -->

---

## Our CI/CD Pipeline Overview

Our project uses **3 workflow files** in `.github/workflows/`:

| File | Purpose | Trigger |
| --- | --- | --- |
| `lint_test.yml` | Runs linters and tests | Push/PR to `main` or `develop` |
| `ci.yml` | Builds and pushes Docker image | When you create a version tag (e.g., `v1.0.0`) |
| `cd.yml` | Deploys to Kubernetes | Manual trigger (you choose when) |

### The Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPER WORKFLOW                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. Write Code                                                             │
│        ↓                                                                    │
│   2. Push to Branch / Create PR ──────→ [lint_test.yml] ← Runs tests        │
│        ↓                                    ↓                               │
│   3. Code Review & Merge                    ✅ or ❌                       │
│        ↓                                                                    │
│   4. Create Version Tag (v1.0.0) ──────→ [ci.yml] ← Builds Docker image     │
│        ↓                                    ↓                               │
│   5. Trigger Deployment ───────────────→ [cd.yml] ← Deploys to GKE          │
│        ↓                                    ↓                               │
│   6. 🎉 Application is LIVE!               ✅                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

<!-- 📷 IMAGE PLACEHOLDER: Add a visual flowchart of the pipeline here -->

---

## The Makefile - Your Command Center

### What is a Makefile?

A **Makefile** is like a recipe book for your project. Instead of typing long commands, you use short commands like `make build` or `make test`.

> 💡 Think of it like this: Instead of saying "go to the kitchen, get flour, get eggs, mix them, put in oven at 350 degrees for 30 minutes", you just say "make cake"!
> 

### Understanding Our Makefile Structure

Our Makefile is located at the project root. Here's how it's organized:

```makefile
# Variables (like ingredients list at the top)
APP_NAME=beacon-core
BUILD_DIR=./bin
DOCKER_IMAGE=beacon-core:latest

# Targets (like recipes)
build:    # Makes the application binary
test:     # Runs all tests
docker-build:  # Creates Docker image

```

### Key Makefile Commands

### 📦 Local Development Commands

| Command | What It Does | When to Use |
| --- | --- | --- |
| `make help` | Shows all available commands | When you're not sure what commands exist |
| `make build` | Compiles the Go application | Before running locally |
| `make run` | Runs the application locally | For local development |
| `make test` | Runs all tests with coverage | Before pushing code |
| `make lint` | Checks code for issues | Before pushing code |
| `make fmt` | Formats your code | After writing code |
| `make tidy` | Cleans up Go modules | After adding dependencies |
| `make clean` | Removes build artifacts | When things get messy |

**Example Usage:**

```bash
# Run tests before pushing your code
make test

# Build the application
make build

# Run locally
make run

```

### 🐳 Docker Commands

| Command | What It Does | When to Use |
| --- | --- | --- |
| `make docker-build` | Builds Docker image | Testing Docker locally |
| `make docker-run` | Runs Docker container | Testing containerized app |
| `make deploy` | Build + Docker image | Preparing for deployment |

### ☁️ GCP/GKE Commands

| Command | What It Does | When to Use |
| --- | --- | --- |
| `make gcp-auth` | Authenticates with GCP | Before pushing to GCP |
| `make gke-build-push` | Build & push to GCP registry | Deploying new version |
| `make gke-deploy` | Deploys to Kubernetes | Deploying to cluster |
| `make gke-status` | Shows deployment status | Checking if app is running |
| `make gke-logs` | Shows application logs | Debugging issues |
| `make gke-pods` | Lists running pods | Checking pod health |
| `make gke-restart` | Restarts the deployment | When app is stuck |

**Example: Full deployment flow**

```bash
# Step 1: Authenticate with GCP
make gcp-auth

# Step 2: Build and push Docker image
make gke-build-push

# Step 3: Deploy to GKE
make gke-deploy

# Step 4: Check status
make gke-status

```

### Understanding Makefile Variables

```makefile
# These are ENVIRONMENT VARIABLES - they come from your system or CI/CD
GCP_PROJECT_ID?=$(shell echo $$GCP_PROJECT_ID)
# The ?= means "use this default if not set"
# $(shell ...) runs a shell command

# This creates the full image path
FULL_IMAGE_PATH=$(GCP_REGION)-docker.pkg.dev/$(GCP_PROJECT_ID)/backend-images/$(IMAGE_NAME)
# Result: europe-west2-docker.pkg.dev/my-project/backend-images/beacon-core

```

> ⚠️ Important: Variables like GCP_PROJECT_ID and GKE_CLUSTER_NAME must be set as GitHub Secrets for CI/CD to work!
> 

### The `.PHONY` Declaration

```makefile
.PHONY: help lint test build run

```

> 💡 What does .PHONY mean?
> 
> 
> It tells Make that these are not actual file names - they're just command names. Without this, if you had a file named `build` in your directory, `make build` wouldn't work correctly!
> 

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of terminal running make commands -->

---

## Docker - Packaging Your Application

### What is Docker?

Docker is like a **shipping container for software**. Just like a shipping container can hold anything and be transported anywhere, a Docker container holds your application with all its dependencies and can run anywhere.

<!-- 📷 IMAGE PLACEHOLDER: Add Docker container analogy diagram -->

### Why Do We Need Docker?

| Problem Without Docker | Solution With Docker |
| --- | --- |
| "It works on my machine!" | Same container runs everywhere |
| Installing dependencies manually | Dependencies bundled in image |
| Different versions conflict | Isolated environment |
| Complex setup instructions | Just run `docker run` |

### Understanding Our Dockerfile

Our Dockerfile uses **multi-stage builds**. Think of it as cooking in two kitchens - one to prepare ingredients (build) and one to serve the meal (final image).

```docker
# ═══════════════════════════════════════════════════════════════════
# STAGE 1: THE BUILDER (Kitchen for preparation)
# ═══════════════════════════════════════════════════════════════════
FROM golang:1.24-alpine AS builder
# ↑ Uses Go 1.24 on Alpine Linux (small and fast)
# ↑ "AS builder" gives this stage a name we can reference later

# Install build dependencies
RUN apk add --no-cache git make
# ↑ apk = Alpine Package Manager (like apt for Ubuntu)
# ↑ --no-cache = Don't store the package index (keeps image small)

# Set working directory (like cd into a folder)
WORKDIR /app

# Copy go mod files FIRST (for caching)
COPY go.mod go.sum ./
# ↑ Docker caches layers. If go.mod hasn't changed,
#   it skips downloading dependencies again!

# Download dependencies
RUN go mod download
# ↑ Downloads all required Go packages

# Copy ALL source code
COPY . .
# ↑ Now copy everything else

# Build the application
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \\
    -a -installsuffix cgo \\
    -ldflags="-w -s" \\
    -o server ./cmd/server
# ↑ Let's break this down:
#   CGO_ENABLED=0    → Don't use C libraries (pure Go)
#   GOOS=linux       → Build for Linux
#   GOARCH=amd64     → Build for 64-bit processors
#   -ldflags="-w -s" → Strip debug info (smaller binary)
#   -o server        → Output file name
#   ./cmd/server     → Where main.go is located

# ═══════════════════════════════════════════════════════════════════
# STAGE 2: THE FINAL IMAGE (Kitchen for serving)
# ═══════════════════════════════════════════════════════════════════
FROM gcr.io/distroless/static:nonroot
# ↑ Distroless = Minimal image with NO shell, NO package manager
# ↑ This is MUCH more secure than regular images!
# ↑ :nonroot = Runs as non-root user by default

# Copy ONLY the binary from the builder stage
COPY --from=builder /app/server /app/server
# ↑ --from=builder references Stage 1
# ↑ We only take the compiled binary, not the source code!

# Copy config file
COPY --from=builder /app/config.yaml /app/config.yaml

# Set working directory
WORKDIR /app

# Expose the port the app runs on
EXPOSE 8080
# ↑ Documentation only - doesn't actually open the port

# Run as non-root user (security best practice)
USER nonroot:nonroot
# ↑ Never run containers as root!

# Start the application
ENTRYPOINT ["./server"]
# ↑ The command to run when container starts

```

### Why Multi-Stage Builds?

| Single-Stage Image | Multi-Stage Image |
| --- | --- |
| ~800MB (includes Go compiler, source code, etc.) | ~15MB (only the binary!) |
| Contains build tools (security risk) | Minimal attack surface |
| Slower to download/upload | Fast to transfer |

### Building Docker Images Locally

```bash
# Build the Docker image
make docker-build
# OR
docker build -t beacon-core:latest .

# Run the container
make docker-run
# OR
docker run --rm -p 8080:8080 beacon-core:latest

# The -p 8080:8080 maps your computer's port 8080 to the container's port 8080

```

### Image Tagging Explained

```bash
# Local development tag
beacon-core:latest

# Production tag (includes registry, project, and version)
europe-west2-docker.pkg.dev/my-project/backend-images/beacon-core:a1b2c3d
#  ↑ Region          ↑ Project ID   ↑ Repository      ↑ Image ↑ Git SHA

```

<!-- 📷 IMAGE PLACEHOLDER: Add diagram showing Docker layer caching -->

---

## GitHub Actions - The Automation Engine

### What are GitHub Actions?

GitHub Actions is **GitHub's built-in CI/CD system**. It's like having a robot that watches your repository and automatically does things when certain events happen.

### Key Concepts

| Term | Definition | Example |
| --- | --- | --- |
| **Workflow** | A YAML file that defines automation | `.github/workflows/ci.yml` |
| **Trigger** | What starts the workflow | Push, Pull Request, Tag |
| **Job** | A set of steps that run together | `build-and-push` |
| **Step** | A single task within a job | "Run tests" |
| **Runner** | The machine that runs your workflow | `ubuntu-latest` |
| **Action** | A reusable piece of automation | `actions/checkout@v4` |

### Workflow File Structure

```yaml
# The name shown in GitHub's UI
name: My Workflow

# TRIGGER: When should this run?
on:
  push:
    branches: [main]      # On push to main branch
  pull_request:
    branches: [main]      # On PR targeting main
  push:
    tags: ['v*']          # On version tags (v1.0.0, v2.0.0)
  workflow_dispatch:      # Manual trigger button

# ENVIRONMENT VARIABLES (available to all jobs)
env:
  MY_VAR: "value"

# JOBS: What should be done?
jobs:
  job-name:
    name: Human Readable Name
    runs-on: ubuntu-latest    # The machine to use

    steps:
      - name: Step 1 Description
        uses: some-action@v1  # Use a pre-made action

      - name: Step 2 Description
        run: echo "Hello!"    # Run a shell command

```

### Our Workflow Files Explained

### 1. `lint_test.yml` - Quality Gate

**Purpose:** Ensures code quality before merging

```yaml
name: Lint & Test

on:
  pull_request:
    branches: [ main, develop ]  # Runs on PRs to these branches
  push:
    branches: [ main, develop ]  # Also runs on direct pushes

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        # ↑ Downloads your repository code to the runner

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.24'
        # ↑ Installs Go 1.24 on the runner

      - name: Install golangci-lint
        run: |
          curl -sSfL <https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh> | sh -s -- -b $(go env GOPATH)/bin latest
        # ↑ Downloads and installs the linter

      - name: Run linter
        run: make lint
        # ↑ Uses our Makefile command!

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      # ... checkout and setup Go ...

      - name: Run tests
        run: make test

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.html
          retention-days: 7
        # ↑ Saves the coverage report for 7 days

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of GitHub Actions running lint_test.yml -->

### 2. `ci.yml` - Build & Push

**Purpose:** Creates a Docker image when you tag a release

```yaml
name: CI

on:
  push:
    tags:
      - 'v*'  # Only runs when you create tags like v1.0.0, v2.1.3, etc.
      # ↑ This is important! Regular pushes DON'T trigger this.

env:
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  # ↑ Secrets are encrypted values stored in GitHub
  # ↑ Access them with ${{ secrets.SECRET_NAME }}

jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract version from tag
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/}
          # ↑ GITHUB_REF = "refs/tags/v1.0.0"
          # ↑ ${GITHUB_REF#refs/tags/} removes the prefix, leaving "v1.0.0"

          echo "version=$VERSION" >> $GITHUB_OUTPUT
          # ↑ Makes this value available to other steps

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
        # ↑ Uses a Service Account key stored in secrets

      - name: Configure Docker for GCP
        run: make gcp-auth
        # ↑ Tells Docker how to authenticate with GCP

      - name: Build and Push Docker image
        env:
          GIT_SHA: ${{ steps.version.outputs.git_sha }}
          # ↑ Accesses the output from the "version" step
        run: make gke-build-push

```

> ⚠️ Important: The CI workflow only runs on tags, not on regular pushes!
> 

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of successful CI run with artifacts -->

### 3. `cd.yml` - Deploy

**Purpose:** Deploys to Kubernetes (manually triggered)

```yaml
run-name: Deploying ${{ inputs.git_ref }} 🚀
# ↑ This creates a custom name for each run

name: CD

on:
  workflow_dispatch:  # Manual trigger
    inputs:
      git_ref:
        description: 'Git tag or branch to deploy (e.g., v1.0.0 or main)'
        required: true
        type: string
      environment:
        description: 'Deployment environment'
        required: true
        type: choice
        default: staging
        options:
          - staging
          - production
        # ↑ Creates a dropdown in the GitHub UI!

```

**How to trigger CD manually:**

1. Go to your repository on GitHub
2. Click **Actions** tab
3. Click **CD** workflow on the left
4. Click **Run workflow** button
5. Enter the version (e.g., `v1.0.0`)
6. Select environment (staging/production)
7. Click **Run workflow**

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of manual workflow trigger UI -->

---

## Kubernetes Deployment

### What is Kubernetes?

Kubernetes (K8s) is like a **smart manager for your containers**. It:

- Keeps your application running 24/7
- Automatically restarts crashed containers
- Scales up when traffic increases
- Distributes traffic across multiple containers

### Our Kubernetes Files

Located in the `k8s/` folder:

| File | Purpose |
| --- | --- |
| `deployment.yml` | Defines how your app runs (replicas, resources, environment) |
| `service.yml` | Exposes your app to the network |
| `secret.yml` | Stores sensitive data (passwords, keys) |
| `istio.yml` | Advanced networking and traffic management |

### Understanding `deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beacon-core
  namespace: ${NAMESPACE}  # Will be replaced with "staging" or "production"
  labels:
    app: beacon-core

spec:
  replicas: 1  # How many copies of your app to run

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Can spin up 1 extra pod during update
      maxUnavailable: 0  # Never have 0 pods running (no downtime!)

  template:
    spec:
      containers:
      - name: beacon-core
        image: ${IMAGE}  # The Docker image to use
        ports:
        - containerPort: 8080

        env:
        - name: BEACON_DATABASE_MASTER_HOST
          valueFrom:
            secretKeyRef:
              name: beacon-core-secrets
              key: db-master-host
        # ↑ Gets sensitive values from Kubernetes Secrets

```

> 💡 What's ${NAMESPACE} and ${IMAGE}?
> 
> 
> These are placeholders that get replaced during deployment using the `envsubst` command in the Makefile.
> 

---

## Versioning Best Practices

### Semantic Versioning (SemVer)

We use **Semantic Versioning** format: `vMAJOR.MINOR.PATCH`

```
v1.2.3
│ │ └── PATCH: Bug fixes (backwards compatible)
│ └──── MINOR: New features (backwards compatible)
└────── MAJOR: Breaking changes

```

### When to Bump Each Number

| Scenario | Version Change | Example |
| --- | --- | --- |
| Fixed a typo | Patch | v1.0.0 → v1.0.1 |
| Fixed a bug | Patch | v1.0.1 → v1.0.2 |
| Added a new API endpoint | Minor | v1.0.2 → v1.1.0 |
| Added a new feature | Minor | v1.1.0 → v1.2.0 |
| Changed API response format | Major | v1.2.0 → v2.0.0 |
| Removed an endpoint | Major | v2.0.0 → v3.0.0 |

### Creating a Version Tag

```bash
# Step 1: Make sure you're on the latest main branch
git checkout main
git pull origin main

# Step 2: Create an annotated tag
git tag -a v1.0.0 -m "Release v1.0.0: Initial release with incident CRUD"
#        ↑ tag name  ↑ message describing the release

# Step 3: Push the tag to GitHub
git push origin v1.0.0

# This will trigger the CI workflow automatically! 🚀

```

### ⚠️ IMPORTANT: Tag Naming Rules

| ✅ Valid Tags | ❌ Invalid Tags |
| --- | --- |
| `v1.0.0` | `1.0.0` (missing 'v') |
| `v2.1.3` | `version-1.0.0` |
| `v0.0.1-alpha` | `V1.0.0` (uppercase) |

Our CI workflow looks for tags starting with `v*`, so always include the lowercase 'v'!

### Good Practices

1. **Never delete tags** - They're part of your release history
2. **Always test before tagging** - Run `make check` first
3. **Use annotated tags** - Include a message describing changes
4. **Keep a CHANGELOG** - Document what changed in each version
5. **Tag from main branch** - Ensure the code is reviewed and tested

<!-- 📷 IMAGE PLACEHOLDER: Add example of GitHub releases page -->

---

## Repository Level Configuration

### GitHub Secrets (REQUIRED!)

Secrets are encrypted environment variables stored in GitHub. Our CI/CD **requires** these secrets to work:

| Secret Name | What It Is | How to Get It |
| --- | --- | --- |
| `GCP_PROJECT_ID` | Your Google Cloud Project ID | GCP Console → Project Settings |
| `GCP_SA_KEY` | Service Account JSON key | GCP Console → IAM → Service Accounts |
| `GKE_CLUSTER_NAME` | Name of your Kubernetes cluster | GCP Console → GKE |

### How to Add Secrets

1. Go to your repository on GitHub
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Enter the name and value
5. Click **Add secret**

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of GitHub Secrets settings page -->

### Branch Protection Rules

Set up branch protection to prevent mistakes:

1. Go to **Settings** → **Branches**
2. Click **Add rule**
3. Configure:

| Setting | Recommended Value | Why |
| --- | --- | --- |
| Branch pattern | `main` | Protects the main branch |
| Require PR before merging | ✅ Enabled | Forces code review |
| Require status checks | ✅ Enabled | CI must pass before merging |
| Required checks | `Lint`, `Test` | These jobs must succeed |
| Require conversation resolution | ✅ Enabled | All comments must be addressed |

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of branch protection configuration -->

### Required Files in Repository

| File/Folder | Purpose | Required? |
| --- | --- | --- |
| `.github/workflows/` | GitHub Actions workflow files | ✅ Yes |
| `Dockerfile` | Instructions to build Docker image | ✅ Yes |
| `Makefile` | Automation commands | ✅ Yes |
| `k8s/` | Kubernetes deployment files | ✅ Yes |
| `.gitignore` | Files to not track in Git | ✅ Yes |
| `.githooks/` | Local Git hooks for development | Recommended |
| `config.yaml` | Application configuration | ✅ Yes |

### Setting Up Git Hooks

We have a post-commit hook that automatically formats code and regenerates Swagger docs:

```bash
# Run this once to enable Git hooks
make setup-hooks

# What it does:
# - After each commit, it runs `go fmt` on your code
# - Regenerates Swagger documentation
# - Creates a "chore" commit with these changes

```

---

## Step-by-Step: How a Deployment Happens

Let's walk through a complete deployment from start to finish:

### Phase 1: Development

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 1: Developer Creates Feature Branch                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   $ git checkout -b feature/add-new-endpoint                 │
│   $ # ... write code ...                                     │
│   $ make test  # Run tests locally                           │
│   $ make lint  # Check code quality                          │
│   $ git add .                                                │
│   $ git commit -m "feat: add new incident endpoint"          │
│   $ git push origin feature/add-new-endpoint                 │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of feature branch creation -->

### Phase 2: Pull Request & Review

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 2: Create Pull Request                                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Go to GitHub → Pull Requests → New Pull Request         │
│   2. Select: main ← feature/add-new-endpoint                 │
│   3. Add description and reviewers                           │
│   4. Create PR                                               │
│                                                              │
│   🤖 GitHub Actions automatically runs:                      │
│      - lint_test.yml                                         │
│      - ✅ Lint: Passed                                       │
│      - ✅ Test: Passed                                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of PR with passing checks -->

### Phase 3: Merge to Main

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 3: Merge After Approval                                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Reviewer approves the PR                                │
│   2. Click "Merge pull request"                              │
│   3. Delete the feature branch (optional but recommended)    │
│                                                              │
│   Your code is now on main! But NOT deployed yet.            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

### Phase 4: Create Release (CI)

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 4: Tag a Release                                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   $ git checkout main                                        │
│   $ git pull origin main                                     │
│   $ git tag -a v1.1.0 -m "Release: Add new incident endpoint"│
│   $ git push origin v1.1.0                                   │
│                                                              │
│   🤖 GitHub Actions automatically runs:                      │
│      - ci.yml                                                │
│      - ✅ Builds Docker image                                │
│      - ✅ Pushes to GCP Artifact Registry                    │
│                                                              │
│   Image: europe-west2-docker.pkg.dev/.../beacon-core:a1b2c3d │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of successful CI build -->

### Phase 5: Deploy (CD)

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 5: Trigger Deployment                                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Go to GitHub → Actions → CD                             │
│   2. Click "Run workflow"                                    │
│   3. Enter:                                                  │
│      - git_ref: v1.1.0                                       │
│      - environment: staging (for testing) or production      │
│   4. Click "Run workflow"                                    │
│                                                              │
│   🤖 GitHub Actions runs:                                    │
│      - cd.yml                                                │
│      - ✅ Gets GKE credentials                               │
│      - ✅ Applies Kubernetes manifests                       │
│      - ✅ Waits for rollout to complete                      │
│                                                              │
│   🎉 Your application is now LIVE!                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of CD workflow with environment selection -->

### Phase 6: Verify Deployment

```
┌──────────────────────────────────────────────────────────────┐
│ STEP 6: Verify Everything Works                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   # Check pod status                                         │
│   $ make gke-pods                                            │
│   NAME                           READY   STATUS    AGE       │
│   beacon-core-7d8f9c6b5d-x2k4n   1/1     Running   2m        │
│                                                              │
│   # Check service status                                     │
│   $ make gke-status                                          │
│   NAME          TYPE           EXTERNAL-IP    PORT(S)        │
│   beacon-core   LoadBalancer   34.89.xxx.xxx  80:31234/TCP   │
│                                                              │
│   # View logs                                                │
│   $ make gke-logs                                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘

```

<!-- 📷 IMAGE PLACEHOLDER: Add screenshot of kubectl output showing healthy pods -->

---

## Troubleshooting Common Issues

### ❌ CI/CD Workflow Not Running

| Symptom | Cause | Solution |
| --- | --- | --- |
| No workflow runs on push | Wrong branch name | Check trigger in workflow file |
| CI not running on tag | Tag doesn't start with 'v' | Use `v1.0.0` not `1.0.0` |
| CD not available | Workflow requires manual trigger | Go to Actions → CD → Run workflow |

### ❌ Build Failures

| Error | Cause | Solution |
| --- | --- | --- |
| `go: module not found` | Missing dependencies | Run `make tidy` locally first |
| `docker build failed` | Dockerfile syntax error | Check Dockerfile carefully |
| `permission denied` | Wrong file permissions | Check file modes |

### ❌ Deployment Failures

| Error | Cause | Solution |
| --- | --- | --- |
| `ImagePullBackOff` | Docker image not found | Verify image was pushed to registry |
| `CrashLoopBackOff` | Application crashing | Check `make gke-logs` for errors |
| `401 Unauthorized` | Bad GCP credentials | Regenerate and update `GCP_SA_KEY` secret |

### ❌ Secret Issues

```bash
# Error: "secret not found"
# Solution: Ensure all required secrets are set in GitHub

# Required secrets:
# - GCP_PROJECT_ID
# - GCP_SA_KEY
# - GKE_CLUSTER_NAME

```

### Getting Help

1. **Check the logs** - Always start with `make gke-logs`
2. **Check pod status** - Run `make gke-pods` and `make gke-describe`
3. **Review GitHub Actions** - Click on the failed step for details
4. **Ask for help** - Share the error message with your team lead

---

## Quick Reference Card

### Most Common Commands

```bash
# Local Development
make build          # Build the app
make run            # Run locally
make test           # Run tests
make lint           # Check code quality
make fmt            # Format code

# Docker
make docker-build   # Build Docker image
make docker-run     # Run in Docker locally

# Deployment
make gcp-auth       # Authenticate with GCP
make gke-build-push # Build & push to registry
make gke-deploy     # Deploy to GKE
make gke-status     # Check deployment status
make gke-logs       # View logs
make gke-restart    # Restart deployment

```

### Deployment Checklist

- [ ]  All tests passing locally (`make test`)
- [ ]  Code linted (`make lint`)
- [ ]  PR approved and merged
- [ ]  CI workflow completed successfully
- [ ]  Version tag created and pushed
- [ ]  CD workflow triggered with correct version
- [ ]  Deployment verified with `make gke-status`

---

## Summary

| Stage | What Happens | Who Triggers It |
| --- | --- | --- |
| **Development** | Write code, run tests locally | You |
| **Pull Request** | Automated lint & tests run | GitHub (on PR) |
| **Merge** | Code goes to main branch | Reviewer approval |
| **CI (Build)** | Docker image built & pushed | Tag creation |
| **CD (Deploy)** | Application deployed to K8s | Manual trigger |

Remember: **The pipeline protects you!** It catches errors before they reach production. Trust the process and let automation do the heavy lifting. 🚀
