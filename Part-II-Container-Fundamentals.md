# The ShopSphere DevOps Book
## Part II — Container Fundamentals

---

### Where we left off

ShopSphere's app runs as plain processes on one shared Linux server. Recall the problems: "works on my machine," risky manual deploys, no isolation between processes, painful scaling, slow onboarding.

Today, your manager asks you to look into "containerizing" the app. Before you write a single Dockerfile, we need to understand what a container actually *is* — because if you skip this, Docker will always feel like a magic black box, and you won't be able to debug it when it misbehaves.

---

## Chapter 1.1 — What Problem Are We Actually Solving?

Let's state the core problem precisely: ShopSphere needs a way to package an application together with everything it needs to run — the exact language runtime, libraries, and configuration — so that it behaves identically on a developer's laptop, in CI, and in production. And it needs to run several of these packaged applications on the same machine without them interfering with each other.

There are two broad approaches to this: **virtual machines** and **containers**. Let's compare them properly, because this comparison comes up in almost every DevOps interview.

### Virtualization and VMs

**Simple explanation:** A virtual machine is a fake computer running inside a real computer — complete with its own pretend hardware and its own full operating system.

**Proper definition:** **Virtualization** uses a **hypervisor** (software like VMware, KVM, or Hyper-V) to partition a physical machine's hardware resources (CPU, memory, disk) into multiple isolated virtual machines, each running a complete, independent operating system.

```text
        Physical Server (Hardware)
                    |
               Hypervisor
         /          |          \
      VM 1         VM 2         VM 3
   Full OS       Full OS       Full OS
   (Ubuntu)      (Ubuntu)      (CentOS)
      |             |             |
     App A         App B         App C
```

**Why it exists:** before virtualization, "one application per physical server" was common, because running multiple unrelated apps on one machine was risky and hard to isolate. Buying, racking, and powering a separate physical machine for every application is enormously wasteful. Virtualization let a company run many isolated workloads on one physical box.

**The cost:** every VM carries a full copy of an operating system — its own kernel, its own set of system processes, its own multi-second-to-minutes boot time, and a meaningful chunk of memory and disk just to exist, before your application even starts. If ShopSphere ran the frontend, backend, and worker each in their own VM, that's three full operating systems' worth of overhead for three programs that might individually need only a small fraction of a CPU core.

### Containers: a different idea

**Simple explanation:** A container doesn't pretend to be a whole computer. It's a normal process running on the *same* operating system as everything else on the machine — it just has a strong, isolated illusion that it's alone, with its own filesystem, its own network stack, its own process list.

