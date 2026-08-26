# The ShopSphere DevOps Book
## Part IX — Resource Management, Health Probes, and Scheduling

---

### Where we left off

ShopSphere's backend is deployed, reachable internally by Service, and reachable externally through Ingress. It works — under calm conditions. This Part is about what happens when conditions aren't calm: traffic spikes, a bad code path leaks memory, a dependency hangs. This is where a "working" Deployment becomes a genuinely *reliable* one.

---

## Chapter 8.1 — Resource Requests and Limits

### Why this matters immediately

Recall cgroups from Part II — the kernel mechanism that limits how much CPU and memory a process can actually use. Kubernetes resource requests and limits are, precisely, cgroups configuration applied through the kubelet to each container. Nothing new mechanically — just a new, cluster-aware layer of *decision-making* about what values to set and why.

### Requests: what a container is guaranteed, and what the Scheduler uses

**Simple explanation:** a **request** is what you tell Kubernetes a container needs to run properly — and it's also the number the Scheduler uses to decide which node has room for it.

**Proper definition:** a **resource request** specifies the minimum amount of CPU and/or memory a container is guaranteed to receive, and is the value the Scheduler (Chapter 5.3) uses when deciding whether a node has sufficient available capacity to host a given Pod.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
```

`250m` means 250 **millicpu** — a quarter of one CPU core. `256Mi` means 256 mebibytes.

**Why this connects directly back to Chapter 5.3:** when the Scheduler filters candidate nodes, it's summing up the requests of everything already running on each node, and checking whether the new Pod's requests still fit within that node's actual capacity. Set requests too low, across many Pods, and you can end up with a node that's technically "not full" by the Scheduler's accounting, but genuinely starved in practice once real traffic hits — a real, common production issue known as overcommitment.

### Limits: the hard ceiling

**Simple explanation:** a **limit** is the absolute maximum a container is ever allowed to use — the kernel enforces this directly, exactly like the `--memory`/`--cpus` flags from Part II's Docker lab.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**What happens when a container exceeds its CPU limit:** it gets **throttled** — the kernel restricts its CPU time, slowing it down, but the process keeps running.

**What happens when a container exceeds its memory limit — OOMKilled.** Memory behaves completely differently from CPU here, and this distinction is a very common, very real interview question: memory can't be "throttled" the way CPU can — you can't partially deny a process memory it's already trying to use. So when a container tries to exceed its memory limit, the kernel's **OOM (Out Of Memory) killer** terminates the container's process outright. Kubernetes then reports this as **OOMKilled** in `kubectl describe pod`, and — because this is a Deployment, and the ReplicaSet controller from Chapter 6.3 is watching — a replacement Pod gets created automatically. From the outside, this can look like "the Pod keeps randomly restarting," when the real, precise cause is "this container's actual memory usage exceeds its configured limit."

**Interview question (intermediate, and genuinely one of the most common Kubernetes interview questions there is):** "What's the difference between what happens when a container exceeds its CPU limit versus its memory limit?" — Exceeding a CPU limit results in throttling — the container is slowed down but keeps running; exceeding a memory limit results in the container being OOMKilled by the kernel, because memory usage can't be throttled the way CPU time can — the process is simply terminated, and Kubernetes then restarts it per the Deployment's desired state, which can look like a crash loop from the outside if it keeps happening repeatedly.

### QoS classes

Kubernetes automatically assigns every Pod one of three **Quality of Service (QoS) classes**, based purely on how its requests and limits are configured — and this classification directly affects eviction priority under node pressure.

- **Guaranteed** — every container in the Pod has requests *equal to* limits, for both CPU and memory. Highest priority; evicted last under node resource pressure.
- **Burstable** — at least one container has requests set, but they don't exactly equal limits (the common, sensible middle ground — our ShopSphere example above is Burstable).
- **BestEffort** — no requests or limits set at all, for any container. Lowest priority; evicted first under node resource pressure.

**Why this exists:** when a node genuinely runs low on memory, Kubernetes needs a principled way to decide which Pods to evict first, to relieve pressure — and QoS class is the primary factor in that decision, precisely because it reflects how much of a "promise" was made to each Pod in the first place.

**Interview question (advanced):** "You want a critical, latency-sensitive workload to be the very last thing evicted under node memory pressure. How do you configure it?" — Set its `requests` exactly equal to its `limits` for both CPU and memory, giving it the Guaranteed QoS class — the highest eviction priority tier.

### ResourceQuota and LimitRange

Individual Pod-level requests/limits are only half the picture — at the **Namespace** level (Chapter 6.6), two more objects govern resource usage collectively:

- **ResourceQuota** — caps the *total* resource consumption across an entire namespace (e.g., "the `shopsphere-staging` namespace may request at most 4 CPU cores and 8Gi of memory, combined, across every Pod in it").
- **LimitRange** — sets default requests/limits for containers that don't specify their own, and can enforce minimum/maximum bounds on what any single container is allowed to request.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: shopsphere-staging-quota
  namespace: shopsphere-staging
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

**Real-world use:** these are precisely the tools a platform team uses to prevent one team's staging namespace from silently consuming an entire shared cluster's capacity — a genuinely common multi-tenant cluster governance concern.

---

## Chapter 8.2 — Health Probes

### Why probes exist, connecting directly back to Part IV

Recall the `depends_on`/healthcheck problem from Docker Compose in Chapter 3.3: knowing a container has *started* is not the same as knowing it's actually *ready to do useful work*. Kubernetes formalizes this exact same idea — with more precision, and three distinct kinds of checks, each answering a different question.

### Liveness probes

**Simple explanation:** a liveness probe answers "is this container still alive and functioning, or has it gotten stuck in a broken state it can't recover from on its own?"

**Proper definition:** a **liveness probe** periodically checks whether a container is still healthy; if it fails repeatedly (beyond a configured threshold), the **kubelet restarts that container**.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

**When to use it:** for genuine deadlock/stuck-state detection — a process that's technically still running (so a simple "is the process alive" check would pass) but has, for example, deadlocked on an internal lock and will never recover without a restart.

**A genuinely important, commonly-tested caution:** a poorly-designed liveness probe is actively dangerous. If your `/health` endpoint checks the database connection, and the database has a brief hiccup, every backend Pod's liveness probe fails simultaneously, and Kubernetes restarts *all of them at once* — turning a transient database blip into a full application outage, entirely self-inflicted by an overly aggressive liveness check. The correct scope for a liveness check is narrow: "is *this process itself* stuck," not "are all of my dependencies currently healthy" — that broader question is exactly what readiness probes are for, next.

### Readiness probes

**Simple explanation:** a readiness probe answers "is this container currently able to correctly handle traffic right now?" — and unlike a liveness failure, a readiness failure doesn't restart anything; it just temporarily stops sending traffic there.

**Proper definition:** a **readiness probe** checks whether a container is ready to receive traffic; if it fails, the Pod is marked **NotReady** and is **removed from the Service's endpoints** (recall `kubectl get endpoints` from Chapter 6.5) until it passes again — the container itself is left running, untouched.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**This is precisely the right place for the "are my dependencies healthy" check** that we just said doesn't belong in a liveness probe: if the backend genuinely can't reach the database right now, its readiness probe should fail — correctly removing it from the Service's rotation so it stops receiving requests it can't actually fulfill — while its liveness probe stays passing, because the process itself isn't stuck, and doesn't need to be restarted over a transient, external dependency issue.

**Interview question (advanced, and this exact distinction is asked constantly):** "What's the difference between a liveness probe and a readiness probe, and why shouldn't they check the same thing?" — A failed liveness probe causes a container **restart**; a failed readiness probe causes the Pod to be temporarily **removed from Service traffic**, with no restart. They should generally check different things: liveness should narrowly detect the process itself being stuck (deadlock, unrecoverable internal state), while readiness should reflect whether the container is currently able to serve traffic correctly, which can legitimately include upstream dependency health. Conflating the two — especially making liveness depend on external dependencies — risks cascading, self-inflicted outages when a shared dependency has a brief issue.

### Startup probes

**Simple explanation:** a startup probe gives a slow-starting application room to finish booting, without the liveness probe getting impatient and restarting it mid-startup.

**Proper definition:** a **startup probe** runs first, before liveness and readiness probes begin at all; while it hasn't yet succeeded, liveness and readiness checks are disabled entirely, preventing a slow-starting container from being killed for "failing" a liveness check it simply hasn't had time to pass yet.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8000
  failureThreshold: 30
  periodSeconds: 2
```

