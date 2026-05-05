# Part 1: Container & Docker Fundamentals + ECR Basics

---

## Table of Contents

1. [Why Containers Matter for Modern Applications](#1-why-containers-matter-for-modern-applications)
2. [What is a Container — The Core Concept](#2-what-is-a-container--the-core-concept)
3. [Containers vs Virtual Machines — The Real Difference](#3-containers-vs-virtual-machines--the-real-difference)
4. [Docker Fundamentals](#4-docker-fundamentals)
5. [Container Images Explained](#5-container-images-explained)
6. [Docker Architecture Deep Dive](#6-docker-architecture-deep-dive)
7. [Introduction to Amazon ECR](#7-introduction-to-amazon-ecr)
8. [ECR vs Docker Hub vs Other Registries](#8-ecr-vs-docker-hub-vs-other-registries)
9. [ECR Repository Structure](#9-ecr-repository-structure)
10. [Image Naming and Tagging Strategies](#10-image-naming-and-tagging-strategies)
11. [ECR Authentication Mechanisms](#11-ecr-authentication-mechanisms)
12. [Quick Reference Cheatsheet](#12-quick-reference-cheatsheet)

---

## 1. Why Containers Matter for Modern Applications

Before containers, deploying applications was painful and unpredictable. You would develop an application on your laptop, it works perfectly, then you deploy it to a server and suddenly:

```
"It doesn't work on production!"
"But it works on my machine..."
```

This happened because every environment had different:
- Operating system versions
- Library versions
- System dependencies
- Environment variables
- Configuration files

Containers solve this by **packaging your application with everything it needs to run** — the code, runtime, libraries, dependencies, and configuration — into a single portable unit that runs identically everywhere.

### The Three Core Benefits

**1. Consistency Across Environments**

```
Developer Laptop → Test Server → Staging → Production
        ↓              ↓           ↓           ↓
   Same container image runs identically in all environments
```

If it works in development, it works in production. No more "works on my machine" problems.

**2. Resource Efficiency**

Containers share the host operating system kernel and isolate only the application process. This makes them:
- **Lightweight** — megabytes vs gigabytes for VMs
- **Fast to start** — seconds vs minutes for VMs
- **Efficient** — you can run 10-100x more containers than VMs on the same hardware

**3. Microservices Architecture Enablement**

Modern applications are not monolithic — they are collections of small, independent services. Containers make it practical to:
- Deploy each microservice independently
- Scale services individually based on demand
- Update services without affecting others
- Use different languages/frameworks for different services

---

## 2. What is a Container — The Core Concept

A container is **a running instance of an image** that packages an application and all its dependencies in a standardized, isolated unit.

### The Building Analogy

Think of containers like shipping containers in global logistics:

```
Before shipping containers (1950s):
─────────────────────────────────
Different cargo (boxes, barrels, crates) → different loading methods
→ slow, expensive, error-prone

After shipping containers:
─────────────────────────────────
Everything packed in standard 20ft/40ft containers
→ universal handling, any ship/truck/crane can move them
→ revolutionized global trade

Software containers do the same for applications:
─────────────────────────────────
Before: apps deployed as loose files, scripts, manual configs
After: everything packaged in a standard container format
→ any container runtime can run them
```

### What Containers Actually Contain

```
┌─────────────────────────────────────────┐
│         YOUR APPLICATION                │  ← Your code
├─────────────────────────────────────────┤
│  Application Dependencies               │  ← npm packages, pip packages, etc.
│  (libraries, frameworks)                │
├─────────────────────────────────────────┤
│  Runtime Environment                    │  ← Node.js, Python, Java, etc.
├─────────────────────────────────────────┤
│  System Libraries                       │  ← libc, OpenSSL, etc.
├─────────────────────────────────────────┤
│  OS Utilities                           │  ← bash, curl, basic tools
├─────────────────────────────────────────┤
│  Container Runtime Interface            │
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│    HOST OPERATING SYSTEM KERNEL         │  ← Shared by all containers
└─────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│         PHYSICAL HARDWARE               │
└─────────────────────────────────────────┘
```

### Key Characteristics

- **Isolated** — Each container has its own filesystem, network, and process space
- **Shared kernel** — All containers share the host OS kernel (unlike VMs)
- **Portable** — The same container runs on laptop, data center, or cloud
- **Ephemeral** — Containers are temporary by design. When destroyed, all data inside is lost (unless using volumes)
- **Immutable** — You don't update a running container; you replace it with a new version

---

## 3. Containers vs Virtual Machines — The Real Difference

Both containers and VMs provide isolation, but they work fundamentally differently.

### Virtual Machines Architecture

```
┌────────────┐  ┌────────────┐  ┌────────────┐
│   App A    │  │   App B    │  │   App C    │
├────────────┤  ├────────────┤  ├────────────┤
│   Bins/    │  │   Bins/    │  │   Bins/    │
│  Libraries │  │  Libraries │  │  Libraries │
├────────────┤  ├────────────┤  ├────────────┤
│  Guest OS  │  │  Guest OS  │  │  Guest OS  │  ← Full OS per VM
│  (Linux)   │  │  (Linux)   │  │  (Windows) │
└────────────┘  └────────────┘  └────────────┘
       ↓               ↓               ↓
┌──────────────────────────────────────────┐
│           Hypervisor (ESXi, KVM)         │  ← Virtualizes hardware
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│           Host Operating System          │
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│           Physical Hardware              │
└──────────────────────────────────────────┘
```

Each VM includes a **complete guest operating system** (kernel + utilities), which:
- Takes gigabytes of disk space
- Requires gigabytes of RAM
- Takes minutes to boot
- Needs OS updates and patching

### Containers Architecture

```
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ App A  │ │ App B  │ │ App C  │ │ App D  │ │ App E  │
├────────┤ ├────────┤ ├────────┤ ├────────┤ ├────────┤
│ Bins/  │ │ Bins/  │ │ Bins/  │ │ Bins/  │ │ Bins/  │
│ Libs   │ │ Libs   │ │ Libs   │ │ Libs   │ │ Libs   │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘
      ↓          ↓          ↓          ↓          ↓
┌──────────────────────────────────────────────────────┐
│         Container Runtime (Docker, containerd)       │
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│         Host Operating System (Linux Kernel)         │  ← Single shared kernel
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│              Physical Hardware                       │
└──────────────────────────────────────────────────────┘
```

All containers share the **host OS kernel**. Each container includes only:
- The application code
- Application dependencies
- Minimal OS utilities needed by the app

### Side-by-Side Comparison

| Aspect | Virtual Machines | Containers |
|:-------|:-----------------|:-----------|
| **Size** | Gigabytes (full OS) | Megabytes (app + deps only) |
| **Startup Time** | Minutes | Seconds (often < 1 second) |
| **Resource Usage** | High (each VM reserves RAM/CPU) | Low (share host resources efficiently) |
| **Isolation Level** | Strong (hardware-level via hypervisor) | Process-level (kernel namespaces/cgroups) |
| **Kernel** | Each VM has its own kernel | All containers share host kernel |
| **OS Compatibility** | Can run any OS (Linux VM on Windows host) | Must match host kernel (Linux containers need Linux host) |
| **Density** | ~10-20 VMs per host | ~100-1000 containers per host |
| **Use Case** | Run different OSes, legacy apps, strong isolation | Microservices, modern cloud-native apps, CI/CD |

### Why You Can't Run Windows Containers on a Linux Host

This is a common point of confusion. Containers share the host kernel.

```
Linux host → can only run Linux containers (Ubuntu, Alpine, CentOS containers)
Windows host → can only run Windows containers

Exception: Docker Desktop on Windows/Mac uses a lightweight VM to run a Linux kernel,
then runs Linux containers inside that VM. You are still running Linux containers
on a Linux kernel — just with an extra VM layer hidden underneath.
```

### When to Use VMs vs Containers

**Use Virtual Machines when:**
- You need to run different operating systems on the same hardware (Windows + Linux)
- You require strong security isolation (multi-tenant systems, hostile environments)
- You are running legacy applications that cannot be containerized
- Compliance requires hardware-level isolation

**Use Containers when:**
- Building microservices architectures
- Running modern cloud-native applications
- You need rapid deployment and scaling
- Resource efficiency is critical
- CI/CD pipelines
- Development environments (local dev matches production exactly)

---

## 4. Docker Fundamentals

Docker is the most popular container platform. It provides tools to build, ship, and run containers. When people say "containers," they usually mean Docker containers (though other runtimes like containerd, CRI-O exist).

### Docker Components

```
┌──────────────────────────────────────────────────────────┐
│                   Docker Platform                        │
├──────────────────────────────────────────────────────────┤
│  Docker CLI                                              │  ← Command-line interface
│  (docker build, docker run, docker push)                │     you interact with
├──────────────────────────────────────────────────────────┤
│  Docker Daemon (dockerd)                                 │  ← Background service that
│  - Builds images                                         │     does the actual work
│  - Runs containers                                       │
│  - Manages networks/volumes                              │
├──────────────────────────────────────────────────────────┤
│  containerd                                              │  ← Container runtime
│  (manages container lifecycle)                           │     (lower-level execution)
├──────────────────────────────────────────────────────────┤
│  runc                                                    │  ← Actually spawns containers
│  (OCI runtime — spawns and runs containers)             │     (creates processes)
└──────────────────────────────────────────────────────────┘
```

### Core Docker Concepts

**1. Image** — A read-only template containing your application and dependencies. Think of it as a class in OOP.

**2. Container** — A running instance of an image. Think of it as an object instantiated from a class. You can run multiple containers from the same image.

**3. Dockerfile** — A text file with instructions for building an image. Like a recipe.

**4. Registry** — A repository for storing and distributing images. Docker Hub is the default public registry. ECR is AWS's private registry.

**5. Volume** — Persistent storage that exists outside the container lifecycle. When a container dies, volumes survive.

**6. Network** — Docker creates virtual networks to allow containers to communicate with each other and the outside world.

### Basic Docker Commands

```bash
# Pull an image from a registry
docker pull nginx:latest

# List local images
docker images

# Run a container from an image
docker run -d -p 80:80 --name my-nginx nginx:latest
#  -d          run in background (detached mode)
#  -p 80:80    map host port 80 to container port 80
#  --name      give the container a friendly name

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop my-nginx

# Remove a container
docker rm my-nginx

# View container logs
docker logs my-nginx

# Execute a command inside a running container
docker exec -it my-nginx bash
#  -it  = interactive terminal

# Build an image from a Dockerfile
docker build -t my-app:1.0 .

# Push an image to a registry
docker push my-app:1.0

# Remove an image
docker rmi nginx:latest
```

---

## 5. Container Images Explained

### What is a Container Image?

An image is a **layered filesystem** bundled with metadata (environment variables, entry point command, exposed ports, etc.). It is immutable — once created, it never changes.

### Image Layers — The Key to Efficiency

Images are built in layers. Each instruction in a Dockerfile creates a new layer.

```dockerfile
# Example Dockerfile
FROM ubuntu:20.04          # Layer 1: Base OS (Ubuntu filesystem)
RUN apt-get update         # Layer 2: Package index update
RUN apt-get install -y python3  # Layer 3: Python installation
COPY app.py /app/          # Layer 4: Your application code
CMD ["python3", "/app/app.py"]  # Metadata (not a filesystem layer)
```

**Visual representation of layers:**

```
┌──────────────────────────────────────┐
│  Layer 4: app.py (1 MB)              │  ← Your code
├──────────────────────────────────────┤
│  Layer 3: Python3 installed (50 MB)  │  ← apt-get install
├──────────────────────────────────────┤
│  Layer 2: apt cache updated (20 MB)  │  ← apt-get update
├──────────────────────────────────────┤
│  Layer 1: Ubuntu base OS (70 MB)     │  ← FROM ubuntu:20.04
└──────────────────────────────────────┘

Total image size: 141 MB
```

### Why Layers Matter

**1. Caching**

Each layer is cached. If you rebuild the image and Layer 1-3 haven't changed, Docker reuses the cached layers. Only Layer 4 (your app code) gets rebuilt. This makes builds incredibly fast.

```
First build:  takes 5 minutes (downloads Ubuntu, installs Python)
Second build: takes 5 seconds (reuses cached layers, only copies new app.py)
```

**2. Sharing Between Images**

If you have 10 different applications all using `ubuntu:20.04` as the base, Docker stores the Ubuntu layer **once** and shares it across all images. This saves massive amounts of disk space.

```
Without layers:
─────────────
Image A (ubuntu + app A): 150 MB
Image B (ubuntu + app B): 150 MB
Image C (ubuntu + app C): 150 MB
Total: 450 MB

With layers (Docker's approach):
─────────────
Ubuntu base layer: 70 MB (stored once, shared by A, B, C)
App A layer: 10 MB
App B layer: 15 MB
App C layer: 12 MB
Total: 107 MB  ← 4x space savings
```

**3. Atomic Updates**

When you update your app, you only replace the top layer(s) containing your code. The base OS and dependencies remain unchanged. This makes deployments fast.

### Image Tags

A tag is a label attached to an image to identify different versions.

```
nginx:latest        ← points to the most recent version
nginx:1.21.6        ← specific version
nginx:1.21.6-alpine ← version + variant (Alpine Linux base)
myapp:v1.0.0        ← your app version 1.0.0
myapp:prod          ← your app production tag
myapp:abc123        ← tag based on git commit SHA
```

**Important:** `latest` is just a convention, not a special tag. It does NOT auto-update. `latest` = whatever was tagged `latest` at the time you pulled it.

### Image Naming Convention

```
[registry]/[namespace]/[repository]:[tag]

Examples:
─────────
docker.io/library/nginx:latest
  ↑         ↑       ↑      ↑
registry  namespace repo  tag

123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
  ↑                                             ↑      ↑
AWS ECR registry URL                          repo   tag
(includes account ID and region)

If omitted:
─────────
nginx              → resolves to docker.io/library/nginx:latest
nginx:1.21         → resolves to docker.io/library/nginx:1.21
my-app             → no registry = local image only (not in a registry)
```

---

## 6. Docker Architecture Deep Dive

### How Docker Actually Runs Containers

Docker leverages Linux kernel features to create isolation:

**1. Namespaces** — Create isolated views of system resources

```
┌───────────────────────────────────────────────────────┐
│  Host System                                          │
│                                                       │
│  ┌─────────────────┐  ┌─────────────────┐           │
│  │  Container A    │  │  Container B    │           │
│  │                 │  │                 │           │
│  │  PID namespace: │  │  PID namespace: │           │
│  │  Process 1      │  │  Process 1      │  ← Both see "PID 1"
│  │  (nginx)        │  │  (postgres)     │     but actually
│  │                 │  │                 │     different processes
│  └─────────────────┘  └─────────────────┘           │
│                                                       │
│  Actual PIDs: 1234, 1235, 1236 ...                   │  ← Host sees real PIDs
└───────────────────────────────────────────────────────┘
```

**Types of namespaces:**
- **PID** — Process IDs (container sees its own process tree)
- **NET** — Network interfaces, IP addresses, routing tables
- **MNT** — Filesystem mounts (container has its own root filesystem)
- **UTS** — Hostname and domain name
- **IPC** — Inter-process communication (message queues, semaphores)
- **USER** — User and group IDs (map container root to non-root on host)

**2. Control Groups (cgroups)** — Limit and monitor resource usage

```
Container A limits:
─────────────────
CPU:    1 core
Memory: 512 MB
Disk I/O: 10 MB/s

Container B limits:
─────────────────
CPU:    2 cores
Memory: 2 GB
Disk I/O: 50 MB/s
```

This ensures Container A cannot consume resources belonging to Container B. cgroups enforce hard limits.

**3. Union Filesystem (Overlay2, AUFS)** — Layer multiple filesystems efficiently

```
When you run a container:
─────────────────────────
Image layers (read-only):
  Layer 1: Ubuntu
  Layer 2: Python
  Layer 3: App code

Container layer (read-write):
  Layer 4: Runtime changes (logs, temp files, etc.)

All layers appear as a single filesystem to the container.
When container is deleted, Layer 4 is destroyed. Layers 1-3 remain for reuse.
```

### Docker Security

**Containers are NOT full security boundaries like VMs.** They share the host kernel. A kernel exploit in a container can potentially compromise the host.

Best practices:
- **Run containers as non-root users** — don't run processes as UID 0 inside containers
- **Use minimal base images** (Alpine, Distroless) — fewer packages = smaller attack surface
- **Scan images for vulnerabilities** — tools like Trivy, Clair, ECR image scanning
- **Limit capabilities** — drop unnecessary Linux capabilities (like `CAP_SYS_ADMIN`)
- **Use read-only filesystems** when possible

---

## 7. Introduction to Amazon ECR

Amazon Elastic Container Registry (ECR) is a fully managed container image registry service. It is AWS's equivalent of Docker Hub but private by default and integrated with AWS services.

### What ECR Does

```
Developer Workflow:
───────────────────

1. Write code locally
2. Build Docker image locally
   ↓
   docker build -t my-app:v1.0 .
   ↓
3. Push image to ECR
   ↓
   docker tag my-app:v1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
   docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
   ↓
4. ECR stores the image securely
   ↓
5. ECS/EKS/Lambda pulls image from ECR when deploying
```

### Why Use ECR Instead of Docker Hub?

| Feature | Docker Hub (Public Free Tier) | Amazon ECR |
|:--------|:------------------------------|:-----------|
| **Privacy** | Public by default (private = paid) | Private by default |
| **Integration** | Generic, works anywhere | Deep AWS integration (IAM, VPC Endpoints, CloudTrail) |
| **Pull Rate Limits** | 100 pulls/6hrs (anonymous), 200/6hrs (free account) | No limits (within AWS) |
| **Image Scanning** | Manual/third-party | Built-in (Clair-based) automated scanning |
| **Data Transfer Cost** | Free (but slow from AWS regions) | Free within same region, standard AWS charges across regions |
| **Encryption** | Not default | Encrypted at rest (AES-256) by default |
| **Access Control** | Docker Hub accounts | AWS IAM policies (fine-grained per repository) |
| **High Availability** | Docker Hub SLA | AWS regional SLA |

### ECR Key Features

**1. Fully Managed**
- No servers to maintain
- Automatic scaling
- Highly available (replicated across AZs in a region)

**2. Secure**
- Images encrypted at rest using AWS KMS
- Images encrypted in transit (TLS)
- IAM-based access control
- VPC Endpoint support (private access without internet gateway)

**3. Integrated**
- Works natively with ECS, EKS, Lambda, CodeBuild, CodePipeline
- CloudTrail logging for all API calls
- CloudWatch metrics and alarms
- EventBridge integration for automation

**4. Image Scanning**
- Automated vulnerability scanning on push
- Continuous scanning (rescans images as new CVE databases are released)
- Severity-based findings (Critical, High, Medium, Low)

**5. Lifecycle Policies**
- Automatically expire old/unused images
- Keep only the last N images
- Delete images older than X days
- Save costs on storage

**6. Replication**
- Cross-region replication (disaster recovery, multi-region apps)
- Cross-account replication (share images between accounts)

---

## 8. ECR vs Docker Hub vs Other Registries

### Registry Options Comparison

| Feature | ECR | Docker Hub | GitHub Container Registry | Google Container Registry | Azure Container Registry |
|:--------|:----|:-----------|:-------------------------|:--------------------------|:------------------------|
| **Cloud Provider** | AWS | Independent | GitHub | Google Cloud | Azure |
| **Default Privacy** | Private | Public (free), Private (paid) | Private | Private | Private |
| **Authentication** | AWS IAM | Docker Hub accounts | GitHub tokens | GCP IAM | Azure AD |
| **Image Scanning** | Built-in (Clair) | Paid tier | Third-party | Built-in | Built-in (Qualys) |
| **Pull Rate Limits** | None (in AWS) | Yes (100-200/6hrs) | Generous | None | None |
| **Cost Model** | Storage ($0.10/GB/month) | Free tier, paid plans | Free for public, paid for private | Storage + network | Storage + operations |
| **Best For** | AWS workloads | Public images, small projects | OSS projects on GitHub | GCP workloads | Azure workloads |

### When to Use ECR

**Use ECR when:**
- Running applications on AWS (ECS, EKS, Lambda, EC2)
- You need private images by default
- You want IAM-based access control
- You require image scanning and lifecycle management
- You are already using AWS services
- Compliance requires AWS-native encryption and logging

**Use Docker Hub when:**
- Sharing public open-source images
- Working with non-AWS environments
- Small projects with <100 pulls per 6 hours
- Using well-known public base images (nginx, node, python, etc.)

**Hybrid approach (common in production):**
```
Use Docker Hub for public base images:
  FROM node:18-alpine

Use ECR for your private application images:
  docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

---

## 9. ECR Repository Structure

### Repositories in ECR

In ECR, a **repository** holds multiple versions (tags) of a single application/image. Think of it like a Git repository but for Docker images.

```
Account: 123456789012
Region:  us-east-1

Repositories:
─────────────────────────────────────────────────
my-web-app              (contains all versions of web app)
  ├── v1.0.0
  ├── v1.1.0
  ├── v1.2.0
  ├── latest
  └── dev-branch-abc123

my-api-service          (separate repository for API)
  ├── v2.0.0
  ├── v2.0.1
  └── latest

ml-model-inference      (separate repository for ML model)
  ├── model-v1
  ├── model-v2
  └── latest
─────────────────────────────────────────────────
```

**Key rules:**
- One repository per application/microservice
- Multiple tags within a repository (versions, environments, git commits)
- Repository names must be unique within an AWS account + region

### ECR Naming Conventions

**Repository URI format:**
```
[account-id].dkr.ecr.[region].amazonaws.com/[repository-name]:[tag]

Example:
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-web-app:v1.0.0
     ↑              ↑                              ↑        ↑
  account ID     region                      repository   tag
```

**Best practices for repository names:**
```
Good names:
───────────
web-app
api-service
user-auth-service
ml-inference
data-processor

Avoid:
───────────
app                    (too generic)
project1               (not descriptive)
MyAppV2                (capital letters, versioning in name — use tags instead)
web_app                (use hyphens, not underscores)
```

### Public vs Private Repositories

**Private repositories (default):**
- Only accessible with proper IAM permissions
- Not visible on the internet
- Require authentication to pull images
- Standard for all internal applications

**Public repositories (ECR Public):**
- Accessible to anyone on the internet
- Separate service: `public.ecr.aws` (not account-specific)
- Used for open-source projects
- Equivalent to public repos on Docker Hub

```
Private ECR image:
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
  ↑ Requires IAM permissions tied to your account

Public ECR image:
public.ecr.aws/company-name/my-oss-tool:v1.0
  ↑ Anyone can pull without authentication
```

---

## 10. Image Naming and Tagging Strategies

Tagging is critical for production environments. Poor tagging practices lead to confusion, deployment errors, and outages.

### Semantic Versioning

Use semantic versioning for production releases:

```
v[MAJOR].[MINOR].[PATCH]

v1.0.0   → Initial release
v1.0.1   → Bug fix (backward compatible)
v1.1.0   → New feature (backward compatible)
v2.0.0   → Breaking change (NOT backward compatible)

Example tags in ECR:
─────────────────────
my-app:v1.0.0
my-app:v1.0.1
my-app:v1.1.0
my-app:v2.0.0
```

### Git Commit SHA Tags

Tag images with git commit hash for traceability:

```
my-app:abc123def456  ← exact commit that built this image

Benefits:
─────────
✓ Reproducible — you can always trace back to the exact code
✓ Unique — no two images share the same commit SHA
✓ Audit trail — git history + ECR image history
```

### Environment-Specific Tags

Use environment tags for deployment stages:

```
my-app:dev       ← latest development build
my-app:staging   ← deployed to staging environment
my-app:prod      ← deployed to production

WARNING: These are mutable (you overwrite them with new builds).
Always ALSO tag with immutable tags (version or commit SHA).
```

### Multi-Tagging Strategy (Best Practice)

Tag the same image with multiple tags:

```bash
# Build the image
docker build -t my-app .

# Tag it with multiple tags
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc123def
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:prod
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Push all tags
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc123def
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:prod
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

Now in ECR you have:
```
my-app:v1.2.3      → immutable semantic version
my-app:abc123def   → immutable commit hash
my-app:prod        → mutable environment tag
my-app:latest      → mutable "newest" tag
```

All four tags point to the **exact same image** (same digest). Pushing the same image multiple times with different tags does not consume extra storage — ECR is content-addressed.

### Tagging Anti-Patterns

```
❌ Using only "latest"
   Problem: No versioning, impossible to rollback, no audit trail

❌ Using only mutable tags (dev, staging, prod)
   Problem: You lose track of what actually ran in production

❌ Including build numbers in repository names
   my-app-build-123
   my-app-build-124
   Problem: Should be tags, not separate repositories

❌ Using dates as the only tag
   my-app:2026-04-29
   Problem: Not meaningful, doesn't indicate what changed

✅ Best practice: Combine semantic version + commit SHA + environment tag
   my-app:v1.2.3
   my-app:abc123def
   my-app:prod
```

---

## 11. ECR Authentication Mechanisms

Unlike Docker Hub where you log in with a username/password, ECR uses **AWS IAM credentials** for authentication.

### How ECR Authentication Works

```
Step 1: Your AWS credentials (IAM user or role)
         ↓
Step 2: Call ECR's GetAuthorizationToken API
         ↓
Step 3: Receive a temporary token (valid for 12 hours)
         ↓
Step 4: Use token to authenticate Docker CLI to ECR
         ↓
Step 5: Push/pull images
```

### Getting a Token with AWS CLI

```bash
# Retrieve authentication token and log Docker into ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Explanation:
# aws ecr get-login-password    → retrieves base64-encoded token
# --username AWS                → literal string "AWS" (always)
# --password-stdin              → read password from stdin (the token)
# 123456789012.dkr.ecr.us-east-1.amazonaws.com  → ECR registry URL
```

After this command succeeds, you can push/pull images:

```bash
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

### IAM Permissions Required

**To push images (developer/CI pipeline):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

**To pull images (ECS task, Lambda function, developer):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### ECR Authentication for ECS

When ECS launches a task, it automatically:
1. Uses the **ECS Task Execution Role** to authenticate to ECR
2. Pulls the image
3. Starts the container

You must attach ECR pull permissions to the Task Execution Role:

```
Managed policy to attach:
─────────────────────────
AmazonECSTaskExecutionRolePolicy

(includes ECR pull permissions + CloudWatch Logs write permissions)
```

---

## 12. Quick Reference Cheatsheet

### Container vs VM

| Aspect | Container | Virtual Machine |
|:-------|:----------|:----------------|
| Size | Megabytes | Gigabytes |
| Startup | Seconds | Minutes |
| Isolation | Process-level | Hardware-level |
| Kernel | Shared | Independent |
| Density | 100-1000 per host | 10-20 per host |

### Docker Commands

```bash
# Images
docker pull nginx:latest              # Download image
docker images                          # List local images
docker build -t myapp:v1 .            # Build from Dockerfile
docker tag myapp:v1 myapp:latest      # Tag an image
docker rmi myapp:v1                   # Delete image

# Containers
docker run -d -p 80:80 nginx          # Run container
docker ps                              # List running containers
docker ps -a                           # List all containers
docker stop <container-id>             # Stop container
docker rm <container-id>               # Remove container
docker logs <container-id>             # View logs
docker exec -it <container-id> bash   # Shell into container

# Cleanup
docker system prune -a                 # Remove all unused images/containers
```

### ECR Commands

```bash
# Authentication
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name my-app --region us-east-1

# Tag for ECR
docker tag my-app:v1.0 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

# Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

# Pull from ECR
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

# List repositories
aws ecr describe-repositories --region us-east-1

# List images in repository
aws ecr describe-images --repository-name my-app --region us-east-1
```

### ECR Image URI Structure

```
[account-id].dkr.ecr.[region].amazonaws.com/[repository]:[tag]

123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
     ↑              ↑                            ↑       ↑
 account ID      region                     repository  tag
```

### Image Tagging Best Practices

```
✅ Multi-tag strategy:
   my-app:v1.2.3       (semantic version)
   my-app:abc123       (git commit SHA)
   my-app:prod         (environment)
   my-app:latest       (convenience)

❌ Avoid:
   Using only "latest"
   No version tagging
   Versions in repository names
```

### Key Concepts

```
Image     = Immutable template (blueprint)
Container = Running instance of an image
Layer     = Filesystem changes (cached, shared)
Tag       = Label for image version
Registry  = Storage for images (ECR, Docker Hub)
Repository = Collection of tagged images for one app
```

---

**End of Part 1 — Container & Docker Fundamentals + ECR Basics**
