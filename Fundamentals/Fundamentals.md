# 🐳 Phase 1 — Docker Fundamentals

> **Before you run a single `docker run` command — understand WHY Docker exists, WHAT problem it solves, and HOW it actually works under the hood.**
>
> Most people skip fundamentals and spend months confused. Don't be that person.

---

## 🗺️ What You Will Learn in This Phase

| Topic | Concept |
|---|---|
| Virtualization | What it is, why it was invented |
| Hypervisors | Type 1 vs Type 2 |
| VMs vs Containers | Core differences, tradeoffs |
| Linux Namespaces | How containers get isolation |
| cgroups | How containers get resource limits |
| OCI Runtime | The open standard behind containers |
| Docker Architecture | Engine, Daemon, containerd, runc |
| Container Lifecycle | Every state a container passes through |

---

## 📚 THEORY

---

### 1️⃣ What is Virtualization?

**Virtualization** is the technology that allows you to run multiple isolated computing environments on a single physical machine.

Before virtualization:
- 1 application = 1 physical server
- Most servers ran at 5–15% CPU utilization
- Huge waste of hardware, power, and money

After virtualization:
- 1 physical server = many virtual machines
- Each VM thinks it owns the full machine
- Hardware utilization jumped to 60–80%

```
Physical Server
└── Hypervisor
    ├── VM 1  (has its own OS, CPU, RAM, Disk)
    ├── VM 2  (has its own OS, CPU, RAM, Disk)
    └── VM 3  (has its own OS, CPU, RAM, Disk)
```

---

### 2️⃣ Hypervisors

A **hypervisor** is the software layer that creates and manages virtual machines.

#### Type 1 Hypervisor — Bare Metal
Runs **directly on hardware**. No host OS underneath.

```
Physical Hardware
└── Hypervisor (Type 1)
    ├── VM 1
    ├── VM 2
    └── VM 3
```

| Product | Vendor |
|---|---|
| VMware ESXi | VMware |
| Microsoft Hyper-V | Microsoft |
| KVM | Linux kernel |
| Xen | Open source |

✅ High performance — direct hardware access
✅ Used in enterprise data centers and AWS
❌ Complex to set up

#### Type 2 Hypervisor — Hosted
Runs **on top of a host OS** (like a regular application).

```
Physical Hardware
└── Host OS (Windows / macOS / Linux)
    └── Hypervisor (Type 2)
        ├── VM 1
        └── VM 2
```

| Product | Vendor |
|---|---|
| VirtualBox | Oracle |
| VMware Workstation | VMware |
| Parallels | Corel |

✅ Easy to set up on laptops
❌ Slower — extra OS layer between hardware and VM

---

### 3️⃣ Virtual Machines vs Containers

This is the most important comparison to understand before learning Docker.

```
┌─────────────────────────────────┐   ┌─────────────────────────────────┐
│        VIRTUAL MACHINE          │   │           CONTAINER             │
├─────────────────────────────────┤   ├─────────────────────────────────┤
│  App A      App B      App C    │   │  App A      App B      App C    │
│  ─────      ─────      ─────    │   │  ─────      ─────      ─────    │
│  Libs       Libs       Libs     │   │  Libs       Libs       Libs     │
│  ─────      ─────      ─────    │   │  ─────      ─────      ─────    │
│  Guest OS   Guest OS   Guest OS │   │                                 │
│  ─────────────────────────────  │   │       Container Runtime         │
│         Hypervisor              │   │  ─────────────────────────────  │
│  ─────────────────────────────  │   │         Host OS Kernel          │
│        Physical Hardware        │   │  ─────────────────────────────  │
└─────────────────────────────────┘   │        Physical Hardware        │
                                       └─────────────────────────────────┘
```

#### Direct Comparison

| Property | Virtual Machine | Container |
|---|---|---|
| Startup time | Minutes | Milliseconds |
| Size | GBs (full OS) | MBs (just app + libs) |
| Isolation | Full OS-level | Process-level (namespaces) |
| Performance | Slower (hypervisor overhead) | Near-native speed |
| Portability | Heavy, hypervisor-dependent | Lightweight, runs anywhere |
| OS | Each VM has its own OS | Shares host kernel |
| Resource usage | High | Very low |
| Use case | Full OS isolation needed | App deployment, microservices |

#### Key Insight

> Containers are **NOT** mini VMs.
>
> A container is a **Linux process** (or group of processes) that believes it is isolated — because of namespaces and cgroups. There is **no second OS** running inside a container.

---

### 4️⃣ Why Docker Exists

Before Docker (circa 2012):

```
Developer's laptop:      "Works on my machine ✅"
Staging server:          "App crashes ❌"
Production server:       "Different behavior ❌"

Reason: Different OS, different library versions,
        different environment variables, different configs.
```