**When you actually need this, specifically:** an application with a genuinely slow, variable startup time — loading a large in-memory cache, running database migrations on boot — where a liveness probe tuned for normal steady-state behavior would otherwise trigger a restart before the app ever got the chance to finish starting up in the first place.

**Interview question (intermediate):** "Why would you add a startup probe to a Pod that already has a liveness probe configured?" — To prevent the liveness probe from prematurely killing a container that simply needs more time to finish starting than the liveness probe's normal steady-state timing allows — the startup probe holds off liveness (and readiness) checks until the application has genuinely finished booting.

### Probe types beyond HTTP

Besides `httpGet`, probes can also use `tcpSocket` (just check that a port accepts a connection — useful for non-HTTP services) or `exec` (run an arbitrary command inside the container and check its exit code — useful when there's no convenient network endpoint to check at all).

---

## Chapter 8.3 — Scheduling

### Beyond "does it fit" — expressing real placement requirements

Chapter 5.3 introduced the Scheduler's basic job: find a node with enough available capacity. Real production clusters often need more nuanced placement rules than that alone.

### Node selectors — the simple case

**Simple explanation:** the simplest way to say "only run this Pod on nodes with this specific label."

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

This requires an exact label match on candidate nodes — simple, but inflexible; there's no concept of "prefer this, but it's fine if not."

### Node affinity — the more expressive version

**Node affinity** does the same fundamental job as `nodeSelector`, but with meaningfully more expressive rules, and — critically — a **soft/preferred** option alongside the hard/required one.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
              operator: In
              values: ["ssd"]
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
            - key: zone
              operator: In
              values: ["us-east-1a"]
```

`requiredDuringScheduling...` is a hard constraint — the Pod simply won't be scheduled anywhere that doesn't satisfy it. `preferredDuringScheduling...` is a soft preference — the Scheduler tries to honor it, weighted against other preferences, but will still schedule the Pod elsewhere rather than leave it unscheduled entirely.

### Pod affinity and anti-affinity

**Pod affinity** expresses a placement preference *relative to other Pods*, rather than relative to node labels directly — "schedule this Pod near Pods with this label." **Pod anti-affinity** does the opposite — "schedule this Pod away from Pods with this label."

**Why anti-affinity matters enormously for reliability, and directly connects back to Chapter 5.1's Problem 1 (single point of failure):** without anti-affinity, the Scheduler could easily place all three of ShopSphere's backend replicas on the *same physical node* — meaning a single node failure takes down every replica simultaneously, completely defeating the purpose of running 3 replicas in the first place. Pod anti-affinity is exactly the tool that prevents this:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: shopsphere-backend
          topologyKey: kubernetes.io/hostname
```

`topologyKey: kubernetes.io/hostname` means "spread these across different *nodes*" specifically (you could also target `topology.kubernetes.io/zone` to spread across different AWS Availability Zones — genuinely important for real high availability, and something we'll return to directly in the EKS chapter, Part XI).

**Interview question (advanced, and a genuinely important production concept):** "You have 3 replicas of a critical service for high availability, but a single node failure took down all 3 at once. What Kubernetes feature would have prevented this, and how?" — Pod anti-affinity, configured with `topologyKey: kubernetes.io/hostname` (or `topology.kubernetes.io/zone` for AZ-level resilience), instructing the Scheduler to spread the replicas across different nodes (or zones) rather than allowing them to land on the same one — without it, having multiple replicas provides no real protection against a single node's failure.

### Taints and tolerations — the inverse mechanism

Node affinity is about a *Pod* expressing where it wants to go. **Taints and tolerations** work in the opposite direction: a *node* repels Pods by default, unless a Pod specifically declares it can tolerate that repulsion.

```bash
kubectl taint nodes gpu-node-1 workload=gpu:NoSchedule
```

This taints `gpu-node-1` so that, by default, *no* Pod will be scheduled there. Only a Pod with a matching **toleration** can be scheduled onto it:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

**Why this exists, and the precise distinction from affinity worth having ready for an interview:** affinity says "I, the Pod, prefer/require this kind of node." Tolerations say "I, the Pod, am specifically permitted onto this otherwise-repelling node." Taints/tolerations are commonly used to reserve specialized nodes (GPU nodes, nodes with special licensing) for only the specific workloads that declare they belong there, keeping ordinary workloads from accidentally landing on expensive or specialized hardware they don't actually need.

**Interview question (intermediate):** "What's the conceptual difference between node affinity and a toleration?" — Node affinity is expressed by a Pod, pulling it *toward* nodes with certain characteristics; a taint is expressed by a node, pushing Pods *away* by default, and a toleration is what a specific Pod declares to be allowed past that repulsion. They're often used together: taint specialized nodes to keep ordinary workloads off them, and give only the Pods that specifically need those nodes both a toleration (permission to land there) and often an affinity rule (an active preference to actually go there).

---

## Chapter 8.4 — Checkpoint

**Beginner:**
1. What's the practical difference between a resource *request* and a resource *limit*?
2. What does "OOMKilled" mean, precisely?

**Intermediate:**
3. Why shouldn't a liveness probe check the health of an external dependency like a database?
4. What's the difference between what a failed liveness probe does versus what a failed readiness probe does?

**Advanced:**
5. Explain QoS classes, and how a Pod earns "Guaranteed" status.
6. Why is Pod anti-affinity necessary even when you're already running multiple replicas of a Deployment for high availability?

**Scenario:**
7. A team reports that during a brief, transient database outage, their entire backend fleet went down and stayed down for several minutes even after the database recovered. Using everything in Chapter 8.2, diagnose the likely misconfiguration.

---

### Hands-On Lab 8.1 — Break things on purpose

**Objective:** Directly observe OOMKilled, a bad liveness probe cascading failure, and a good readiness probe correctly draining traffic.

**Prerequisites:** A kind cluster; a simple test image that can simulate memory growth and toggleable health-endpoint failure (a small Python Flask app with `/burn-memory`, `/health`, and `/ready` endpoints works well — build one, or adapt an existing test image).

**Steps:**

1. Deploy with a deliberately low memory limit, then trigger the memory-growth endpoint:
   ```yaml
   resources:
     requests:
       memory: "32Mi"
     limits:
       memory: "64Mi"
   ```
   ```bash
   kubectl apply -f low-memory-deployment.yaml
   kubectl exec deploy/memtest -- curl -s localhost:8000/burn-memory
   kubectl get pods --watch
   ```

2. Confirm the cause directly:
   ```bash
   kubectl describe pod -l app=memtest | grep -A5 "Last State"
   ```
   You should see `OOMKilled` explicitly named as the reason.

3. Deploy a second app with an intentionally bad liveness probe pointed at a `/health` endpoint that also depends on a (simulated) flaky downstream, and toggle that downstream to fail — watch every replica restart simultaneously.

4. Now fix it: move that same dependency check to a `readinessProbe` instead, toggle the downstream to fail again, and confirm this time the Pods stay Running (no restart), just marked NotReady and pulled from Service endpoints:
   ```bash
   kubectl get endpoints backend-service --watch
   ```

**Expected result:** step 2 shows an explicit OOMKilled reason; step 3 shows a full, simultaneous, self-inflicted outage; step 4 shows graceful, correct traffic-draining behavior instead, with the container never restarting.

**Verification:** the side-by-side contrast between steps 3 and 4, using the exact same simulated failure, is the real lesson — same underlying dependency blip, dramatically different blast radius, purely due to which probe type it was wired to.

**Troubleshooting:** if Pods don't OOMKill as expected, confirm the memory limit is actually being applied (`kubectl describe pod` should show it under Limits) — a missing `limits` block with only `requests` set leaves the container unbounded.

**Cleanup:**
```bash
kubectl delete deployment memtest backend-badprobe
```

**Challenge:** add a startup probe to a container with an artificially long, simulated boot delay, and confirm — by comparing behavior with and without the startup probe — that it prevents a premature liveness-triggered restart during that boot window.

---

*End of Part IX. Part X covers scaling (HPA, Metrics Server, Cluster Autoscaler), deployment and release strategies (rolling updates, rollbacks, blue/green, canary), and Kubernetes security in depth (RBAC, SecurityContext, Pod Security Standards, NetworkPolicy) — the last major building blocks before we move into systematic troubleshooting and Helm.*
