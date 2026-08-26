# The ShopSphere DevOps Book
## Part X — Scaling, Release Strategies, and Kubernetes Security

---

### Where we left off

ShopSphere's backend is reliable under normal conditions — properly resourced, properly probed, properly spread across nodes. Two things remain before this is genuinely production-grade: it needs to react to real load automatically, deploy new versions without risk, and be secured against both external attackers and accidental internal overreach.

---

## Chapter 9.1 — Scaling

### Manual scaling, as a baseline

```bash
kubectl scale deployment shopsphere-backend -n shopsphere --replicas=6
```

This directly updates the Deployment's desired replica count — the exact reconciliation loop from Chapter 5.1 then does the rest. Fine for a known, planned event; useless for reacting to unpredictable real-world traffic in real time.

### HPA — Horizontal Pod Autoscaler

**Simple explanation:** the HPA watches a metric — usually CPU or memory usage — across a Deployment's Pods, and automatically adjusts the replica count up or down to keep that metric near a target you define.

**Proper definition:** the **Horizontal Pod Autoscaler (HPA)** is a controller (following the same reconciliation-loop pattern from Chapter 5.1 — observe actual metric, compare to target, act) that automatically scales the number of Pod replicas in a Deployment or ReplicaSet based on observed resource utilization or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shopsphere-backend-hpa
  namespace: shopsphere
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shopsphere-backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This says: keep average CPU utilization across all backend Pods near 70% of their configured *requests* (Chapter 8.1) — scaling up toward 10 replicas if load increases, back down toward 3 if it drops, never going outside that `min`/`max` range.

**A dependency worth naming explicitly: the Metrics Server.** The HPA doesn't measure CPU/memory usage itself — it reads that data from the **Metrics Server**, a lightweight cluster add-on that collects real-time resource usage from every kubelet and exposes it through the Kubernetes API. Without the Metrics Server installed, an HPA referencing CPU/memory utilization simply has no data to act on, and won't scale at all — a genuinely common, easy-to-miss real-world gotcha.

**Interview question (intermediate):** "You've created an HPA targeting CPU utilization, but it never scales up, even under heavy load. What's the first thing you'd check?" — Whether the Metrics Server is installed and functioning in the cluster (`kubectl top pods` is a quick way to confirm — if that command fails, so will the HPA) — the HPA has no usage data to act on without it.

### Cluster Autoscaler — the other half of the scaling picture

**Simple explanation:** the HPA scales the number of *Pods*. But if there's no room left on any existing node to actually place those new Pods, more Pods alone doesn't help — something also needs to add more *nodes*.

**Proper definition:** the **Cluster Autoscaler** automatically adjusts the number of nodes in the cluster — adding nodes when Pods are unschedulable due to insufficient capacity, and removing underutilized nodes to reduce cost.

**Scaling Pods vs. scaling Nodes — the distinction the book's outline specifically asked to make explicit:**

```text
        HPA                                Cluster Autoscaler
   ("do we have enough                ("do we have enough
    Pod replicas for                    node CAPACITY to
    current load?")                     actually SCHEDULE them?")
        |                                       |
        v                                       v
   more/fewer Pods                      more/fewer NODES
        |                                       |
        +-------------------+-------------------+
                             |
                             v
              Together: the app scales end-to-end,
           from "not enough replicas" all the way down
              to "not enough physical machines"
```

If only the HPA is configured and the cluster's nodes are already full, new Pods the HPA creates simply sit **Pending** — a real, common production symptom, and a genuine gap between "I configured autoscaling" and "my application actually scales." Both pieces are needed together for true elastic scaling. On EKS specifically (Part XI), this is implemented via **managed node groups** with autoscaling enabled, or the newer **Karpenter** autoscaler — we'll set this up directly in Part XI.

### VPA — Vertical Pod Autoscaler