**Proper definition:** A **container** is a lightweight, isolated user-space instance created using operating-system-level virtualization — multiple containers share the same host kernel, but are isolated from one another and from the host using kernel features (which we're about to unpack).

```text
        Physical Server (Hardware)
                    |
              Host Operating System
              (one shared kernel)
                    |
             Container Engine
         /          |          \
   Container 1  Container 2  Container 3
   (Frontend)    (Backend)    (Worker)
   own filesystem view, own process view, own network view
   — but all sharing the SAME underlying kernel
```

**The key difference, stated precisely:** a VM virtualizes the *hardware* and boots a full OS on top of it. A container virtualizes the *operating system's process and resource view* — no separate kernel, no separate boot process. That's why a container starts in milliseconds instead of minutes, and why you can run dozens of containers on a machine that could maybe run three or four VMs.

| | Virtual Machine | Container |
|---|---|---|
| Isolation boundary | Hardware (via hypervisor) | OS kernel (via namespaces/cgroups) |
| Includes its own kernel? | Yes, full OS | No, shares host kernel |
| Startup time | Seconds to minutes | Milliseconds to seconds |
| Typical size | Gigabytes | Megabytes to low gigabytes |
| Isolation strength | Very strong (separate kernel) | Strong, but weaker than a VM |
| Density per host | Low (few per machine) | High (many per machine) |

**Interview question (beginner):** "What is the main difference between a VM and a container?" — A VM virtualizes hardware and runs a full separate OS with its own kernel per VM; a container virtualizes the OS itself and shares the host's kernel across containers, which makes it far lighter and faster to start.

**Interview question (advanced, and a genuinely important nuance):** "Is a container less secure than a VM?" — Generally yes, in the sense that containers share a kernel: a severe kernel-level vulnerability could theoretically let a process escape container isolation and affect the host or other containers, which is architecturally impossible in the same way with separate-kernel VMs. This is exactly why production Kubernetes clusters apply defense-in-depth — non-root containers, read-only filesystems, resource limits, NetworkPolicy, Pod Security Standards — rather than treating container isolation as an absolute security boundary. We'll cover every one of these in Part VIII.

---

## Chapter 1.2 — What's Actually Happening Under the Hood

This is the part most tutorials skip, and it's exactly the part that separates "I can run `docker run`" from "I understand containers." Let's not treat Docker as magic. A container is built from a small number of Linux kernel features, combined together. Docker didn't invent these — it made them dramatically easier to use.

### Linux namespaces

**Simple explanation:** Linux namespaces are one of the mechanisms that let a container have its own isolated view of things like processes, networking, and filesystems — in simple terms, they make a container feel like it has its own small environment, even though it's really just a normal process on the same machine as everything else.

**Proper definition:** A **Linux namespace** is a kernel feature that partitions a specific type of global system resource so that a process (or group of processes) sees its own isolated instance of that resource.

There isn't just one namespace — there are several, each isolating a different thing:

- **PID namespace** — isolates the process ID list. Inside the container, your application might appear as PID 1, even though on the actual host it's really PID 48213. The container can't see, and can't kill, processes outside its own PID namespace.
- **Network namespace** — gives the container its own network interfaces, IP address, routing table, and ports. This is why two different containers can both "listen on port 8000" without conflicting — each has its own private network stack.
- **Mount namespace** — gives the container its own view of the filesystem, so it can have `/app`, `/etc`, `/usr` that look nothing like the host's real filesystem.
- **UTS namespace** — isolates the hostname, so a container can have its own hostname independent of the host machine's.
- **IPC namespace** — isolates inter-process communication resources (like shared memory segments) so containers can't interfere with each other's IPC.
- **User namespace** — allows a process to appear as root *inside* the container while actually mapping to an unprivileged, non-root user on the host — an important security tool we'll return to.

**Why this exists:** the Linux kernel has always supported multiple processes, but historically all processes shared one single, global view of process IDs, network interfaces, mount points, and so on. Namespaces let the kernel give a process (or a tree of processes) its *own* private view of a specific resource, without the overhead of a separate kernel.

**Practical example:** run a container and look inside it:

```bash
docker run -it --rm alpine sh
# inside the container:
ps aux
```

You'll see almost nothing running — maybe just `sh` as PID 1. But on the host machine, running `ps aux` shows dozens or hundreds of processes, including that same container process, just with a completely different PID. Same process, two different views — that's the PID namespace at work.

**How you'd encounter this in a real company:** when someone says "why can't the container see other processes on the host," or "why does `kill -9 1` inside the container not work the way you'd expect," the answer is namespaces.

**Interview question (advanced):** "What Linux kernel feature gives each container its own isolated network stack?" — The network namespace.

### cgroups (control groups)

**Simple explanation:** if namespaces control what a container can *see*, cgroups control how much of the machine's actual resources — CPU, memory — a container is *allowed to use*.

**Proper definition:** **cgroups (control groups)** are a Linux kernel feature that limits, accounts for, and isolates the resource usage (CPU, memory, disk I/O, network bandwidth) of a collection of processes.

**Why this exists:** namespaces alone don't stop one container from consuming all the CPU or memory on a shared machine and starving every other container. A runaway process in one container without resource limits can take down everything else on the same host — exactly the "worker process leaks memory and starves the backend" problem ShopSphere had *before* containerizing.

**Practical example:**

```bash
docker run -d --memory=256m --cpus=0.5 shopsphere-backend
```

This tells the kernel, through cgroups, "this container may use at most 256 MB of memory and half of one CPU core — no more, ever." If the process tries to allocate more memory than that, the kernel will kill it (you'll see this show up later as the dreaded **OOMKilled** status in Kubernetes — same underlying mechanism, just triggered by Kubernetes-managed cgroup limits instead of a Docker flag).

**Where you'll see this constantly:** Kubernetes **resource requests and limits** (Part VIII) are, under the hood, just cgroups configuration applied to the container. Understanding cgroups now means "OOMKilled" won't feel mysterious later — it's the kernel enforcing a memory ceiling, exactly as instructed.

**Interview question (intermediate):** "A container keeps getting killed with an out-of-memory error, even though the host machine has plenty of free RAM. Why might that happen?" — The container likely has a memory limit set via cgroups (directly, or via Kubernetes resource limits) that's lower than what the application actually needs — the host having free memory is irrelevant, because the *container's own ceiling* was hit.

### The overlay filesystem

**Simple explanation:** container images are built in layers, stacked on top of each other like transparent sheets, and the overlay filesystem is the trick that makes many containers share the same underlying layers on disk without duplicating them.

**Proper definition:** **OverlayFS** is a union filesystem that combines multiple directories (layers) into a single merged view. A container image is a stack of read-only layers; when a container runs, a thin writable layer is added on top.

```text
   Writable layer (this container only) ← changes go here
--------------------------------------------
   Layer 3: application code (COPY . .)
--------------------------------------------
   Layer 2: installed dependencies (RUN pip install)
--------------------------------------------
   Layer 1: base OS image (FROM python:3.12-slim)
```

**Why this exists:** without it, every container would need its own full, independent copy of every file in the image — wasteful in disk space and slow to start. With a layered, shared filesystem, ten containers all built from the same base image share the same base-image layers on disk; only each container's own writable layer is unique.

**A crucial consequence you'll meet again in the Docker Images chapter:** because lower layers are shared and read-only, Docker can *cache* them. If you change only your application code (the top layer) and rebuild, Docker reuses the cached dependency-installation layer instead of redoing it — which is exactly why Dockerfile *layer ordering* matters for build speed, something we'll drill into properly soon.

**How you'd encounter this in a real company:** "why is my Docker build so slow" or "why did changing one line in my code invalidate my entire dependency-install cache" are both overlay-filesystem-and-layer-caching questions in disguise.

**Interview question (intermediate):** "Why are Docker images described as 'layered,' and why does that matter for build performance?" — Each instruction in a Dockerfile typically creates a new, cacheable, read-only layer; unchanged layers are reused across builds via the overlay filesystem, so structuring a Dockerfile to put rarely-changing instructions (like dependency installation) before frequently-changing ones (like copying source code) keeps builds fast.

### Container isolation, summarized

Put the three pieces together, and you have the full picture of "what is a container, really":

```text
                A Linux Process
                       |
     +-----------------+-----------------+
     |                                   |
 Namespaces                          cgroups
 (WHAT it can see)              (HOW MUCH it can use)
     |                                   |
 own PIDs, network,              CPU limit, memory limit,
 mounts, hostname, IPC           I/O limit
                       |
                Overlay Filesystem
             (WHERE its files come from —
          layered, shared, cheap to start)
                       |
              = "a container"
```

A container is not a lightweight VM. It's a completely normal Linux process, made to *feel* isolated through namespaces, made to be *resource-constrained* through cgroups, and given a fast, disk-efficient filesystem through OverlayFS. There's no separate kernel, no hypervisor, no virtual hardware. This is why containers start almost instantly and why the same host kernel is shared by every container on that machine — which is also precisely why kernel-level isolation is weaker than VM-level isolation, as we noted above.

---

## Chapter 1.3 — Images, Runtimes, Engines, and OCI

A few more terms you need before we touch Docker directly, because "Docker" is actually several distinct pieces working together, and interviewers will absolutely ask you to distinguish them.

**Container image.** A **container image** is the packaged, read-only bundle of layers (application code, dependencies, runtime, filesystem) described above — it's the *blueprint*. A **container** is a running instance of that image, with a writable layer added on top. This is the same relationship as a class and an object, or a recipe and the meal — one image, many containers can be started from it.

**Container runtime.** The **container runtime** is the low-level component actually responsible for running containers — creating the namespaces, applying cgroup limits, starting the process. The industry-standard low-level runtime is **runc**. Above that often sits a higher-level runtime like **containerd**, which manages images, storage, and networking, and calls down to `runc` to actually start the container process.

**Container engine.** The **container engine** is the user-facing tool — Docker Engine, for example — that provides the CLI, the build system, and the daemon that developers actually interact with. Docker Engine, under the hood, delegates the actual container-running work to containerd, which delegates to runc.

```text
   docker CLI  (what you type)
        |
   Docker Engine / dockerd  (daemon: builds images, manages containers)
        |
    containerd  (manages container lifecycle, image storage)
        |
      runc  (actually creates namespaces/cgroups and starts the process)
        |
   Linux Kernel
```

**OCI (Open Container Initiative).** **OCI** is an open governance body that defines standard specifications — the **OCI Image Spec** (what a container image must look like) and the **OCI Runtime Spec** (how a compliant runtime must run a container). This is *why* an image built with Docker can be run by other tools like Podman or by Kubernetes's container runtime — they all agree on the same standard format, instead of everyone inventing their own incompatible container format.

**Why this matters for your career, concretely:** Kubernetes does not use Docker Engine internally anymore (a change that surprises a lot of people — we'll cover it properly when we get to Kubernetes architecture). It talks to any OCI-compliant runtime, typically containerd directly, through an interface called the **Container Runtime Interface (CRI)**. Knowing that Docker Engine and "the thing that runs containers" are not the same thing is a genuinely common interview differentiator.

**Interview question (advanced):** "Does Kubernetes require Docker to be installed on its nodes?" — No. Kubernetes requires an OCI-compliant, CRI-compatible container runtime — commonly containerd or CRI-O — and does not depend on Docker Engine specifically. Images built by Docker still work fine, because they follow the OCI Image Spec.

### Container lifecycle

One more piece of vocabulary you'll use constantly: the states a container moves through.

```text
   Created  --->  Running  --->  Paused (optional)
                     |
                     v
                  Stopped  --->  Removed
```

- **Created** — the container exists (namespaces/filesystem prepared) but its main process hasn't started yet.
- **Running** — the main process is executing.
- **Paused** — all processes inside are frozen (via a cgroup freezer), without being terminated.
- **Stopped/Exited** — the main process has ended, but the container's filesystem and metadata still exist until it's removed.
- **Removed** — the container and its writable layer are deleted.

This maps directly onto the Docker commands you'll learn next (`docker create`, `docker start`, `docker stop`, `docker rm`), and it maps onto Kubernetes Pod phases later too — same underlying lifecycle, different layer of abstraction.

---

## Chapter 1.4 — Checkpoint

**Beginner:**
1. In your own words, what is a container?
2. What is the difference between a container image and a container?

**Intermediate:**
3. Name the two main Linux kernel mechanisms that make containers possible, and say what each one is responsible for.
4. Why do containers start so much faster than virtual machines?

**Advanced:**
5. Explain what OCI is and why it matters that Kubernetes doesn't depend on Docker Engine specifically.
6. A container is killed unexpectedly, and `docker inspect` shows an OOM-related exit reason. Which kernel mechanism caused that, and why?

**Scenario:**
7. A senior engineer says: "Containers are basically just tiny, fast virtual machines." Push back on this, precisely — what's actually inaccurate about that statement, and why does the distinction matter for security?

---

### Hands-On Lab 1.1 — See namespaces and cgroups for yourself

**Objective:** Directly observe process isolation and resource limits, instead of taking them on faith.

**Prerequisites:** Docker installed (Docker Desktop or Docker Engine on Linux).

**Steps:**

1. Start a container and check its process view from the inside:
   ```bash
   docker run -dit --name pidtest alpine sh
   docker exec pidtest ps aux
   ```
   Notice how few processes are visible.

2. Compare that to the host's view of the same container's process:
   ```bash
   docker inspect --format '{{.State.Pid}}' pidtest
   ps aux | grep <that PID>
   ```
   Same process, visible from the host — but the container itself only sees a tiny isolated slice.

3. Apply and test a memory limit:
   ```bash
   docker run -it --rm --memory=50m alpine sh -c "cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes"
   ```
   You should see a number reflecting the ~50MB limit — this is cgroups, directly visible.

4. Clean up:
   ```bash
   docker rm -f pidtest
   ```

**Expected result:** Inside the container, `ps aux` shows almost nothing; from the host, the same container process is visible among everything else running; the cgroup file shows your memory limit applied.

**Verification:** You directly observed a namespace (limited process visibility) and a cgroup (an enforced memory ceiling) without Docker's higher-level commands hiding what's happening.

**Troubleshooting:** If the `/sys/fs/cgroup` path differs, your system may be using cgroups v2 with a different layout — try `cat /sys/fs/cgroup/memory.max` first; that's the modern (v2) path.

**Challenge:** Start two containers with the same `--name`-free image but different `--cpus` limits, and use `docker stats` to watch their CPU usage ceilings in real time under load (you can generate load inside with `yes > /dev/null &`). *(No solution given — try it, then check `docker stats` yourself; we'll return to CPU limits formally in the Docker Security and Kubernetes Resource Management chapters.)*

---

*End of Part II. Part III moves into Docker itself: Docker's architecture (CLI, daemon, containerd), the full command set explained with real ShopSphere usage, and building the first genuinely production-quality Dockerfile for the backend API — including why the "quick and dirty" version we'll write first is actually a bad Dockerfile, and how to fix it.*
