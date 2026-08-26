# The ShopSphere DevOps Book
## Part VI — Kubernetes Fundamentals

---

### Where we left off

ShopSphere runs locally via Compose, and its backend image is hardened and sitting in ECR. Now imagine six months have passed. ShopSphere had a good quarter. Traffic tripled. The company hired two more engineers. And a new, uncomfortable set of problems has shown up — problems that Docker and Compose, on their own, were never designed to solve.

---

## Chapter 5.1 — What Problem Does Kubernetes Actually Solve?

Let's be concrete about what's actually going wrong at ShopSphere now, because Kubernetes should never be introduced as "the next tool in the tutorial" — it should be introduced because specific, real problems demand it.

**Problem 1 — one server isn't enough, and isn't safe.** ShopSphere is currently running its containers with Compose, on one EC2 instance. That instance is a **single point of failure**: if it goes down — hardware failure, a bad kernel update, AWS having a bad day in that specific data center — the entire application goes down with it, all at once.

**Problem 2 — scaling is manual and slow.** During a flash sale, traffic spikes hard. Right now, "scaling up" means an engineer manually noticing the spike, manually provisioning a bigger or additional server, and manually redeploying — far too slow to react to a traffic spike actually happening *right now*, and it doesn't reliably scale back down afterward either (nobody wants to be the one who remembers to do that at 11 p.m.).

**Problem 3 — deploys are risky, with no clean rollback.** Compose's `up` doesn't have a real concept of gradually replacing running containers with new ones while checking that the new ones are actually healthy before finishing the switch. A bad deploy can mean real downtime, and rolling back means manually reversing whatever was just done, under pressure.

**Problem 4 — no self-healing.** If the backend container crashes at 3 a.m., nothing is watching it and automatically restarting it (Compose's `restart: always` policy helps here, to be fair — but it only covers the single-machine case, and doesn't know anything about *where else* it might reschedule that workload if the whole machine is unhealthy).

**Problem 5 — nothing is multi-machine aware.** Compose fundamentally orchestrates containers on *one* machine. The moment ShopSphere needs to run across several machines — for actual redundancy, for capacity beyond a single box — Compose has no concept of "run this container somewhere in this pool of machines, wherever there's room, and know how to find it afterward."

### What Kubernetes actually is

**Simple explanation:** Kubernetes is a system that manages a pool of machines and a pool of containers together — you tell it "I want three copies of this container running, always, with this much CPU and memory," and it continuously works to make that true, across however many machines you give it, even as machines and containers come and go.

**Proper definition:** **Kubernetes (K8s)** is an open-source **container orchestration platform** that automates the deployment, scaling, networking, and lifecycle management of containerized applications across a cluster of machines.

**The core idea underneath everything Kubernetes does — declarative, desired-state management.** This is genuinely the single most important concept to internalize before anything else in this Part, because every Kubernetes behavior you'll learn from here on is really just this one idea, applied over and over:

> You **declare** what you want ("3 replicas of the backend, this image, this much memory"). Kubernetes continuously **observes** the actual current state, **compares** it to your desired state, and **acts** to close any gap — over and over, forever, without you telling it *how* each time.

```text
   You declare:  "I want 3 backend Pods running"
                          |
                          v
            +----------------------------+
            |   Kubernetes Control Loop   |
            |                              |
            |  Observe actual state        |
            |         |                    |
            |         v                    |
            |  Compare to desired state     |
            |         |                    |
            |         v                    |
            |  Take action to reconcile     |
            |  (start / stop / reschedule)   |
            +----------------------------+
                          |
                          v
              Actual state converges
              toward desired state,
              continuously, forever
```

This pattern — called a **reconciliation loop** or **control loop** — is why, if a backend Pod crashes, Kubernetes doesn't need a special "restart crashed pods" feature bolted on; restarting is just the natural, automatic consequence of "actual state (2 running) no longer matches desired state (3 running)," and Kubernetes closing that gap. Every single feature in this Part — self-healing, scaling, rolling deployments — is this exact same loop, applied to a different kind of desired state. Hold onto this. It will make the rest of Kubernetes dramatically less mysterious.

**Interview question (beginner, but the single most important one in this whole Part):** "What does it mean for Kubernetes to be 'declarative'?" — Instead of issuing step-by-step imperative commands ("start this container, now start that one"), you describe the *desired end state* of your system, and Kubernetes continuously and automatically works to make the actual state match it — including automatically correcting drift caused by failures, without additional manual intervention.