**Docker solved the "works on my machine" problem** by packaging the application AND its entire environment (libraries, configs, runtime) into a single portable unit — the **Docker image**.

```
Docker Image = App code
             + Runtime (Node, Python, Java...)
             + Libraries and dependencies
             + Config files
             + Environment variables

Run the same image anywhere → same behavior. Always.
```

Docker also made containers easy. Linux containers existed before Docker (LXC since 2008), but using them directly required deep Linux expertise. Docker added:
- A simple CLI (`docker run`, `docker build`)
- A packaging format (Dockerfile)
- A distribution system (Docker Hub)
- A layered image format (faster builds and pulls)

---

### 5️⃣ Linux Namespaces — How Isolation Works

Containers feel isolated because Linux **namespaces** make them feel that way. A namespace is a Linux kernel feature that wraps a global resource so that a process thinks it owns it exclusively.

Docker uses **7 namespaces**:

| Namespace | What it Isolates | Example |
|---|---|---|
| `pid` | Process IDs | Container sees its own PID 1, doesn't see host PIDs |
| `net` | Network interfaces | Container gets its own `eth0`, IP address |
| `mnt` | Filesystem mount points | Container sees its own `/`, not host filesystem |
| `uts` | Hostname & domain name | Container can have its own hostname |
| `ipc` | Inter-process communication | Shared memory, message queues isolated |
| `user` | User and group IDs | Container root ≠ host root |
| `cgroup` | cgroup root | Container sees its own cgroup hierarchy |

```bash
# See the namespaces of a running container:
docker run -d --name test nginx
docker inspect test | grep -i pid

# On the host:
ls -la /proc/<pid>/ns/
```

#### Key Insight

> A container is NOT isolated by a hypervisor.
> It is isolated by **kernel namespaces**.
> It shares the **same kernel** as the host — it just can't see outside its own namespace.

---

### 6️⃣ cgroups — How Resource Limits Work

**cgroups** (control groups) is the Linux kernel feature that **limits and measures** resources used by a group of processes.

Without cgroups, one container could eat all CPU/RAM on the host and starve all other containers.

cgroups let Docker enforce:

| Resource | What cgroups Controls |
|---|---|
| CPU | How much CPU time a container can use |
| Memory | Max RAM a container can consume |
| Disk I/O | Read/write bandwidth limits |
| Network | Bandwidth throttling |
| PIDs | Max number of processes inside container |

```bash
# Set resource limits when running a container:
docker run \
  --memory="512m" \           # Max 512 MB RAM
  --memory-swap="512m" \      # No swap
  --cpus="1.5" \              # Max 1.5 CPU cores
  nginx

# Inspect actual cgroup limits from inside the container:
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

#### What is OOMKilled?

When a container exceeds its memory limit, the Linux kernel's **OOM Killer** (Out Of Memory Killer) terminates the container process.

```bash
docker inspect <container> | grep -i oomkilled
# "OOMKilled": true   ← your container was killed for exceeding memory limit
```

**Solution:** Either increase `--memory` limit or fix a memory leak in your app.

---

### 7️⃣ OCI — Open Container Initiative

**OCI** (Open Container Initiative) is an open standard that defines:

1. **Image Spec** — What a container image must look like (layers, manifest, config)
2. **Runtime Spec** — How a container runtime must behave (how to start/stop a container)
3. **Distribution Spec** — How images are pushed/pulled from a registry

#### Why OCI Matters

Before OCI, Docker controlled everything. If Docker changed their format, everyone's tooling broke.

OCI means:
- Any OCI-compliant image works with any OCI-compliant runtime
- Docker images work with Kubernetes (containerd), Podman, and others
- No vendor lock-in at the image format level

```
OCI-compliant runtimes:
├── runc          ← Docker's default (also used by K8s)
├── crun          ← Faster C-based alternative
├── kata          ← VM-backed containers (extra isolation)
└── gVisor        ← Sandboxed userspace kernel (Google)
```

---

### 8️⃣ Docker Architecture — Full Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker CLI                               │
│                  (docker run, docker build...)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │  REST API (HTTP/Unix socket)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Docker Daemon                              │
│                       (dockerd)                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    containerd                           │    │
│  │          (manages container lifecycle)                  │    │
│  │                                                         │    │
│  │   ┌─────────────────────────────────────────────────┐   │    │
│  │   │              containerd-shim                    │   │    │
│  │   │   ┌─────────────────────────────────────────┐   │   │    │
│  │   │   │              runc                       │   │   │    │
│  │   │   │  (creates namespaces + cgroups)         │   │   │    │
│  │   │   │  (actually starts the container)        │   │   │    │
│  │   │   └─────────────────────────────────────────┘   │   │    │
│  │   └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────┐
              │      Linux Kernel        │
              │  (namespaces + cgroups)  │
              └──────────────────────────┘
```

