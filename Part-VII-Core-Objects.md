# The ShopSphere DevOps Book
## Part VII — Kubernetes Core Objects

---

### Where we left off

You understand *why* Kubernetes exists and *how* it makes decisions internally. Now we actually put ShopSphere on it — properly, with real YAML, starting from the smallest building block and working up to a fully networked, configured application.

For every object in this chapter, we'll consistently answer: **what it is, why it exists, what problem it solves, how it relates to other objects, a YAML example, real-world use, troubleshooting, and interview questions** — exactly as promised at the start of this book.

---

## Chapter 6.1 — YAML Basics, Briefly

Every Kubernetes object you define follows the same basic shape:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopsphere-backend
spec:
  # ...object-specific configuration
```

- **`apiVersion`** — which version of the Kubernetes API this object belongs to (different object kinds live at different API versions — e.g. `v1` for core objects like Pods, `apps/v1` for Deployments).
- **`kind`** — what type of object this is (Pod, Deployment, Service, and so on).
- **`metadata`** — identifying information: `name`, `namespace`, `labels`, `annotations`.
- **`spec`** — the desired state you're declaring, in Chapter 5.1's sense exactly. This is the part that differs the most between object kinds.

Most objects also report a `status` field once created — but you don't write that yourself; Kubernetes fills it in to reflect actual observed state. That `spec` vs `status` split *is* the desired-state-vs-actual-state model from Part VI, made literal in the object's own structure.

---

## Chapter 6.2 — Pod

**What it is.** A **Pod** is the smallest deployable unit in Kubernetes — one or more containers that are always scheduled together, on the same node, sharing the same network namespace and, optionally, storage.

**Why it exists, and why "one or more" containers rather than always exactly one.** Most Pods run a single container — that's the common case, and it's fine to think of a Pod as "a container" most of the time. But sometimes two containers are so tightly coupled they genuinely need to share network and storage — a classic example is a **sidecar container** that ships logs from a shared volume alongside the main app container. Kubernetes doesn't orchestrate individual containers directly; it orchestrates Pods, so this tightly-coupled-group case is supported natively, without it complicating the common single-container case at all.

**What problem it solves.** It gives Kubernetes one consistent unit of scheduling and lifecycle management, whether an application needs one container or a small tightly-coupled group of them.

**Relationship to other objects.** You will almost never create a bare Pod directly in production — you'll create a Deployment (Chapter 6.4), which creates and manages Pods for you. We're covering Pods directly first purely so the layer above makes sense.

**YAML example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shopsphere-backend-manual
  labels:
    app: shopsphere-backend
spec:
  containers:
    - name: backend
      image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
      ports:
        - containerPort: 8000
```

**Real-world use.** Bare Pods show up mostly for one-off debugging tasks or short-lived test workloads — not for anything that needs to survive a failure or scale.

**Troubleshooting.** `kubectl get pods` shows the Pod's **phase**: `Pending` (not yet scheduled or still pulling images), `Running`, `Succeeded`, `Failed`, or `Unknown`. `kubectl describe pod <name>` shows the Events section — the single most useful first troubleshooting step for almost any Pod problem, as you saw directly in Part VI's lab.

**Interview question (beginner):** "Why doesn't Kubernetes schedule individual containers directly?" — Because some containers need to always run together, sharing network and storage — the Pod is the abstraction that groups them as one atomic scheduling unit, while still supporting the common single-container case cleanly.

---

## Chapter 6.3 — ReplicaSet

**What it is.** A **ReplicaSet** ensures a specified number of identical Pod replicas are running at all times.

**Why it exists.** This is Chapter 5.1's reconciliation loop, applied specifically to "how many copies of this Pod should exist." If a Pod crashes, the ReplicaSet controller notices actual (2) no longer matches desired (3), and creates a replacement.

**What problem it solves.** Without it, a crashed Pod just stays crashed — nothing is watching to replace it. This is the mechanism underneath ShopSphere's self-healing requirement from Chapter 5.1.