---

## Chapter 5.2 — Kubernetes Architecture: The Big Picture

A Kubernetes **cluster** is made of two categories of machines, doing fundamentally different jobs.

```text
                         KUBERNETES CLUSTER
     +--------------------------------------------------------+
     |                    CONTROL PLANE                        |
     |  (the "brain" — makes decisions, doesn't run your apps)  |
     |                                                            |
     |   API Server   etcd   Scheduler   Controller Manager       |
     +--------------------------------------------------------+
                              |
              +---------------+---------------+
              |                               |
     +-----------------+             +-----------------+
     |   WORKER NODE 1  |             |   WORKER NODE 2  |
     |  (runs your apps) |             |  (runs your apps) |
     |                    |             |                    |
     |  kubelet           |             |  kubelet           |
     |  kube-proxy         |             |  kube-proxy         |
     |  Container Runtime   |             |  Container Runtime   |
     |                       |             |                       |
     |  [backend Pod]         |             |  [backend Pod]         |
     |  [frontend Pod]         |             |  [worker Pod]           |
     +-----------------+             +-----------------+
```

**Node.** A **node** is a single machine (physical or virtual) that's part of the cluster — either a control plane node or a worker node.

**Cluster.** A **cluster** is the entire collection of nodes — control plane plus workers — managed together as one system.

Let's go through the control plane and worker node components properly, one at a time, because interviewers absolutely expect you to be able to name each one and its specific job.

---

## Chapter 5.3 — The Control Plane, Component by Component

The control plane is where every decision gets made — but critically, it does not run your application containers itself. It's the brain, not the muscle.

### API Server

**Simple explanation:** the API Server is the front door to the entire cluster — literally everything, including `kubectl`, talks to Kubernetes exclusively through it. Nothing reaches into the cluster's internals directly.

**Proper definition:** the **kube-apiserver** is a REST API that validates and processes all requests to read or modify cluster state, and is the only component that talks directly to `etcd`.

**Why it exists as a single, deliberate chokepoint:** having one consistent, authenticated, validated entry point for every change to the cluster — instead of many components independently writing to shared state — is what makes the whole system's state consistent and secure. Every single other component we're about to cover — including other control plane components — talks to the cluster *through* the API Server too, never by reaching around it.

### etcd

**Simple explanation:** etcd is the cluster's single source of truth — a database that stores the entire actual and desired state of everything in the cluster.

**Proper definition:** **etcd** is a distributed, consistent key-value store that holds all cluster state — every object's specification and status.

**Why this matters:** when we said "you declare 3 backend replicas," that declaration is written into etcd. When the Scheduler decides where a Pod runs, that decision is written into etcd. **etcd is the actual state Kubernetes is built on** — the API Server is the front door, but etcd is where everything actually lives.

**A genuinely important production fact:** etcd backups are one of the single most critical operational responsibilities for anyone running self-managed Kubernetes — lose etcd with no backup, and you've lost the record of your entire cluster's state. We'll return to this directly in Production Operations (Part XIV). One meaningful reason companies choose a managed control plane like EKS (Part XI) is that AWS takes over etcd's operation and backup responsibility entirely.

### Scheduler

**Simple explanation:** when a new Pod needs to run somewhere, the Scheduler's job is to pick which specific worker node it should run on.

**Proper definition:** the **kube-scheduler** watches for newly created Pods that don't yet have a node assigned, and assigns each one to a suitable node based on resource availability, constraints, and scheduling rules.

**How it decides, at a high level:** it filters out nodes that can't satisfy the Pod's requirements (not enough CPU/memory available, a required label missing — we'll cover this properly in Part VIII's Scheduling chapter with node affinity, taints, and tolerations), then ranks the remaining eligible nodes and picks the best fit.

**Important nuance:** the Scheduler only *decides* where a Pod should run — it doesn't actually start anything itself. That's the kubelet's job, on the chosen node, as we're about to see.

### Controller Manager

**Simple explanation:** this runs the actual reconciliation loops from Chapter 5.1 — it's the component continuously comparing "what you asked for" against "what's actually true" and taking corrective action.

**Proper definition:** the **kube-controller-manager** runs a collection of **controllers**, each responsible for one kind of resource, each independently running that same observe-compare-act loop.

For example: the **ReplicaSet controller** watches ReplicaSets, notices "desired: 3, actual: 2," and creates one more Pod. The **Node controller** watches node health, and if a node stops reporting in, begins taking action to reschedule that node's Pods elsewhere. We'll meet several more specific controllers as we go — but they're all the same underlying pattern, each focused on their own slice of cluster state.