#### Component Breakdown

---

#### 🔹 Docker CLI
The command-line tool you interact with.

```bash
docker run nginx
docker build -t myapp .
docker ps
```

It translates your commands into REST API calls and sends them to the Docker Daemon over a Unix socket (`/var/run/docker.sock`) or TCP.

---

#### 🔹 Docker Daemon (`dockerd`)
The background service that does the actual work.

Responsibilities:
- Receives API requests from CLI
- Manages images (pull, build, store)
- Manages containers (create, start, stop)
- Manages networks and volumes
- Delegates actual container execution to containerd

```bash
# Check daemon status
systemctl status docker

# Daemon config location
cat /etc/docker/daemon.json
```

---

#### 🔹 containerd
An industry-standard container runtime (now a CNCF project).

Responsibilities:
- Downloads and stores images
- Manages container lifecycle (create, start, stop, delete)
- Manages storage and networking
- Calls `runc` to actually start the container process

**Important:** Kubernetes can use containerd **directly** — without Docker Daemon. This is how Kubernetes runs containers in modern clusters.

```bash
# containerd is also installed when you install Docker
systemctl status containerd

# List containers via containerd directly:
sudo ctr containers list
```

---

#### 🔹 containerd-shim
A lightweight process that sits between containerd and runc.

Purpose:
- After runc starts the container and exits, shim keeps running
- Shim holds the container's stdio (logs) and reports exit status to containerd
- Allows containerd to restart/upgrade without killing containers

---

#### 🔹 runc
The actual low-level container runner. An OCI-compliant runtime.

Responsibilities:
- Reads the OCI bundle (filesystem + config)
- Creates Linux namespaces
- Applies cgroup limits
- Starts the container's PID 1 process
- Then **runc exits** (shim takes over)

```bash
# runc binary location
which runc

# runc is what actually calls clone() in the Linux kernel
# to create namespaces and start the process
```

---

### 9️⃣ Container Lifecycle — Every State Explained

```
                     docker create
                          │
                          ▼
                      CREATED
                    (filesystem ready,
                     not started yet)
                          │
                     docker start
                          │
                          ▼
                      RUNNING ◄──────────────────┐
                    (process active)              │
                          │                       │
              ┌───────────┴──────────┐            │
         docker pause           docker stop       │
              │                      │            │
              ▼                      ▼            │
           PAUSED               STOPPED           │
         (frozen,            (process stopped,    │
         SIGSTOP)             filesystem kept)    │
              │                      │            │
         docker unpause        docker start ──────┘
              │                      │
              └──────────────────────┘
                                     │
                                docker rm
                                     │
                                     ▼
                                  DELETED
                             (gone completely)
```

#### All States

| State | Description | How to Get There |
|---|---|---|
| `created` | Container exists but process not started | `docker create` |
| `running` | Process is active | `docker start` / `docker run` |
| `paused` | Process frozen (SIGSTOP sent) | `docker pause` |
| `restarting` | Container restarting (per restart policy) | Automatic |
| `exited` | Process stopped, filesystem still exists | `docker stop` / process exited |
| `dead` | Failed during removal | Rare error state |

```bash
# See container state:
docker inspect <container> | grep '"Status"'

# See all containers including stopped:
docker ps -a

# See exit code (0 = clean exit, non-zero = error):
docker inspect <container> | grep '"ExitCode"'
```

---

## 🛠️ LABS

---

### Lab 1 — Install Docker

#### Ubuntu / Debian
```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Run Docker without sudo (logout & login after this)
sudo usermod -aG docker $USER
```

#### Amazon Linux 2
```bash
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

#### Amazon Linux 2023
```bash
sudo dnf install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

#### WSL2 (Windows)
```
1. Install WSL2: wsl --install
2. Download Docker Desktop from https://www.docker.com/products/docker-desktop/
3. Enable WSL2 backend in Docker Desktop settings
4. Open your WSL terminal and run: docker version
```

---

### Lab 2 — Verify Installation

```bash
# Check CLI and daemon versions
docker version

# Expected output shows:
# Client version
# Server (daemon) version
# containerd version
# runc version

# ──────────────────────────────────────────

# System-level information
docker info

# Pay attention to:
# Containers: X running, X paused, X stopped
# Images: X
# Server Version
# Storage Driver: overlay2
# Cgroup Driver: systemd
# Kernel Version
# Operating System
# Total Memory

# ──────────────────────────────────────────

# Check daemon is running
systemctl status docker

# Check socket exists
ls -la /var/run/docker.sock
```

---

### Lab 3 — Your First Container

```bash
# Pull and run the hello-world image
docker run hello-world
```

What happens when you run this:

```
Step 1: Docker CLI sends request to Docker Daemon
Step 2: Daemon checks local image cache → not found
Step 3: Daemon pulls hello-world image from Docker Hub
Step 4: Daemon calls containerd to create container
Step 5: containerd calls runc
Step 6: runc creates namespaces + cgroups
Step 7: runc starts the container process (prints message)
Step 8: Process exits with code 0
Step 9: Container moves to "exited" state
```

```bash
# See the exited container
docker ps -a

# See the downloaded image
docker images

# Remove the container
docker rm <container-id>

# Remove the image
docker rmi hello-world
```

---

## 🎯 MINI PROJECT — Run a Basic NGINX Website

**Goal:** Run a real web server in a container, access it from your browser, and understand port mapping.

### Step 1 — Pull the NGINX Image

```bash
docker pull nginx:latest

# Inspect what you pulled
docker images nginx
docker inspect nginx:latest | grep -A5 '"Layers"'
```

### Step 2 — Run NGINX Container

```bash
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  nginx:latest
```

Flag breakdown:

| Flag | Meaning |
|---|---|
| `-d` | Detached mode — runs in background |
| `--name my-nginx` | Give the container a name |
| `-p 8080:80` | Map host port 8080 → container port 80 |
| `nginx:latest` | The image to use |

### Step 3 — Verify It's Running

```bash
docker ps

# Expected:
# CONTAINER ID   IMAGE   COMMAND                  STATUS         PORTS
# abc123         nginx   "/docker-entrypoint.…"   Up X seconds   0.0.0.0:8080->80/tcp
```

### Step 4 — Access the Website

Open your browser: **http://localhost:8080**

You should see the NGINX welcome page.

Or from terminal:
```bash
curl http://localhost:8080
```

### Step 5 — View Live Logs

```bash
# See NGINX access logs
docker logs my-nginx

# Follow logs in real time
docker logs -f my-nginx

# Then curl again and watch the log appear live
curl http://localhost:8080
```

### Step 6 — Go Inside the Container

```bash
# Open a bash shell inside the running container
docker exec -it my-nginx bash

# Inside the container:
ls /etc/nginx/          # NGINX config
ls /usr/share/nginx/html/  # Default web root (index.html lives here)
cat /etc/nginx/nginx.conf

# Exit the container
exit
```

### Step 7 — Serve Your Own HTML

```bash
# Create a custom index.html
echo "<h1>Hello from my Docker container! 🐳</h1>" > index.html

# Copy it into the running container
docker cp index.html my-nginx:/usr/share/nginx/html/index.html

# Refresh browser → your custom page appears
curl http://localhost:8080
```

### Step 8 — Stop and Remove

```bash
# Stop the container (process stopped, container still exists)
docker stop my-nginx

# Verify it's stopped
docker ps -a

# Remove the container
docker rm my-nginx

# Remove the image (optional)
docker rmi nginx:latest
```

### What You Just Did

```
You:
✅ Pulled an image from a public registry (Docker Hub)
✅ Started a container in detached mode
✅ Mapped a host port to a container port
✅ Accessed an application running inside a container
✅ Viewed container logs
✅ Executed a command inside a running container
✅ Copied a file into a container
✅ Stopped and cleaned up properly
```

---

## 🧠 CONCEPT CHECK — Test Yourself

Answer these before moving to Phase 2:

1. What is the difference between a Type 1 and Type 2 hypervisor?
2. Why do containers start faster than VMs?
3. What Linux kernel feature provides **isolation** to containers?
4. What Linux kernel feature provides **resource limits** to containers?
5. What does `dockerd` do vs what does `runc` do?
6. What does OCI mean and why does it exist?
7. What is the difference between a container in `exited` state vs a container that has been `rm`'d?
8. What does `-p 8080:80` mean? Which side is the host and which is the container?
9. What command do you run to get a shell inside a running container?
10. What happens step-by-step when you run `docker run hello-world`?

---

## ⚡ KEY TAKEAWAYS

```
✅ Containers are NOT VMs — they share the host kernel
✅ Isolation comes from Linux namespaces
✅ Resource limits come from cgroups
✅ Docker made containers easy — it didn't invent them
✅ dockerd → containerd → runc → your container process
✅ runc creates namespaces, starts the process, then exits
✅ An exited container still exists — use docker rm to delete it
✅ OCI ensures images work across any compliant runtime
✅ Every docker run triggers: pull → create → start → (run) → exit
```

---

## ➡️ Next Phase

Once you're solid on all concepts above, move to:

**[Phase 2 → Docker Basics](../02-DOCKER-BASICS/README.md)**

> Master all container lifecycle commands:
> `docker run`, `docker ps`, `docker stop`, `docker rm`, `docker logs`, `docker exec`

---

*Phase 1 of 14 — Docker Mastery Repository*