**Relationship to other objects.** In practice, you don't create ReplicaSets directly either — a Deployment creates and manages them for you, which is exactly what you saw happen automatically in Part VI's `kubectl create deployment` trace (step 3).

**YAML example** (shown for understanding — again, not something you'll write by hand in practice):

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: shopsphere-backend-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopsphere-backend
  template:
    metadata:
      labels:
        app: shopsphere-backend
    spec:
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
```

Notice the **selector** — this is how the ReplicaSet knows *which* Pods belong to it: any Pod carrying the label `app: shopsphere-backend`, whether the ReplicaSet created it or not. Labels-and-selectors is a pattern you'll see reused constantly across Kubernetes — Services (Chapter 6.5) use the exact same mechanism to find the Pods they should route traffic to.

**Real-world use.** You'll see ReplicaSets in `kubectl get replicasets` output, auto-named and auto-managed underneath a Deployment — useful to understand when reading `kubectl describe deployment` output, even though you're not managing them by hand.

**Troubleshooting.** If `kubectl get replicaset` shows `DESIRED: 3, CURRENT: 3, READY: 1`, that tells you Pods exist but aren't passing their readiness checks — a strong pointer toward checking probes (Part VIII) next, not toward the ReplicaSet itself.

**Interview question (intermediate):** "What's the difference between a ReplicaSet and a Deployment?" — A ReplicaSet's only job is maintaining a stable number of identical Pod replicas; a Deployment sits a layer above it, managing ReplicaSets to provide rolling updates, rollback history, and revision tracking — which is exactly what we cover next.

---

## Chapter 6.4 — Deployment

**What it is.** A **Deployment** manages ReplicaSets on your behalf, and adds the capability that makes it genuinely production-usable: controlled, gradual rollout of changes, with rollback history.

**Why it exists.** A ReplicaSet alone knows how to maintain a fixed set of *identical* Pods — but it has no concept of "replace the old version with a new version, gradually, safely." That's precisely the gap a Deployment fills.

**What problem it solves.** Recall Problem 3 from Chapter 5.1: "deploys are risky, with no clean rollback." A Deployment solves this directly, by creating a *new* ReplicaSet for the new version, scaling it up gradually while scaling the old ReplicaSet down, and keeping a revision history you can roll back to.

**Relationship to other objects.** Deployment → manages → ReplicaSet → manages → Pods. This three-layer chain is exactly what you traced, live, in Part VI's `kubectl create deployment` walkthrough.

**YAML example — this is the real object you'll actually write, constantly, from here on:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopsphere-backend
  labels:
    app: shopsphere-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopsphere-backend
  template:
    metadata:
      labels:
        app: shopsphere-backend
    spec:
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
          ports:
            - containerPort: 8000
          env:
            - name: PORT
              value: "8000"
```

Let's name the two sections precisely, because the distinction matters and is a common point of confusion: `spec.selector` and `spec.template.metadata.labels` must match — the selector says "these are the Pods I own," and the template's own labels must satisfy that selector, or the Deployment will fail to adopt the Pods it creates. `spec.template` is a full embedded Pod specification — everything under it is exactly the Pod-level configuration from Chapter 6.2, just nested one level deeper.

**Real-world use.** This is the object you'll deploy to production constantly — every application ShopSphere runs on Kubernetes will have a Deployment as its core object. We'll return to Deployments heavily in Part VIII for rolling updates and rollbacks specifically.

**Troubleshooting.** `kubectl rollout status deployment/shopsphere-backend` tells you if a rollout is stuck; `kubectl describe deployment shopsphere-backend` shows recent ReplicaSet events; `kubectl get replicasets` shows old and new ReplicaSets side by side during a rollout, letting you see the gradual handoff directly.

**Interview question (advanced):** "You update a Deployment's image, and `kubectl get pods` shows a mix of old and new Pods for a while. Is that a bug?" — No — that's the Deployment's rolling update strategy working as intended: it gradually creates new-version Pods and terminates old-version ones according to configurable `maxSurge`/`maxUnavailable` settings, rather than replacing everything at once, to avoid a moment where the application has zero healthy capacity.

---

## Chapter 6.5 — Service

**What it is.** A **Service** provides a single, stable network address for a group of Pods, and load-balances traffic across whichever of those Pods are currently healthy.

**Why it exists.** Here's the core problem, stated precisely: Pods are **not stable**. They get created and destroyed constantly — by rollouts, by crashes and restarts, by scaling. Every time a Pod is recreated, it gets a **new IP address**. If ShopSphere's frontend tried to talk directly to a specific backend Pod's IP, that connection would break the moment that Pod was replaced.

**What problem it solves.** A Service gives you one address that *never changes*, and continuously, automatically tracks which real Pods currently exist behind it — using the exact same label-selector mechanism you just learned for ReplicaSets in Chapter 6.3.

```text
   Frontend Pod
        |
   talks to: "backend-service" (stable name, stable ClusterIP)
        |
        v
   +----------------------------------------+
   |            Service                       |
   |     selector: app=shopsphere-backend       |
   +----------------------------------------+
        |              |              |
   Backend Pod 1   Backend Pod 2   Backend Pod 3
   (IP changes      (IP changes      (IP changes
    on restart)       on restart)       on restart)
```

**Relationship to other objects.** A Service doesn't "contain" Pods — it dynamically discovers them via a label selector, exactly like a ReplicaSet does. This is worth sitting with: a Service and a ReplicaSet can independently select the *same* Pods for two entirely different purposes (one for lifecycle management, one for network routing) — Kubernetes's label system is a general-purpose grouping mechanism used all over the API, not something specific to any one object kind.

**Service types, explained precisely:**

- **ClusterIP** (the default) — a stable IP address reachable only from *inside* the cluster. This is what you use for internal communication — exactly ShopSphere's frontend-to-backend traffic.
- **NodePort** — additionally exposes the Service on a static port on every node's own IP, making it reachable from outside the cluster without a cloud load balancer. Mostly used for development/testing, or as a building block underneath other approaches.
- **LoadBalancer** — additionally provisions an actual cloud load balancer (recall the Cloud Controller Manager from Part VI), giving the Service a real external IP or DNS name. This is what you'd use to expose something directly to the internet, though we'll mostly route external traffic through Ingress instead (Part VIII), using ClusterIP Services internally behind it.

**YAML example:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shopsphere-backend
spec:
  type: ClusterIP
  selector:
    app: shopsphere-backend
  ports:
    - port: 8000
      targetPort: 8000
```

`port` is the port the *Service* listens on; `targetPort` is the port on the *Pod* traffic actually gets forwarded to — they're often the same number, but don't have to be, which is genuinely useful when you want a friendly, standard Service port fronting a Pod that happens to listen somewhere less conventional.

**Service discovery by name — connecting back to Part IV.** Recall Docker Compose's automatic name-based DNS between services. Kubernetes does the exact same thing, via **CoreDNS** (introduced properly in Part VIII): any Pod can reach this Service simply by name — `http://shopsphere-backend:8000` — and CoreDNS resolves that name to the Service's stable ClusterIP automatically. Same underlying idea you already learned, one layer up.

**Real-world use.** Every Deployment that needs to be reachable by anything else — internally or externally — needs a matching Service. This is one of the most common beginner gaps: writing a perfectly correct Deployment, and then being confused why nothing can reach it, because no Service was ever created pointing at it.

**Troubleshooting.** `kubectl get endpoints shopsphere-backend` is the single most useful debugging command here — it shows the actual Pod IPs the Service has currently discovered. If it's empty, the Service's `selector` doesn't match any existing Pod's labels — almost always a typo between the Service's `selector` and the Deployment's `template.metadata.labels`.

**Interview question (advanced, and genuinely common):** "You have a Deployment with healthy, Running Pods, and a Service pointing at them, but requests to the Service time out. What do you check?" — First, `kubectl get endpoints` — if it's empty, the Service's selector doesn't match the Pods' labels, which is the single most common root cause of this exact symptom; if endpoints *are* populated, check whether the Pods are actually passing their **readiness probe** (Part VIII), since a Pod that's Running but not Ready is deliberately excluded from a Service's endpoints.

---

## Chapter 6.6 — Namespace

**What it is.** A **Namespace** is a way to logically partition a single cluster into multiple isolated-ish "virtual clusters" — separate scopes for object names, and a boundary for applying resource limits and access controls.

**Why it exists.** Without namespaces, every object name in the entire cluster would need to be globally unique — impractical the moment more than one team or environment shares a cluster. Namespaces let ShopSphere run, say, a `staging` and a `production` copy of the backend Deployment, both legitimately named `shopsphere-backend`, without collision.

**What problem it solves.** Logical separation of environments or teams within one physical cluster, plus a natural boundary for RBAC permissions (Part VIII) and ResourceQuotas (Part VIII).

**YAML example:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopsphere-staging
```

```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml -n shopsphere-staging
```

**Real-world use.** A typical real-world pattern: separate namespaces per environment (`dev`, `staging`, `production`) or per team, each with its own RBAC rules and resource quotas — not a full separate cluster per environment (which is far more expensive and operationally heavier, though large enough companies sometimes do use fully separate clusters too, particularly for hard production/non-production isolation).

**Troubleshooting.** The most common beginner mistake here isn't really a "bug" — it's forgetting `-n <namespace>` (or `--all-namespaces`) and being confused why `kubectl get pods` shows nothing, when the Pods are actually running happily in a different namespace than `default`.

**Interview question (beginner):** "Do namespaces provide strong security isolation between workloads, similar to separate clusters?" — Not by default — a namespace is primarily an organizational and access-control boundary; without additional controls like NetworkPolicy (Part VIII) restricting cross-namespace traffic, Pods in different namespaces can still reach each other over the network. Namespaces organize and scope access; they don't automatically network-isolate.

---

## Chapter 6.7 — ConfigMap

**What it is.** A **ConfigMap** stores non-sensitive configuration data as key-value pairs, separately from your application's container image, and makes it available to Pods as environment variables or mounted files.

**Why it exists.** Recall the Docker Compose `environment:` block from Part IV, and the `ARG`/`ENV` discussion from Part III — configuration that changes between environments shouldn't be baked into an image. A ConfigMap is Kubernetes's dedicated object for exactly this: configuration that's genuinely fine to be visible in plaintext (a log level, a feature flag, a non-secret URL), decoupled from the image so the *same* image can run in dev, staging, and production with different configuration in each.

**YAML example:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shopsphere-backend-config
data:
  LOG_LEVEL: "info"
  CACHE_TTL_SECONDS: "300"
```

Using it in a Deployment, as environment variables:

```yaml
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
          envFrom:
            - configMapRef:
                name: shopsphere-backend-config
```

Or mounted as files (useful when an application expects an actual config file on disk rather than environment variables):

```yaml
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
      volumes:
        - name: config-volume
          configMap:
            name: shopsphere-backend-config
```

**Real-world use.** Any non-sensitive, environment-specific setting: log levels, feature flags, timeouts, non-secret external URLs.

**Troubleshooting.** If a Pod isn't picking up an expected value, check for a typo in the ConfigMap's `name` in the Deployment's `configMapRef`/`configMap` reference — a mismatched name here fails somewhat silently (the env var or mount simply doesn't appear, rather than throwing an obvious error), so `kubectl describe pod` and checking the actual mounted/env values with `kubectl exec` are your fastest path to confirming it.