Where HPA changes the *number* of replicas, **VPA (Vertical Pod Autoscaler)** instead adjusts a container's resource *requests/limits* themselves, based on observed usage over time — useful for right-sizing a workload whose per-replica needs are hard to estimate upfront. Worth knowing the name and the distinction from HPA; less commonly the first tool reached for compared to HPA, and generally not combined with HPA on the same CPU/memory metric without care, since the two can otherwise interact in confusing ways.

### KEDA — an advanced topic, briefly

**KEDA (Kubernetes Event-Driven Autoscaling)** extends autoscaling beyond CPU/memory to scale based on external event sources — queue depth, message backlog, custom application metrics. This is genuinely relevant to ShopSphere's **worker** service specifically: scaling the worker based on CPU alone doesn't capture the thing that actually matters for it — how many unprocessed orders are sitting in the queue. KEDA would let the worker scale directly on that number instead. We flag this as an advanced, worth-knowing-exists topic rather than building it out fully here.

---

## Chapter 9.2 — Deployment and Release Strategies

### Rolling deployments, precisely

We saw rolling updates happen automatically back in Chapter 6.4 — now let's make the mechanics explicit and controllable.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

- **`maxSurge`** — how many *extra* Pods beyond the desired count are allowed to exist temporarily during a rollout (creating new-version Pods slightly ahead of removing old ones).
- **`maxUnavailable`** — how many Pods are allowed to be unavailable at once during the rollout.

Setting `maxUnavailable: 0` guarantees full capacity is maintained throughout the entire rollout — new Pods must become Ready (recall readiness probes, Chapter 8.2 — this is exactly where they matter for a safe rollout) before any old Pod is removed. This is a direct, precise solution to Chapter 5.1's Problem 3.

### Rollbacks and revision history

```bash
kubectl rollout history deployment/shopsphere-backend -n shopsphere
kubectl rollout undo deployment/shopsphere-backend -n shopsphere
kubectl rollout undo deployment/shopsphere-backend -n shopsphere --to-revision=3
```

Every rollout creates a new ReplicaSet (Chapter 6.4) while retaining the previous ones (up to a configurable history limit) — a rollback is really just Kubernetes scaling a previous ReplicaSet back up and the current one back down, using the exact same rolling mechanism, in reverse.

**Interview question (beginner):** "How does a Kubernetes rollback actually work, mechanically?" — A Deployment retains previous ReplicaSets from earlier rollouts; a rollback scales the target previous ReplicaSet back up and the current one down, using the same rolling update mechanism as a forward deployment — it's not a special separate operation, just the same primitive applied in reverse.

### Blue/Green deployments

**Simple explanation:** run the entire new version alongside the entire old version, fully separately, then switch all traffic over in one deliberate moment — rather than gradually mixing old and new like a rolling update does.