### Cloud Controller Manager

**Simple explanation:** this is the component that lets Kubernetes talk to your specific cloud provider — creating a real AWS load balancer when you ask for one, for example — without the core Kubernetes codebase needing to know anything provider-specific.

**Proper definition:** the **cloud-controller-manager** runs controllers that integrate Kubernetes with cloud-provider-specific APIs — for example, provisioning an AWS Load Balancer when a Kubernetes `Service` of type `LoadBalancer` is created (a concept we'll build fully in Part VII).

**Why it's separated out like this:** it keeps Kubernetes's core logic cloud-agnostic — the same cluster software runs identically on AWS, GCP, on-prem, or your laptop, with only this one pluggable piece knowing anything about a specific cloud's actual APIs.

---

## Chapter 5.4 — The Worker Node, Component by Component

Every worker node runs three key components that let it actually execute the decisions the control plane makes.

### kubelet

**Simple explanation:** the kubelet is the control plane's agent on each individual node — it's the thing that actually makes containers run, by talking to the local container runtime, and it constantly reports back on what's really happening.

**Proper definition:** the **kubelet** is an agent running on every worker node that ensures the containers described in the Pods assigned to that node are actually running, and reports node and Pod status back to the control plane.

**How it connects back to Part II and Part III:** the kubelet doesn't run containers itself — it talks to a **CRI-compatible container runtime** (commonly containerd — recall Part II's distinction between Docker Engine and the actual runtime underneath it) to create and manage the actual containers, applying exactly the same namespaces-and-cgroups mechanics we already understand deeply from Part II. Nothing new is happening at the container level here — Kubernetes is orchestrating the exact same primitives you already know.

### kube-proxy

**Simple explanation:** kube-proxy is what makes Kubernetes networking actually work at the packet level — specifically, it's responsible for making sure traffic sent to a Service reaches one of the right Pods, even as those Pods come and go.

**Proper definition:** **kube-proxy** runs on every node and maintains network rules (via iptables or IPVS) that implement Kubernetes **Service** routing — load-balancing traffic across the Pods that back a given Service.

We're deliberately not going deep on Services yet — that's the centerpiece of Part VII — but it's worth knowing the name and role now, since it's a standard control-plane-vs-node architecture question.

### Container Runtime

Covered thoroughly in Part II and Part III: containerd (or another CRI-compatible runtime) actually creates and runs the containers, using runc underneath. Nothing new to add here — just note where it sits in the overall picture: it's the component the kubelet delegates to.

---

## Chapter 5.5 — What Actually Happens When You Run `kubectl create deployment`

This is the payoff for everything above — let's trace one command through the entire architecture, step by step, so none of this stays abstract.

```bash
kubectl create deployment shopsphere-backend --image=shopsphere-backend:v1
```

```text
 1. kubectl sends an authenticated HTTPS request
    describing a new Deployment object
                    |
                    v
 2. API Server validates the request, authenticates
    and authorizes it (we'll cover RBAC properly
    in Part VIII), and writes the new Deployment
    object into etcd
                    |
                    v
 3. The Deployment controller (inside the
    Controller Manager) notices the new Deployment
    via a watch on the API Server, and creates a
    ReplicaSet object to match the desired replica count
                    |
                    v
 4. The ReplicaSet controller notices the new
    ReplicaSet has 0 actual Pods but wants 1 (or more),
    and creates the Pod object(s) — written to etcd
    via the API Server, same as every step here
                    |
                    v
 5. The Scheduler notices a new Pod with no node
    assigned yet, evaluates available worker nodes,
    and assigns this Pod to one of them
                    |
                    v
 6. The kubelet on that chosen node notices
    (via the API Server) that a Pod has been
    assigned to it
                    |
                    v
 7. The kubelet instructs the local container
    runtime (containerd) to pull the image
    (from the registry — ECR, in ShopSphere's case)
    and start the container, using the exact
    namespaces/cgroups mechanics from Part II
                    |
                    v
 8. The kubelet continuously reports the Pod's
    real status back to the API Server (and
    therefore etcd) — Running, Ready, etc.
```

Notice something important: **every single step communicates only through the API Server.** The Scheduler doesn't talk to the kubelet directly. The kubelet doesn't talk to the Deployment controller directly. Everything reads and writes through that one consistent front door, exactly as Chapter 5.3 described — this is precisely why the API Server is described as the cluster's central nervous system.