**Interview question (intermediate):** "Why use a ConfigMap instead of baking configuration directly into your Docker image?" — It decouples configuration from the image, letting the exact same, already-tested image artifact be promoted unchanged across dev, staging, and production, with only the ConfigMap differing per environment — consistent with the general principle that what you deploy to production should be the very same artifact you tested earlier, not a rebuild.

---

## Chapter 6.8 — Secret

**What it is.** A **Secret** is structurally almost identical to a ConfigMap, but intended for sensitive values — database passwords, API keys, TLS certificates.

**Why it exists, and an important honest caveat.** By default, Kubernetes Secrets are stored in etcd **base64-encoded, not encrypted** — base64 is an *encoding*, not an encryption scheme; anyone with `kubectl get secret -o yaml` access or direct etcd access can trivially decode it. This is precisely the warning flagged back in the book's own outline: **don't treat a Kubernetes Secret as an automatically secure vault.** Real protection requires additional measures — **encryption at rest for etcd** (a cluster-level configuration, covered properly in Part VIII's security chapter), tightly scoped RBAC controlling exactly who can read Secret objects, and, for genuinely sensitive production values, integration with an external secret manager like **AWS Secrets Manager** (which we introduce properly in Part XI, and which mirrors exactly the "external secret manager" pattern we already flagged as the best practice back in Part V's Docker security chapter).