**Proper definition:** a **blue/green deployment** maintains two complete, independent environments ("blue" = current live version, "green" = new version); traffic is cut over from blue to green all at once (typically by updating a Service's selector, or an Ingress/load balancer target) once green is fully verified.

**Why a team would choose this over a rolling update:** it avoids ever running old and new code simultaneously in front of live traffic (which a rolling update, by design, does temporarily) — valuable when mixed-version behavior would be genuinely problematic (e.g., an incompatible database schema change). The tradeoff: it requires double the resource capacity during the transition, and a real cutover mechanism.

### Canary deployments

**Simple explanation:** release the new version to a small slice of real traffic first — say, 5% — watch it closely, and only then gradually increase that slice, rather than switching everyone over at once.

**Proper definition:** a **canary deployment** routes a small, controlled percentage of production traffic to a new version alongside the stable version, incrementally increasing that percentage as confidence grows, with the ability to abort and roll back quickly if problems appear.

**Why this matters, connecting to Ingress and Service traffic-splitting:** a basic canary can be approximated with two Deployments behind one Service (weighted by replica count — 1 canary replica out of 20 total gives roughly 5% of traffic), but genuinely precise, controllable traffic-percentage splitting typically requires a **service mesh** (like Istio or Linkerd) or a progressive-delivery tool built for exactly this (like **Argo Rollouts** or **Flagger**) — worth knowing these names exist as the real-world tools for this, even though we won't build a full service mesh in this book.

### Progressive delivery, as an umbrella term

**Progressive delivery** is the general name for this whole family of techniques — canary, blue/green, feature flags — unified by one idea: **reduce blast radius by controlling exposure gradually**, rather than exposing 100% of users to a change all at once and hoping for the best.

| Strategy | Old and new run simultaneously? | Blast radius if broken | Resource cost during rollout |
|---|---|---|---|
| Rolling update | Briefly, mixed | Partial, self-limiting | Minimal (default) |
| Blue/Green | Fully, side by side, then instant cutover | Full, but instantly reversible | Double, temporarily |
| Canary | Fully, weighted | Small, deliberately limited | Small extra overhead |

**Interview question (advanced):** "When would you choose a canary deployment over a standard rolling update?" — When the risk of a bad release is high enough to justify limiting exposure to a small percentage of real traffic first, with the ability to observe real production behavior and abort before a broad rollout — particularly valuable for major changes, or when the team's confidence in pre-production testing alone isn't sufficient.

---

## Chapter 9.3 — Kubernetes Security

### Authentication and authorization, distinguished precisely

**Authentication** answers "who are you?" — establishing identity (a user, or a Pod's ServiceAccount). **Authorization** answers "what are you allowed to do?" — given a confirmed identity. Kubernetes handles authentication via several possible methods (client certificates, tokens, integration with external identity providers), and handles authorization primarily through **RBAC**.

### RBAC — Role-Based Access Control

**Simple explanation:** RBAC lets you precisely define what actions a specific person or service is allowed to take, on which kinds of objects, in which namespaces — instead of everyone having full access to everything.

**Proper definition:** **RBAC (Role-Based Access Control)** grants permissions by binding **subjects** (users, groups, or ServiceAccounts) to **roles**, which define allowed actions (verbs like `get`, `list`, `create`, `delete`) on specific resource types.

**Role and RoleBinding — namespace-scoped:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: shopsphere
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-jane
  namespace: shopsphere
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

This grants the user `jane` read-only access to Pods, but only within the `shopsphere` namespace — she can't see or touch anything in `shopsphere-staging`, unless a separate binding grants that too.

**ClusterRole and ClusterRoleBinding — cluster-wide:** structurally identical, but not scoped to a single namespace — used for genuinely cluster-wide permissions (managing Nodes, for example, which aren't namespaced objects at all) or for granting the same role across every namespace at once.

### ServiceAccounts

**What it is.** A **ServiceAccount** is an identity for a *process running inside the cluster* (a Pod) to authenticate to the Kubernetes API — as distinct from a *human* user identity.

**Why it exists.** ShopSphere's backend might legitimately need to call the Kubernetes API itself — for example, to look up configuration, or (in more advanced setups) to manage other objects programmatically. It shouldn't do this with a human's personal credentials; it needs its own scoped, auditable identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: shopsphere-backend-sa
  namespace: shopsphere
```

```yaml
spec:
  template:
    spec:
      serviceAccountName: shopsphere-backend-sa
      containers:
        - name: backend
          # ...
```

Every Pod actually uses a ServiceAccount by default (the `default` ServiceAccount, if none is specified) — the security-relevant question is almost never "does this Pod have a ServiceAccount," but rather "does its ServiceAccount have far more permission than it actually needs." **Least privilege** — give each ServiceAccount (and role, and human user) the absolute minimum permissions required, and nothing more — is the guiding principle across all of this, and directly ties into the **IRSA / Pod identity** concept we'll cover properly in Part XI, which extends this exact same idea to let a Pod's ServiceAccount also carry a scoped *AWS* identity.

### SecurityContext

**What it is.** **SecurityContext** is where Pod- and container-level security settings actually live in a manifest — the formal Kubernetes home for several concepts you already learned conceptually back in Part V's Docker security chapter.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  containers:
    - name: backend
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

Notice how directly this maps onto Part V: `runAsNonRoot`/`runAsUser` is the Kubernetes-level enforcement of the non-root `USER` instruction from your Dockerfile; `readOnlyRootFilesystem` is the same idea as Docker's `--read-only` flag; `capabilities: drop: ["ALL"]` is exactly the `--cap-drop=ALL` pattern from Chapter 4.1, expressed as native Kubernetes configuration instead of a `docker run` flag. **This is the same set of concerns, applied at the layer that actually schedules and runs your containers in production** — nothing conceptually new, just the production-grade home for ideas you already understand.

### Pod Security Standards

**What it is.** **Pod Security Standards** define three tiers of increasingly strict security policy — **Privileged**, **Baseline**, and **Restricted** — that can be enforced at the namespace level, replacing an older, now-deprecated mechanism called PodSecurityPolicy.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopsphere
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**Restricted** enforces the strongest tier — non-root required, no privilege escalation, dropped capabilities, and more — as a genuine, cluster-enforced *policy*, rather than a convention individual teams have to remember to follow correctly on their own.

### NetworkPolicy

**What it is.** A **NetworkPolicy** controls which Pods are allowed to communicate with which other Pods — the direct Kubernetes-level implementation of the exact firewall/allow-list concept introduced back in Part I, and the container-network-segmentation idea from Part IV's Chapter 3.1, now formalized at the cluster level.

**Why it matters, and an important default worth stating clearly:** by default, Kubernetes Pods can reach *any* other Pod in the cluster, across namespace boundaries, with no restriction at all — this is exactly the gap flagged back in Chapter 6.6's checkpoint question about namespaces not being a real network isolation boundary on their own. NetworkPolicy is the tool that closes that gap.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db-only
  namespace: shopsphere
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: shopsphere-backend
      ports:
        - protocol: TCP
          port: 5432
```

This says: only Pods labeled `app: shopsphere-backend` may initiate a connection to Pods labeled `app: postgres`, on port 5432 — everything else is denied by default, once *any* NetworkPolicy selects a given Pod. This is precisely the defense-in-depth layer that answers Part I's checkpoint question directly: even if an attacker compromises the frontend Pod, a correctly configured NetworkPolicy prevents it from ever reaching the database directly, because the frontend isn't in the allowed `podSelector` at all.

**A crucial, commonly-tested caveat:** NetworkPolicy support isn't automatic — it depends entirely on the cluster's **CNI plugin** (mentioned back in Chapter 7.1) actually implementing NetworkPolicy enforcement. Some CNI plugins don't enforce it at all, silently accepting the objects without actually restricting any traffic — a genuinely dangerous, easy-to-miss gap, worth explicitly verifying on any cluster you're securing.

### Admission control, briefly

**Admission controllers** intercept requests to the API Server *after* authentication/authorization but *before* an object is persisted to etcd, and can validate or even mutate the request. Pod Security Standards enforcement above is itself implemented as an admission controller. More advanced setups use tools like **OPA Gatekeeper** or **Kyverno** to enforce custom organizational policy at this exact layer (e.g., "reject any Deployment that doesn't specify resource limits") — worth knowing this concept and these names exist, as the natural next step beyond what this book builds directly.

**Interview question (advanced, tying the whole chapter together):** "A frontend Pod gets compromised. Walk through the layers of defense that should limit the damage." — Strong answer, referencing multiple layers built across this book: the container itself runs as non-root with dropped capabilities and a read-only filesystem (SecurityContext, Part V); its ServiceAccount has only the minimal RBAC permissions it actually needs (least privilege); NetworkPolicy prevents it from reaching anything beyond what it's explicitly allowed to talk to (specifically, it should have no path to the database Pods at all); and resource limits (Chapter 8.1) prevent it from being used to exhaust the node's capacity even if it starts misbehaving. No single layer is treated as sufficient on its own — that's the actual meaning of defense-in-depth.

---

## Chapter 9.4 — Checkpoint

**Beginner:**
1. What's the difference between authentication and authorization?
2. What does a NetworkPolicy control?

**Intermediate:**
3. Why do you need both the HPA and the Cluster Autoscaler for an application to truly scale under load?
4. What's the difference between a rolling update and a canary deployment, in terms of blast radius if something goes wrong?

**Advanced:**
5. Explain how SecurityContext settings map directly onto Docker security concepts from Part V.
6. Why is it dangerous to assume NetworkPolicy is automatically enforced on any Kubernetes cluster?

**Scenario:**
7. New Pods created by an HPA scale-up event are stuck in `Pending` indefinitely, even though the HPA shows the desired replica count increased correctly. Diagnose the likely cause, using this chapter alone.

---

### Hands-On Lab 9.1 — Autoscale, roll out safely, and lock down network access

**Objective:** Configure HPA on the ShopSphere backend, perform a zero-downtime rolling update, and restrict database access with NetworkPolicy.

**Prerequisites:** The `shopsphere` namespace resources from Part VII/VIII's labs; Metrics Server installed in your kind cluster (`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` — for kind specifically, you'll typically also need to add `--kubelet-insecure-tls` to its args, since kind's kubelet certificates aren't set up for the Metrics Server's default trust expectations).

**Steps:**

1. Apply the HPA from Chapter 9.1, confirm it can see metrics:
   ```bash
   kubectl apply -f hpa.yaml
   kubectl top pods -n shopsphere
   kubectl get hpa -n shopsphere
   ```

2. Trigger a rolling update with a safe strategy, and watch it stay fully available throughout:
   ```bash
   kubectl set image deployment/shopsphere-backend backend=<image>:v2 -n shopsphere
   kubectl rollout status deployment/shopsphere-backend -n shopsphere
   ```
   In a second terminal, loop `curl` against the Service the whole time and confirm zero failed requests.

3. Apply the NetworkPolicy from Chapter 9.3, then prove it's actually working — from a Pod that is *not* the backend, confirm the database is unreachable:
   ```bash
   kubectl run -n shopsphere attacker-sim --rm -it --image=busybox --restart=Never -- \
     nc -zv postgres 5432
   ```
   This should fail/time out — then confirm the legitimate backend Pod *can* still reach it, proving the policy is correctly scoped, not just broadly blocking everything.

**Expected result:** HPA shows current CPU usage and a valid target; the rolling update completes with zero dropped requests in your `curl` loop; the non-backend Pod cannot reach Postgres, while the backend Pod still can.

**Verification:** the asymmetric NetworkPolicy result (one Pod blocked, the other allowed) is the real proof the policy is correctly scoped rather than accidentally too broad or doing nothing at all.

**Troubleshooting:** if `kubectl top pods` returns an error, Metrics Server isn't correctly installed/trusted yet — the HPA will not function until this is resolved, exactly as flagged in Chapter 9.1.

**Cleanup:**
```bash
kubectl delete namespace shopsphere
```

**Challenge:** create a Role and RoleBinding granting a hypothetical `shopsphere-oncall` user read-only access (`get`, `list`, `watch` on `pods` and `pods/log`) within the `shopsphere` namespace only, and verify with `kubectl auth can-i get pods --as=shopsphere-oncall -n shopsphere` (should say yes) versus `kubectl auth can-i delete pods --as=shopsphere-oncall -n shopsphere` (should say no).

---

*End of Part X. Part XI is systematic Kubernetes troubleshooting — CrashLoopBackOff, ImagePullBackOff, Pending Pods, and more, taught through intentionally broken versions of ShopSphere and a repeatable diagnostic methodology, followed by Helm, which converts everything we've built by hand across Parts VII–X into a single, reusable, versioned chart.*