Also notice: this entire flow is just the reconciliation loop from Chapter 5.1, playing out concretely, layer by layer — Deployment wants a ReplicaSet to exist → ReplicaSet wants Pods to exist → a Pod wants to be scheduled → a scheduled Pod wants a running container. Each layer is its own small "desired state vs actual state" loop, chained together.

**Interview question (advanced — this is a genuinely common one, and now you can actually answer it properly):** "Walk me through what happens internally when you create a Kubernetes Deployment." — Walk through the exact 8 steps above, in your own words, emphasizing that every component communicates exclusively through the API Server, and that the whole flow is fundamentally a chain of reconciliation loops rather than one monolithic "create" operation.

---

## Chapter 5.6 — Checkpoint

**Beginner:**
1. What is the difference between the control plane and a worker node?
2. What is etcd, and why is it described as the cluster's "source of truth"?

**Intermediate:**
3. What does it mean for Kubernetes to be "declarative," and how does that relate to self-healing?
4. What is the Scheduler actually responsible for — and just as importantly, what is it *not* responsible for?

**Advanced:**
5. Why does every control plane component communicate only through the API Server, rather than directly with each other?
6. Trace, in your own words, what happens between running `kubectl create deployment` and a container actually starting on a node.

**Scenario:**
7. A Pod is stuck with no node assigned at all, indefinitely. Based on the architecture in this chapter alone (before we cover troubleshooting formally in Part IX), which component would you suspect first, and why?

---

### Hands-On Lab 5.1 — Stand up a local cluster and watch the architecture in action

**Objective:** Get a real (if small) Kubernetes cluster running locally, and directly observe the control plane / worker node split.

**Prerequisites:** `kubectl` installed, and a local cluster tool — **kind** (Kubernetes in Docker) or **minikube**. We'll use kind here, since it's lightweight and runs entirely as containers (tying back neatly to everything in Parts II–V).

**Steps:**

1. Install kind and create a cluster:
   ```bash
   kind create cluster --name shopsphere-lab
   ```

2. Look at the nodes — even a single-node kind cluster still shows the control-plane/worker distinction conceptually:
   ```bash
   kubectl get nodes
   kubectl describe node shopsphere-lab-control-plane
   ```

3. Look directly at the control plane components — they actually run as Pods themselves, in the `kube-system` namespace:
   ```bash
   kubectl get pods -n kube-system
   ```
   You should see `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, and `kube-proxy` — the exact components from Chapter 5.3 and 5.4, running as real, inspectable Pods.

4. Create your first Deployment and immediately watch it:
   ```bash
   kubectl create deployment shopsphere-backend --image=nginx
   kubectl get pods --watch
   ```
   (We're using `nginx` here rather than pushing ShopSphere's real image yet — that comes with full YAML manifests in Part VII.)

5. Describe the Pod and find the "Events" section at the bottom — this is a direct, human-readable log of several of the exact reconciliation steps from Chapter 5.5:
   ```bash
   kubectl describe pod -l app=shopsphere-backend
   ```

**Expected result:** `kube-system` shows the named control plane components as running Pods; the new Deployment's Pod transitions through states (Pending → ContainerCreating → Running) that you can watch live; the Events section shows scheduling and image-pull events in order.

**Verification:** the Events output directly names the Scheduler assigning the Pod to a node, and the kubelet pulling and starting the container — you're seeing Chapter 5.5's trace happen for real, not just reading about it.

**Troubleshooting:** if `kind create cluster` fails, confirm Docker itself is running — kind's "nodes" are actually just specially configured Docker containers, another satisfying callback to everything we covered in Parts II and III.

**Cleanup:**
```bash
kind delete cluster --name shopsphere-lab
```

**Cost Warning:** none — kind runs entirely locally using Docker; nothing here touches AWS.

**Challenge:** run `kubectl get pods -n kube-system -o wide` and match each Pod's name against the component list in Chapters 5.3 and 5.4 — for each one, write a single sentence in your own words describing its job, without looking back at this chapter.

---

*End of Part VI. Part VII covers Kubernetes's core objects properly — Pod, ReplicaSet, Deployment, Service, Namespace, ConfigMap, Secret — each with full YAML, the "what/why/how/troubleshoot" treatment promised at the start of this book, and the moment ShopSphere's real backend image finally runs on Kubernetes instead of `nginx`.*