**What problem it solves, even with that caveat.** It still keeps sensitive values out of your Deployment YAML and out of your container image — meaningfully better than an `ENV` line baked into a Dockerfile (Part III) or a plaintext value in a ConfigMap — it's a genuine improvement, just not a complete solution on its own.

**YAML example:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: shopsphere-db-credentials
type: Opaque
data:
  DB_PASSWORD: ZGV2cGFzcw==   # base64 of "devpass" — NOT encrypted, just encoded
```

```bash
echo -n 'devpass' | base64        # produces ZGV2cGFzcw==
echo 'ZGV2cGFzcw==' | base64 -d    # decodes right back to devpass — proving the point above
```

Using it in a Deployment:

```yaml
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: shopsphere-db-credentials
                  key: DB_PASSWORD
```

**Real-world use.** Database credentials, third-party API keys, TLS certificates for Ingress (Part VIII), and image-pull credentials for private registries.

**Troubleshooting.** `kubectl get secret shopsphere-db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d` is genuinely how you'd verify a Secret's actual value during debugging — and internalizing how easy that command is, is itself the whole lesson about why base64 alone isn't real protection.

**Interview question (advanced, and a favorite "gotcha" question):** "Are Kubernetes Secrets encrypted?" — Not by default — values are only base64-encoded in etcd, which is trivially reversible, not encrypted. Real protection requires enabling encryption at rest for etcd, tightly scoping RBAC access to Secret objects, and often integrating with an external secrets manager for genuinely sensitive production values, rather than relying on the Secret object alone.

---

## Chapter 6.9 — Bringing It Together: ShopSphere's Backend on Kubernetes

Let's assemble everything from this Part into one real, working set of manifests — the actual moment ShopSphere's backend runs on Kubernetes.

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shopsphere

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shopsphere-backend-config
  namespace: shopsphere
data:
  LOG_LEVEL: "info"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: shopsphere-db-credentials
  namespace: shopsphere
type: Opaque
data:
  DB_PASSWORD: ZGV2cGFzcw==

---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopsphere-backend
  namespace: shopsphere
  labels:
    app: shopsphere-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopsphere-backend
  template:
    metadata:
      labels:
        app: shopsphere-backend
    spec:
      containers:
        - name: backend
          image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: shopsphere-backend-config
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: shopsphere-db-credentials
                  key: DB_PASSWORD

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: shopsphere-backend
  namespace: shopsphere
spec:
  type: ClusterIP
  selector:
    app: shopsphere-backend
  ports:
    - port: 8000
      targetPort: 8000
```

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml -f secret.yaml -f deployment.yaml -f service.yaml
kubectl get all -n shopsphere
```

Every object here is doing exactly the job this chapter described: the Namespace scopes everything; the ConfigMap and Secret externalize configuration from the image, exactly as Part III and Part V argued they should be; the Deployment maintains 3 self-healing replicas with rollout capability; the Service gives them one stable, discoverable address, insulated from individual Pod IPs constantly changing underneath it.

---

## Chapter 6.10 — Checkpoint

**Beginner:**
1. What is the smallest deployable unit in Kubernetes, and why isn't it "a container"?
2. What's the difference between a ConfigMap and a Secret?

**Intermediate:**
3. Why does a Service need a stable address at all — what specifically changes about Pods that makes this necessary?
4. Explain the three-layer relationship between Deployment, ReplicaSet, and Pod.

**Advanced:**
5. Are Kubernetes Secrets encrypted by default? What would you actually need to add to make them genuinely secure?
6. `kubectl get endpoints` for a Service returns nothing, even though matching Pods are Running. What's the most likely cause, and how would you confirm it?

**Scenario:**
7. A teammate wants to skip creating a Service, arguing the frontend can "just hardcode the backend Pod's IP since it's simple." Explain, using this chapter's concepts, exactly why that will break in production.

---

### Hands-On Lab 6.1 — Deploy ShopSphere's backend for real

**Objective:** Apply the full manifest set from Chapter 6.9 to your kind cluster from Part VI, and verify every layer independently.

**Prerequisites:** The kind cluster from Part VI's lab (recreate it if you deleted it); the manifests above saved as separate files.

**Steps:**

1. Apply everything and watch it come up:
   ```bash
   kubectl apply -f namespace.yaml
   kubectl apply -f configmap.yaml -f secret.yaml -f deployment.yaml -f service.yaml
   kubectl get pods -n shopsphere --watch
   ```
   (If you don't have a real image in ECR yet, swap the image temporarily for `nginx` to prove the mechanics — we'll wire up the real image with CI/CD in Part X.)

2. Verify the Deployment reached desired state:
   ```bash
   kubectl get deployment shopsphere-backend -n shopsphere
   ```

3. Verify the Service found its Pods:
   ```bash
   kubectl get endpoints shopsphere-backend -n shopsphere
   ```

4. Verify the ConfigMap and Secret actually reached the container:
   ```bash
   kubectl exec -n shopsphere deploy/shopsphere-backend -- env | grep -E "LOG_LEVEL|DB_PASSWORD"
   ```

5. Test Service reachability from inside the cluster (a throwaway debug Pod):
   ```bash
   kubectl run -n shopsphere debug --rm -it --image=busybox --restart=Never -- \
     wget -qO- http://shopsphere-backend:8000
   ```

**Expected result:** 3/3 Pods Running and Ready; `endpoints` lists three Pod IPs; the `env` output shows both the ConfigMap and Secret values actually present inside the running container; the `wget` from the debug Pod successfully reaches the Service by name.

**Verification:** step 5 is the real proof — it confirms Service discovery by name is genuinely working, cluster-internally, the same way Compose's service-name DNS worked back in Part IV.

**Troubleshooting:** if step 5 fails, re-run step 3 first — an empty `endpoints` output means the Service's selector doesn't match the Deployment's Pod labels (Chapter 6.5's most common failure mode) — check both YAML files side by side for a typo.

**Cleanup:**
```bash
kubectl delete namespace shopsphere
```
Deleting the Namespace cascades and removes everything inside it — Deployment, Service, ConfigMap, Secret, and every Pod — in one command.

**Challenge:** scale the Deployment to 5 replicas with `kubectl scale deployment shopsphere-backend -n shopsphere --replicas=5`, then re-check `kubectl get endpoints` — confirm it grew to 5 entries automatically, with zero changes needed to the Service itself. Explain in one sentence why the Service didn't need to be touched.

---

*End of Part VII. Part VIII covers Kubernetes networking and storage in depth — Pod IP vs. Service IP, CoreDNS, Ingress and Ingress Controllers, PersistentVolumes and PersistentVolumeClaims — followed immediately by resource management, health probes, scheduling, scaling, and release strategies, completing ShopSphere's path to a genuinely production-grade deployment.*
