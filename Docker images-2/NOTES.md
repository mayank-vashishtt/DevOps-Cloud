# Complete Docker & Containers Guide

## Table of Contents
1. [Basic Terminal Commands](#basic-terminal-commands)
2. [Containers Deep Dive](#containers-deep-dive)
3. [Virtual Machines vs Containers](#virtual-machines-vs-containers)
4. [Networking Fundamentals](#networking-fundamentals)
5. [Docker Commands & Options](#docker-commands--options)
6. [Background Processes](#background-processes)
7. [Container Internals](#container-internals)
8. [AWS Concepts](#aws-concepts)

---

## Basic Terminal Commands

### `history` Command
- Shows all commands you've previously run in the terminal
- Commands are stored in `~/.bash_history` (or similar file depending on shell)
- Usage:
  ```bash
  history           # Show all history
  history 20        # Show last 20 commands
  !123              # Re-run command number 123
  ```

### `ps -ef` Command
- **ps** = Process Status
- **-e** = Show all processes (every process on system)
- **-f** = Full format listing (shows more details)
- Displays:
  - UID: User ID
  - PID: Process ID
  - PPID: Parent Process ID
  - C: CPU utilization
  - STIME: Start time
  - TTY: Terminal type
  - TIME: CPU time consumed
  - CMD: Command that started the process

### `hostname` Command
- Displays the system's hostname (computer name on network)
- Can also set hostname with `hostname <new-name>` (requires root)

---

## Containers Deep Dive

### What is a Container?

A **container** is a lightweight, standalone, executable package that includes:
- Application code
- Runtime environment
- System tools
- System libraries
- Settings

**Key Characteristics:**
- **Isolated** from other containers and host system
- **Portable** - runs consistently across different environments
- **Lightweight** - shares OS kernel, no need for full OS per container
- **Fast** - starts in seconds (vs minutes for VMs)
- **Efficient** - uses fewer resources than VMs

**Think of it as:** A standardized shipping container for software - contains everything needed to run, isolated from surroundings, portable anywhere.

---

## Virtual Machines vs Containers

### Virtual Machines (VMs)
```
┌─────────────────────────────────────┐
│         Application                 │
├─────────────────────────────────────┤
│         Guest OS (Full OS)          │
├─────────────────────────────────────┤
│         Hypervisor                  │
├─────────────────────────────────────┤
│         Host OS                     │
├─────────────────────────────────────┤
│         Physical Hardware           │
└─────────────────────────────────────┘
```

**VMs Have Separate:**
- **Network Stack** - Own IP addresses, port ranges, sockets, routing tables
- **Process Space** - Independent process IDs, isolated processes
- **File System** - Complete separate file system (virtual disks)
- **User Space** - Different users, permissions, authentication
- **IPC (Inter-Process Communication)** - Isolated message queues, semaphores, shared memory

### Containers
```
┌─────────────────────────────────────┐
│     App A    │    App B    │ App C  │
├──────────────┴─────────────┴────────┤
│        Container Runtime            │
├─────────────────────────────────────┤
│         Host OS Kernel              │
├─────────────────────────────────────┤
│         Physical Hardware           │
└─────────────────────────────────────┘
```

**Containers Share:**
- OS Kernel
- Host resources (managed through isolation)

**Key Differences:**

| Aspect | Virtual Machine | Container |
|--------|----------------|-----------|
| **Boot Time** | Minutes | Seconds |
| **Size** | GBs (entire OS) | MBs (app + dependencies) |
| **Performance** | Slower (overhead) | Near-native |
| **Isolation** | Complete (hardware-level) | Process-level |
| **Resource Usage** | Heavy | Lightweight |

### Hypervisor

The **hypervisor** (Virtual Machine Monitor) is software that creates and runs virtual machines.

**Types:**
1. **Type 1 (Bare Metal)**
   - Runs directly on hardware
   - Examples: VMware ESXi, Microsoft Hyper-V, KVM
   - Better performance

2. **Type 2 (Hosted)**
   - Runs on top of host OS
   - Examples: VMware Workstation, VirtualBox, Parallels
   - Easier to set up

---

## Networking Fundamentals

### IP Address
- **IP** = Internet Protocol
- Unique identifier for devices on a network
- **IPv4 Format:** `192.168.1.100` (four octets, 0-255 each)
- **IPv6 Format:** `2001:0db8:85a3:0000:0000:8a2e:0370:7334`

### MAC Address
- **MAC** = Media Access Control
- Physical hardware address burned into network interface card (NIC)
- **Format:** `00:1A:2B:3C:4D:5E` (six pairs of hexadecimal)
- **Scope:** Works at Layer 2 (Data Link Layer)
- **Uniqueness:** Should be globally unique (assigned by manufacturer)
- **Purpose:** Used for communication within local network segment

**IP vs MAC:**
- IP: Logical address (can change)
- MAC: Physical address (permanent)

### Subnetting

**Subnetting** divides a large network into smaller, manageable sub-networks.

**CIDR Notation:** `192.168.1.0/24`
- `/24` = subnet mask `255.255.255.0`
- Means first 24 bits are network portion, last 8 bits are host portion
- Can have 256 addresses (254 usable: .1 to .254)

**Common Subnet Masks:**
- `/32` - Single host (255.255.255.255)
- `/24` - 256 addresses (255.255.255.0)
- `/16` - 65,536 addresses (255.255.0.0)
- `/8` - 16,777,216 addresses (255.0.0.0)

### `0.0.0.0/0`
- Represents **ALL possible IP addresses**
- The entire IPv4 address space
- Used in routing to mean "default route" (anywhere)
- In security groups: "allow from anywhere/entire internet"

### `ip a` Command
Shows network interface information:
- Interface names (eth0, lo, docker0, etc.)
- IP addresses assigned to each interface
- MAC addresses
- Network state (UP/DOWN)
- MTU (Maximum Transmission Unit)
- Broadcast addresses

Example output:
```bash
1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo
    
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.100/24
    link/ether 00:1a:2b:3c:4d:5e
```

### Network Stack Components
1. **IP Address** - Network identifier
2. **Port Ranges** - 0-65535 (identifies application/service)
3. **Sockets** - Endpoint for sending/receiving data (IP + Port)
4. **Routing Tables** - Where to send packets
5. **Firewall Rules** - Traffic filtering
6. **DNS Resolution** - Domain name to IP mapping

---

## Docker Commands & Options

### Basic Docker Run
```bash
docker run [OPTIONS] IMAGE [COMMAND]
```

### `-d` Flag (Detached Mode)
- **d** = detached
- Runs container in **background**
- Returns container ID immediately
- Container continues running even after command exits
- Access with `docker attach` or `docker logs`

```bash
docker run -d nginx
# Returns: 3f4d7a2b9c1e... (container ID)
# Container runs in background
```

### `-p` Flag (Port Mapping)
- **p** = publish/port
- Maps host port to container port
- Format: `-p HOST_PORT:CONTAINER_PORT`
- Allows external access to containerized services

```bash
docker run -p 8080:80 nginx
# Maps host port 8080 → container port 80
# Access via http://localhost:8080
```

### `-it` Flags (Interactive Terminal)
- **i** = interactive (keeps STDIN open)
- **t** = tty (allocates pseudo-terminal)
- Combined: allows interactive shell access

```bash
docker run -it ubuntu bash
# Opens interactive bash shell inside Ubuntu container
```

### Docker Inspect
```bash
docker container inspect <container_id>
```
Returns detailed JSON with:
- IP address
- Network settings
- Mounted volumes
- Environment variables
- Resource limits
- And much more

Find IP specifically:
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_id>
```

### Why Does `docker run ubuntu` Exit Immediately?

**Containers need a running process to stay alive.**

When you run:
```bash
docker run ubuntu
```

What happens:
1. Container starts
2. Runs default command (usually `/bin/bash`)
3. No interactive terminal, bash exits immediately
4. No running process → container stops

**Solutions:**

```bash
# Keep container alive with long-running process
docker run ubuntu sleep infinity

# Interactive session
docker run -it ubuntu bash

# Run specific service
docker run ubuntu tail -f /dev/null
```

**Key Principle:** Container lifecycle = main process lifecycle

---

## Background Processes

### `sleep 2000 &`
- `sleep 2000` - Pause for 2000 seconds
- `&` - Run in **background**
- Returns job number and PID
- Terminal remains available for other commands

**Managing Background Jobs:**
```bash
jobs              # List background jobs
fg %1             # Bring job 1 to foreground
bg %1             # Resume job 1 in background
kill %1           # Kill job 1
```

---

## Container Internals

### How Containers Achieve Isolation

Containers use **Linux kernel features** to create isolated environments without full virtualization.

### 1. Namespaces

**Namespaces** provide isolation - each namespace type isolates a different resource.

#### PID Namespace (Process ID)
- Isolates process IDs
- Container sees its own process tree starting from PID 1
- Processes inside can't see host processes
- Each container thinks it's alone on the system

```bash
# Inside container
ps aux  # Shows only container processes

# On host
ps aux  # Shows all processes including container processes
```

#### Network Namespace
- Isolates network interfaces, IP addresses, routing tables
- Each container gets its own:
  - Network interface (usually veth pair)
  - IP address
  - Routing table
  - Firewall rules (iptables)
  - Ports (can have same port in different containers)

#### Mount Namespace
- Isolates filesystem mount points
- Each container has its own root filesystem
- Can't see host filesystem (unless explicitly mounted)
- Supports layered filesystems (Union FS)

#### UTS Namespace (Unix Timesharing System)
- Isolates hostname and domain name
- Each container can have different hostname
- `hostname` command shows container-specific name

#### IPC Namespace (Inter-Process Communication)
- Isolates IPC resources:
  - Message queues
  - Semaphores
  - Shared memory segments
- Processes in different containers can't communicate via IPC

#### User Namespace
- Isolates user and group IDs
- Can be root (UID 0) inside container
- But mapped to unprivileged user on host
- Enhanced security

#### Cgroup Namespace
- Isolates cgroup root directory
- Virtualizes view of `/proc/self/cgroup`

### 2. Control Groups (cgroups)

**Cgroups** (Control Groups) limit and monitor resource usage.

**Purpose:** Prevent one container from consuming all host resources.

#### Resource Types Controlled:

**CPU:**
```bash
docker run --cpus="1.5" nginx
# Container can use maximum 1.5 CPU cores
```

**Memory:**
```bash
docker run --memory="512m" nginx
# Container limited to 512MB RAM
```

**Disk I/O:**
```bash
docker run --device-read-bps /dev/sda:1mb nginx
# Limit read speed to 1MB/s
```

**Network Bandwidth:**
- Can be limited using tc (traffic control)

**PIDs (Process Count):**
```bash
docker run --pids-limit=100 nginx
# Maximum 100 processes
```

#### Cgroup Subsystems:
1. **cpu** - CPU time distribution
2. **cpuacct** - CPU accounting
3. **cpuset** - CPU and memory node binding
4. **memory** - Memory limits and reporting
5. **blkio** - Block device I/O
6. **devices** - Device access control
7. **freezer** - Suspend/resume containers
8. **net_cls** - Network packet tagging
9. **net_prio** - Network priority
10. **pids** - Process number limits

### 3. Union File Systems (UnionFS)

Enables **layered filesystem** approach:

```
┌─────────────────────────┐
│  Container Layer (R/W)  │  ← Your changes
├─────────────────────────┤
│  App Layer (R/O)        │  ← Your application
├─────────────────────────┤
│  Dependencies (R/O)     │  ← Libraries
├─────────────────────────┤
│  Base OS Layer (R/O)    │  ← Ubuntu/Alpine
└─────────────────────────┘
```

**Benefits:**
- **Efficient storage** - Layers shared between containers
- **Fast startup** - Only create writable layer
- **Easy updates** - Replace single layer

**Common implementations:**
- OverlayFS (most common)
- AUFS
- Btrfs
- ZFS
- Device Mapper

### 4. Capabilities

Linux capabilities divide root privileges into distinct units:

Instead of all-or-nothing root access, containers can have specific capabilities:

```bash
docker run --cap-add=NET_ADMIN nginx  # Add network admin capability
docker run --cap-drop=ALL nginx       # Drop all capabilities
```

**Common capabilities:**
- CAP_NET_ADMIN - Network configuration
- CAP_SYS_ADMIN - System administration
- CAP_SYS_TIME - Change system clock
- CAP_CHOWN - Change file ownership

### 5. Seccomp (Secure Computing Mode)

Filters system calls that container can make:

- Default Docker profile blocks ~44 dangerous syscalls
- Prevents:
  - Kernel module loading
  - Changing kernel parameters
  - Mounting filesystems
  - Other privileged operations

### 6. AppArmor / SELinux

**Mandatory Access Control (MAC)** systems:

- Define what processes can access
- Additional security layer beyond user permissions
- Docker has default profiles

### Complete Container Stack

```
┌──────────────────────────────────────────┐
│         Application Process              │
├──────────────────────────────────────────┤
│         Runtime (containerd/runc)        │
├──────────────────────────────────────────┤
│                                          │
│  ┌────────────┐      ┌──────────────┐   │
│  │ Namespaces │      │   Cgroups    │   │
│  │ Isolation  │      │   Limits     │   │
│  └────────────┘      └──────────────┘   │
│                                          │
│  ┌────────────┐      ┌──────────────┐   │
│  │  UnionFS   │      │ Capabilities │   │
│  │  Layers    │      │   Seccomp    │   │
│  └────────────┘      └──────────────┘   │
│                                          │
├──────────────────────────────────────────┤
│         Linux Kernel                     │
└──────────────────────────────────────────┘
```

---

## AWS Concepts

### Federated Account

**Federation** allows users to access AWS using existing identity systems (not AWS IAM users).

**How it works:**
1. User authenticates with corporate identity provider (IdP)
   - Examples: Active Directory, Okta, Google Workspace
2. IdP issues SAML token or uses OpenID Connect
3. AWS exchanges token for temporary credentials
4. User accesses AWS with temporary credentials (no permanent IAM users)

**Benefits:**
- **Single Sign-On (SSO)** - One login for multiple systems
- **Centralized management** - Control access in one place
- **No password management** - No AWS passwords to maintain
- **Temporary credentials** - Automatic expiration (more secure)
- **Compliance** - Meet corporate identity requirements

**Common Federation Methods:**
- SAML 2.0 (Security Assertion Markup Language)
- AWS SSO (now AWS IAM Identity Center)
- OpenID Connect (OIDC)
- Web Identity Federation (for mobile/web apps)

**Use Cases:**
- Enterprise employee access
- Cross-account access
- Mobile app authentication
- Web application authentication

---

## RPC (Remote Procedure Call)

**RPC** allows a program to execute code on another computer/process as if it were local.

**Concept:**
```
┌─────────────┐         Network          ┌─────────────┐
│   Client    │ ──────────────────────→  │   Server    │
│ calls func()│                          │ runs func() │
│             │ ←──────────────────────  │ returns val │
└─────────────┘                          └─────────────┘
```

**How it works:**
1. Client calls function (stub)
2. Stub marshals parameters (serializes)
3. Sends request over network
4. Server unmarshals parameters
5. Executes actual function
6. Returns result (marshaled)
7. Client receives result (unmarshaled)

**Popular RPC Frameworks:**
- **gRPC** - Google's modern RPC (uses Protocol Buffers)
- **Apache Thrift** - Facebook's RPC
- **JSON-RPC** - Simple, uses JSON
- **XML-RPC** - Uses XML (older)

**Advantages:**
- Call remote functions like local ones
- Language-agnostic (usually)
- Efficient binary protocols (in modern versions)

**Disadvantages:**
- Network latency
- Failure handling complexity
- Versioning challenges

---

## Summary Table

| Concept | Purpose | Example |
|---------|---------|---------|
| **Container** | Isolated application environment | Docker container |
| **Namespace** | Isolate resources | PID, Network, Mount |
| **Cgroup** | Limit resources | CPU, Memory limits |
| **VM** | Full OS virtualization | VirtualBox, VMware |
| **Hypervisor** | Runs VMs | ESXi, KVM |
| **MAC Address** | Hardware identifier | 00:1A:2B:3C:4D:5E |
| **IP Address** | Network identifier | 192.168.1.100 |
| **Subnetting** | Divide networks | /24, /16, /8 |
| **Federation** | Use external identity | AWS SSO, SAML |
| **RPC** | Call remote functions | gRPC, Thrift |

---

## Quick Reference Commands

```bash
# Container Management
docker run -d -p 8080:80 nginx          # Run detached with port mapping
docker run -it ubuntu bash              # Interactive shell
docker ps                               # List running containers
docker ps -a                            # List all containers
docker stop <container_id>              # Stop container
docker rm <container_id>                # Remove container
docker inspect <container_id>           # Detailed info

# Networking
ip a                                    # Show network interfaces
hostname                                # Show system hostname

# Process Management
ps -ef                                  # List all processes
sleep 2000 &                            # Run in background
jobs                                    # List background jobs
fg %1                                   # Bring job to foreground

# History
history                                 # Show command history
!123                                    # Re-run command 123
```

---

## Interview Questions & Answers

### Beginner Level

#### Q1: What is the difference between a Container and a Virtual Machine?

**Answer:**
- **VM** runs a complete OS with its own kernel, while **containers** share the host OS kernel
- **VMs** are isolated at hardware level via hypervisor, **containers** use Linux namespaces and cgroups
- **VMs** take minutes to boot and use GBs of space, **containers** start in seconds and use MBs
- **VMs** have complete isolation, **containers** have process-level isolation
- **Use VMs** when you need complete OS isolation or different OS types; **use containers** for microservices and rapid deployment

#### Q2: Why does `docker run ubuntu` exit immediately?

**Answer:**
Containers need a running foreground process to stay alive. When you run `docker run ubuntu`, it:
1. Starts the container
2. Executes the default command (typically `/bin/bash`)
3. Since there's no interactive terminal (`-it` flag), bash has nothing to do and exits
4. When the main process exits, the container stops

**Solutions:**
```bash
docker run -it ubuntu bash           # Interactive shell
docker run ubuntu sleep infinity     # Keep alive with long process
docker run -d ubuntu tail -f /dev/null  # Background with dummy process
```

#### Q3: What does the `-d` flag do in Docker?

**Answer:**
The `-d` (detached) flag runs the container in the **background**:
- Container runs without blocking your terminal
- Returns container ID immediately
- You can continue using terminal for other commands
- Access logs with `docker logs <container_id>`
- Attach back with `docker attach <container_id>`

#### Q4: Explain the `-p` flag in Docker.

**Answer:**
The `-p` (publish/port) flag maps ports from container to host:
- Format: `-p HOST_PORT:CONTAINER_PORT`
- Example: `-p 8080:80` maps host port 8080 to container port 80
- Allows external access to containerized services
- Can map multiple ports: `-p 8080:80 -p 443:443`
- Use `-P` (capital P) to auto-assign random host ports

#### Q5: What is a Docker image vs a Docker container?

**Answer:**
- **Image**: Read-only template with application and dependencies (like a class in OOP)
- **Container**: Running instance of an image (like an object/instance)
- One image can create multiple containers
- Images are built from Dockerfiles
- Containers add a writable layer on top of image

```bash
docker images          # List images
docker ps              # List running containers
docker run nginx       # Creates container from nginx image
```

### Intermediate Level

#### Q6: Explain Docker layered filesystem and how it saves space.

**Answer:**
Docker uses **Union File System** with read-only layers:

```
Container Layer (R/W)  ← Your changes
    ↓
App Layer (R/O)        ← Application files
    ↓
Dependencies (R/O)     ← Libraries
    ↓
Base OS Layer (R/O)    ← Ubuntu/Alpine
```

**Benefits:**
- **Shared layers**: Multiple containers using same base image share those layers
- **Efficient storage**: If 10 containers use Ubuntu base, only one copy of Ubuntu stored
- **Fast startup**: Only writable layer created per container
- **Easy updates**: Update single layer, all containers benefit

**Example:**
- nginx:latest image might be 133MB
- 100 containers from this image don't use 13.3GB
- They share base layers, only writable layers differ (few KBs each)

#### Q7: What are Docker namespaces? Name and explain at least 4 types.

**Answer:**
**Namespaces** provide isolation by virtualizing system resources:

1. **PID Namespace**: 
   - Isolates process IDs
   - Container sees its own process tree starting from PID 1
   - Can't see host processes

2. **Network Namespace**:
   - Isolates network stack (interfaces, IPs, routing, firewall)
   - Each container gets own network configuration
   - Containers can use same ports without conflict

3. **Mount Namespace**:
   - Isolates filesystem mount points
   - Each container has its own root filesystem
   - Can't access host files unless explicitly mounted

4. **UTS Namespace**:
   - Isolates hostname and domain name
   - Each container can have different hostname

5. **IPC Namespace**:
   - Isolates Inter-Process Communication (message queues, semaphores, shared memory)

6. **User Namespace**:
   - Maps container users to different host users
   - Can be root inside container but unprivileged on host

#### Q8: What are cgroups and why are they important?

**Answer:**
**Cgroups** (Control Groups) limit and monitor resource usage to prevent resource exhaustion:

**Controlled Resources:**
- **CPU**: Limit CPU cores/time
- **Memory**: Set RAM limits
- **Disk I/O**: Control read/write speeds
- **Network**: Bandwidth limits
- **PIDs**: Max number of processes

**Why important:**
- **Prevent resource hogging**: One container can't consume all host resources
- **Ensure fairness**: Guarantee resources to critical containers
- **Multi-tenancy**: Run untrusted containers safely
- **Cost control**: Enforce resource quotas

**Example:**
```bash
docker run --cpus="2" --memory="1g" --pids-limit=100 nginx
```

#### Q9: How do you troubleshoot a container that keeps restarting?

**Answer:**
**Steps:**

1. **Check logs:**
   ```bash
   docker logs <container_id>
   docker logs --tail 50 <container_id>  # Last 50 lines
   ```

2. **Inspect container:**
   ```bash
   docker inspect <container_id>
   # Check "State" section for exit code and error
   ```

3. **Check resource limits:**
   - Out of memory (OOM)
   - CPU throttling
   - Disk space full

4. **Verify configuration:**
   - Environment variables
   - Volume mounts
   - Network connectivity
   - Port conflicts

5. **Run interactively for debugging:**
   ```bash
   docker run -it <image> bash
   # Test commands manually
   ```

6. **Check dependencies:**
   - Database connectivity
   - External service availability
   - Configuration files

**Common causes:**
- Application crash on startup
- Missing environment variables
- Port already in use
- Insufficient memory
- Failed health checks

#### Q10: What is the difference between CMD and ENTRYPOINT in Dockerfile?

**Answer:**

**CMD:**
- Provides default command and/or parameters
- Can be **overridden** from `docker run`
- Used for default behavior that users might change

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
# Override: docker run myimage /bin/bash
```

**ENTRYPOINT:**
- Sets the main executable
- **Not easily overridden** (need --entrypoint flag)
- Used when container should always run specific executable

```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
# docker run myimage -t  → Runs: nginx -t
```

**Best Practice:**
```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
# Always runs python, but can change which script
# docker run myimage test.py  → python test.py
```

### Advanced Level

#### Q11: Explain Docker networking modes and when to use each.

**Answer:**

1. **Bridge (Default)**:
   - Containers on same bridge can communicate
   - Isolated from host network
   - NAT for external connectivity
   - **Use when**: Multiple containers need to communicate on single host

2. **Host**:
   - Container uses host's network directly
   - No network isolation
   - Best performance (no NAT overhead)
   - **Use when**: Need maximum network performance or access host services

3. **None**:
   - No networking
   - Complete network isolation
   - **Use when**: Processing sensitive data, no network needed

4. **Container**:
   - Share network namespace with another container
   - Same IP, same ports
   - **Use when**: Sidecar pattern (e.g., logging, monitoring)

5. **Overlay**:
   - Multi-host networking
   - Used in Docker Swarm/Kubernetes
   - **Use when**: Containers across multiple hosts need to communicate

6. **Macvlan**:
   - Assigns MAC address to container
   - Appears as physical device on network
   - **Use when**: Legacy applications expect physical network

```bash
docker run --network=host nginx
docker run --network=none alpine
docker network create my-bridge
docker run --network=my-bridge nginx
```

#### Q12: How does Docker achieve better performance than VMs?

**Answer:**

**Key Factors:**

1. **Shared Kernel**:
   - No OS boot time
   - No kernel overhead per container
   - Direct syscalls to host kernel

2. **No Hardware Emulation**:
   - VMs emulate hardware (CPU, memory, devices)
   - Containers use host hardware directly
   - Hypervisor adds CPU/memory overhead (5-10%)

3. **Lightweight**:
   - No full OS per instance
   - Only application and dependencies
   - Less memory footprint

4. **Copy-on-Write Filesystem**:
   - Shared layers across containers
   - Only changes written to disk
   - Faster I/O for reads

5. **Fast Startup**:
   - No BIOS/bootloader
   - No OS initialization
   - Just process fork

**Performance Comparison:**
- **VM boot**: 1-3 minutes
- **Container start**: 1-3 seconds
- **VM overhead**: 5-10% CPU/memory
- **Container overhead**: ~1-2%

**Tradeoff:**
- VMs: Complete isolation, can run different OS types
- Containers: Better performance, must share compatible kernel

#### Q13: What are multi-stage builds in Docker and why use them?

**Answer:**

**Multi-stage builds** use multiple FROM statements to create smaller, more secure final images.

**Problem it solves:**
- Build tools and dependencies bloat image size
- Security risk from build tools in production
- Complex build scripts

**Example:**

```dockerfile
# Stage 1: Build
FROM golang:1.20 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

# Stage 2: Final minimal image
FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

**Benefits:**
1. **Smaller images**: golang:1.20 is ~800MB, alpine is ~5MB
2. **Security**: No build tools, compilers, source code in final image
3. **Faster deployments**: Less data to transfer
4. **Clear separation**: Build vs runtime dependencies

**Real-world impact:**
- Before: 800MB image with Go compiler
- After: 15MB image with just binary
- 98% size reduction!

#### Q14: Explain Docker storage drivers and which to use when.

**Answer:**

**Storage drivers** manage container layers and image layers.

**Common Drivers:**

1. **overlay2** (Recommended):
   - Most modern and performant
   - Works with Linux kernel 4.0+
   - Best for most use cases
   - Good performance for read/write

2. **aufs**:
   - Older, being deprecated
   - Good read performance
   - High latency for write-intensive workloads

3. **devicemapper**:
   - Works with older kernels
   - Can be configured for better performance
   - More complex setup

4. **btrfs**:
   - If host uses btrfs filesystem
   - Good for snapshotting
   - Advanced features (deduplication, compression)

5. **zfs**:
   - If host uses ZFS
   - Enterprise-grade features
   - Excellent for databases

**Choosing:**
- **Default**: Use overlay2 (works for 95% of cases)
- **Legacy systems**: devicemapper
- **Existing btrfs/zfs**: Use matching driver
- **High performance needs**: Test overlay2 vs devicemapper with your workload

**Check current driver:**
```bash
docker info | grep "Storage Driver"
```

#### Q15: How would you secure a production Docker environment?

**Answer:**

**1. Image Security:**
- Use official/verified images
- Scan images for vulnerabilities: `docker scan <image>`
- Use minimal base images (alpine, distroless)
- Don't use `latest` tag in production
- Implement image signing (Docker Content Trust)

```bash
export DOCKER_CONTENT_TRUST=1  # Enable image signing
```

**2. Runtime Security:**
- Run containers as non-root user:
  ```dockerfile
  USER nobody
  ```
- Drop unnecessary capabilities:
  ```bash
  docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
  ```
- Use read-only filesystems:
  ```bash
  docker run --read-only nginx
  ```
- Enable user namespaces (map root to non-root on host)

**3. Network Security:**
- Isolate containers with custom networks
- Limit port exposure (only expose what's needed)
- Use secrets management (Docker secrets, Vault)
- Never hardcode credentials in images

**4. Resource Limits:**
```bash
docker run --memory="512m" --cpus="1" \
  --pids-limit=100 nginx
```

**5. Monitoring & Auditing:**
- Enable Docker daemon logging
- Monitor container behavior (Falco, Sysdig)
- Regular security audits
- Use SELinux/AppArmor profiles

**6. Host Security:**
- Keep Docker daemon updated
- Secure Docker socket (don't expose TCP without TLS)
- Use least privilege for users
- Regular host patching

**7. Supply Chain:**
- Verify image sources
- Private registry for internal images
- Automated vulnerability scanning in CI/CD

#### Q16: Explain the difference between Docker volumes, bind mounts, and tmpfs.

**Answer:**

**1. Volumes (Recommended):**
- Managed by Docker
- Stored in `/var/lib/docker/volumes/`
- Independent of host directory structure
- Can be shared between containers
- Persist after container removal

```bash
docker volume create mydata
docker run -v mydata:/app/data nginx
```

**When to use:**
- Production data persistence
- Sharing data between containers
- Backing up data
- Remote storage drivers

**2. Bind Mounts:**
- Maps host directory to container
- Full host path required
- Can mount any host location
- Changes reflected immediately on both sides

```bash
docker run -v /host/path:/container/path nginx
# Or newer syntax:
docker run --mount type=bind,source=/host/path,target=/container/path nginx
```

**When to use:**
- Development (live code reload)
- Configuration files
- Sharing files between host and container

**3. tmpfs (Temporary Filesystem):**
- Stored in host memory (RAM)
- Never written to disk
- Removed when container stops
- Fast but limited by RAM

```bash
docker run --tmpfs /app/cache:rw,size=100m nginx
```

**When to use:**
- Temporary caching
- Sensitive data (passwords, tokens)
- High-speed temporary storage

**Comparison:**

| Feature | Volume | Bind Mount | tmpfs |
|---------|--------|------------|-------|
| Managed by Docker | Yes | No | No |
| Persists data | Yes | Yes | No |
| Performance | Good | Good | Fastest |
| Security | Better | Risky | Secure |
| Backup easy | Yes | Manual | N/A |

#### Q17: What happens when you run `docker run`? Explain the complete flow.

**Answer:**

**Complete Docker Run Flow:**

1. **Image Check (Local)**:
   - Docker checks if image exists locally
   - Looks in `/var/lib/docker/`

2. **Image Pull (if needed)**:
   - If not local, pulls from registry (Docker Hub by default)
   - Downloads layers in parallel
   - Verifies checksums

3. **Container Creation**:
   - Creates container from image
   - Generates unique container ID
   - Sets up container config (env vars, ports, volumes)

4. **Filesystem Setup**:
   - Creates Union FS layers
   - Mounts volumes and bind mounts
   - Creates writable layer on top

5. **Namespace Creation**:
   - Creates PID namespace (process isolation)
   - Creates Network namespace (network isolation)
   - Creates Mount namespace (filesystem isolation)
   - Creates UTS, IPC, User namespaces

6. **Cgroup Setup**:
   - Creates cgroups for resource limits
   - Sets CPU, memory, I/O limits
   - Configures accounting

7. **Network Configuration**:
   - Assigns IP address (from docker network)
   - Creates veth pair (virtual ethernet)
   - Configures iptables rules
   - Sets up DNS resolution

8. **Security Setup**:
   - Applies AppArmor/SELinux profiles
   - Sets capabilities
   - Configures seccomp filters

9. **Process Execution**:
   - Executes ENTRYPOINT or CMD
   - Process gets PID 1 inside container
   - STDIN/STDOUT connected if `-it` flag

10. **Container State**:
    - Container state: Running
    - Docker daemon monitors process
    - Logs collected

**Visual Flow:**
```
docker run → Check Image → Pull if needed → Create Container →
Setup Namespaces → Setup Cgroups → Configure Network →
Apply Security → Start Process → Running Container
```

#### Q18: How do you optimize a Dockerfile for production?

**Answer:**

**Best Practices:**

**1. Use Minimal Base Images:**
```dockerfile
# Bad: 1GB+
FROM ubuntu:latest

# Good: ~5MB
FROM alpine:latest

# Best for some: <2MB
FROM gcr.io/distroless/base
```

**2. Multi-stage Builds:**
```dockerfile
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

**3. Layer Optimization:**
```dockerfile
# Bad: Each RUN creates a layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim

# Good: Single layer
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*
```

**4. Leverage Build Cache:**
```dockerfile
# Copy dependencies first (changes less frequently)
COPY package*.json ./
RUN npm install

# Copy source code last (changes frequently)
COPY . .
```

**5. .dockerignore File:**
```
node_modules
.git
.env
*.md
.dockerignore
Dockerfile
```

**6. Use Specific Tags:**
```dockerfile
# Bad
FROM node:latest

# Good
FROM node:16.14.0-alpine3.15
```

**7. Security Hardening:**
```dockerfile
# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

# Switch to non-root
USER appuser

# Use read-only root filesystem where possible
# Run with: docker run --read-only
```

**8. Minimize Layers:**
- Each RUN, COPY, ADD creates a layer
- Combine related commands
- Remove temporary files in same layer

**9. Metadata:**
```dockerfile
LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="Production-ready app"
```

**10. Health Checks:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
```

**Result:**
- **Before**: 1.5GB, 15 layers, 5 min build
- **After**: 50MB, 8 layers, 1 min build

---

## Practice Scenarios

### Scenario 1: Container Not Accessible
**Problem**: Container running but can't access via browser.

**Debugging:**
```bash
docker ps  # Verify container running
docker inspect <id> | grep IPAddress  # Check IP
docker logs <id>  # Check application logs
docker port <id>  # Check port mappings
curl http://localhost:8080  # Test locally
```

**Common Issues:**
- Forgot `-p` flag
- Wrong port mapping
- Application not binding to 0.0.0.0
- Firewall rules

### Scenario 2: Out of Disk Space
**Problem**: Can't create new containers.

**Solution:**
```bash
docker system df  # Check disk usage
docker system prune  # Remove unused data
docker volume prune  # Remove unused volumes
docker image prune -a  # Remove unused images

# More aggressive cleanup
docker rm $(docker ps -aq)  # Remove all containers
docker rmi $(docker images -q)  # Remove all images
```

### Scenario 3: Performance Issues
**Problem**: Container running slowly.

**Investigation:**
```bash
docker stats  # Real-time resource usage
docker inspect <id> | grep -A 10 "Memory"  # Check limits
docker top <id>  # Check processes inside
docker logs --tail 100 <id>  # Check for errors
```

**Solutions:**
- Increase resource limits
- Optimize application
- Check host resources
- Review storage driver

---

*This guide covers fundamental concepts of containers, Docker, networking, and related technologies. Practice these commands and concepts to build practical understanding.*