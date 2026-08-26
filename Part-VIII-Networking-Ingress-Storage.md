# The ShopSphere DevOps Book
## Part VIII — Kubernetes Networking, Ingress, and Storage

---

### Where we left off

ShopSphere's backend runs on Kubernetes with three self-healing replicas, reachable internally by a stable Service name. Two problems remain before this is a real, usable system: nothing from *outside* the cluster can reach it yet, and the database still has nowhere durable to keep its data if we move it onto Kubernetes too.

---

## Chapter 7.1 — Kubernetes Networking, Precisely

Let's build the full picture of addresses in a Kubernetes cluster, layer by layer, because this is exactly where beginners get tangled up, and precision here pays off for the rest of the book.

### Pod IP

Every Pod gets its own **Pod IP**, from the cluster's internal network — assigned when the Pod starts. Crucially, this IP is **not stable**: recreate the Pod (a crash, a rollout, a reschedule), and it gets a *new* Pod IP. This is exactly the fact that motivated Services in Chapter 6.5.

### Service IP (ClusterIP)

A Service gets its own **ClusterIP**, from a separate internal address range, allocated once and held for the Service's entire lifetime — unlike a Pod IP, this genuinely doesn't change. kube-proxy (introduced in Part VI) is the component that actually makes packets sent to a ClusterIP get transparently routed to one of the real, current Pod IPs behind it.

```text
   Stable forever:        Changes on every Pod restart:
   Service ClusterIP  -->  Pod IP 1
   10.96.12.44             Pod IP 2
                            Pod IP 3
```

### Pod-to-Pod communication

A foundational Kubernetes networking guarantee, worth stating explicitly: **every Pod can reach every other Pod's IP directly, across the entire cluster, without NAT** — regardless of which node either one is running on. This is a deliberate design requirement of Kubernetes networking (implemented by whichever **CNI plugin** — Container Network Interface — the cluster uses, such as Calico, Cilium, or the AWS VPC CNI we'll use with EKS in Part XI). You don't manage this yourself; it's a property the cluster's networking layer guarantees underneath everything else in this chapter.

### Pod-to-Service communication

This is what you actually use in practice, and it's what Chapter 6.5 already covered: a Pod talks to a Service's stable ClusterIP (usually via its DNS name, next), and kube-proxy transparently load-balances that to one of the Service's currently-healthy backing Pods.

### CoreDNS and service discovery

**Simple explanation:** CoreDNS is Kubernetes's internal phonebook — it's what lets a Pod look up `shopsphere-backend` and get back that Service's ClusterIP, instead of anyone needing to hardcode IP addresses anywhere.

**Proper definition:** **CoreDNS** is the DNS server that runs inside the cluster (itself deployed as Pods, in `kube-system` — you actually saw it in Part VI's lab output) and provides name resolution for Kubernetes Services and Pods.

**The naming convention worth knowing precisely:** a Service named `shopsphere-backend` in namespace `shopsphere` is reachable, cluster-wide, at `shopsphere-backend.shopsphere.svc.cluster.local` — and, importantly, at just `shopsphere-backend` from *within the same namespace*, which is the short form you'll use in practice, exactly as we did in Part VII's lab. This is the same DNS concept from Part I, applied at the cluster level, using the exact same "give it a stable name because the address changes" motivation as Docker Compose's service-name DNS in Part IV — the third and final layer where you've now seen this identical idea.

### External traffic: the piece still missing

Everything so far is internal-cluster-only. ShopSphere's actual customers, out on the real internet, still can't reach any of this. That gap is exactly what Ingress exists to close.

**Interview question (intermediate):** "What's the difference between a Pod IP and a ClusterIP?" — A Pod IP belongs to one specific Pod and changes whenever that Pod is recreated; a ClusterIP belongs to a Service, is stable for the Service's entire lifetime, and load-balances across whichever Pods currently match its selector.

---

## Chapter 7.2 — Ingress

### The problem, precisely

A `LoadBalancer`-type Service (Chapter 6.5) *can* expose something to the internet — but provisioning one real cloud load balancer per Service gets expensive and unwieldy fast. ShopSphere doesn't just have one thing to expose — it has a frontend and a backend API, potentially both needing external routing, plus TLS termination, plus the ability to route based on the URL path or hostname a request came in on.

**Simple explanation:** Ingress is a single, smart front door for the whole cluster — instead of one load balancer per service, you get one entry point that routes requests to the right internal Service based on the hostname or path in the request.

**Proper definition:** an **Ingress** is a Kubernetes object that defines HTTP/HTTPS routing rules — directing external traffic to internal Services based on hostname and/or URL path — but it does nothing on its own without an **Ingress Controller** actually implementing those rules.

**Why the split between Ingress (the rules) and Ingress Controller (the implementation) exists.** This mirrors a pattern you've now seen a few times in this book: Kubernetes defines a general object/interface, and a pluggable, swappable implementation does the actual work — the Cloud Controller Manager pattern from Part VI is another example. Popular Ingress Controllers include NGINX Ingress Controller, and — since we're on AWS — the **AWS Load Balancer Controller**, which we'll use directly in Part XI to provision a real Application Load Balancer from Kubernetes Ingress objects.

```text
                    Internet
                       |
                       v
              AWS Load Balancer
       (provisioned by the Ingress Controller)
                       |
                       v
                  Ingress Controller
             (reads Ingress rules, routes accordingly)
                       |
          +------------+------------+
          |                         |
   Host: shop.example       Host: api.shop.example
   -> frontend-service       -> backend-service (ClusterIP)
   (ClusterIP)
```

### Host-based and path-based routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shopsphere-ingress
  namespace: shopsphere
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shopsphere-frontend
                port:
                  number: 3000
    - host: api.shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shopsphere-backend
                port:
                  number: 8000
```

This single Ingress routes two different hostnames to two different internal Services — one Ingress Controller, one (or one small set of) real cloud load balancer(s) behind it, instead of one per Service.

**Path-based routing** works the same way, keying off `path` instead of (or in addition to) `host` — for example, routing `/api/*` to the backend and everything else to the frontend, under a single shared hostname.

### TLS / HTTPS

```yaml
spec:
  tls:
    - hosts:
        - shop.example.com
      secretName: shopsphere-tls-cert
  rules:
    - host: shop.example.com
      # ...
```

The referenced Secret (recall Chapter 6.8 — same object type, this time holding a TLS certificate rather than an application secret) contains the certificate and private key; the Ingress Controller uses it to **terminate TLS** — decrypting HTTPS traffic at the cluster boundary, so the internal Services themselves can typically speak plain HTTP behind it. In production, certificates are often issued and renewed automatically via **cert-manager**, a widely-used add-on that integrates with Let's Encrypt or AWS Certificate Manager — worth knowing the name, even though we won't build it out fully in this book.

### AWS Load Balancer integration

On EKS specifically, the **AWS Load Balancer Controller** watches for Ingress objects and automatically provisions a real **Application Load Balancer (ALB)** to match — this is the Cloud Controller Manager pattern from Part VI, made concrete. We'll set this up for real in Part XI.

### The Gateway API — a brief, honest look ahead

Ingress has some real, acknowledged limitations — its capabilities are somewhat constrained, and different Ingress Controllers extend it in inconsistent, non-portable ways via annotations (notice the `kubernetes.io/ingress.class` annotation above — that's exactly this kind of controller-specific extension). The **Gateway API** is a newer, more expressive, and more standardized successor, modeling routing as a cleaner set of composable objects. It's genuinely worth knowing this exists and is actively gaining adoption — but Ingress remains extremely widely deployed and is still the right starting point to understand deeply first, which is why this book teaches Ingress as the primary path.

**Interview question (advanced):** "Why doesn't Kubernetes give every Service that needs external access its own dedicated cloud load balancer?" — Cost and manageability — provisioning one real load balancer per Service doesn't scale well as the number of services grows. Ingress consolidates external routing behind a single (or small number of) load balancer(s), using host- and path-based rules to direct traffic to the correct internal ClusterIP Service, with an Ingress Controller doing the actual routing work.

---

## Chapter 7.3 — Kubernetes Storage

### Why this needs its own careful treatment

Recall Chapter 6.9: ShopSphere's backend now runs happily on Kubernetes. But we haven't moved the database there — and there's a good reason for the caution. Kubernetes Pods are meant to be disposable and replaceable (the exact same philosophy as disposable containers from Part III, one layer up) — but a database's *data* is the opposite of disposable. This chapter is about the mechanism that lets Kubernetes support genuinely stateful workloads without contradicting its own replaceable-Pod philosophy.

```text
   Pod
    |
   PVC   (PersistentVolumeClaim — "I need 10GB of storage")
    |
   PV    (PersistentVolume — an actual piece of provisioned storage)
    |
StorageClass  (the "recipe" for how to dynamically provision that PV)
    |
Actual Storage  (e.g., an AWS EBS volume)
```

### PersistentVolume (PV)

**What it is.** A **PersistentVolume** represents an actual piece of storage in the cluster — provisioned either ahead of time by an administrator, or dynamically on demand (which is by far the more common modern approach, and what we'll focus on).

**Why it exists.** It gives Kubernetes a consistent object to represent "a piece of real, durable storage," regardless of what kind of underlying storage system actually backs it — AWS EBS, AWS EFS, or something else entirely.

### PersistentVolumeClaim (PVC)

**What it is.** A **PersistentVolumeClaim** is a *request* for storage, made by a Pod (indirectly, via its owning Deployment/StatefulSet) — "I need 10GB, with these access characteristics" — which Kubernetes then matches (or dynamically provisions) to satisfy with an actual PersistentVolume.

**Why this two-object split (PV and PVC) exists, rather than just one object.** This is a genuinely good interview-level question to be able to answer precisely: it separates the concerns of *requesting* storage (which application developers do, without needing to know anything about the underlying infrastructure) from *provisioning* storage (which is an infrastructure concern, often handled entirely automatically via a StorageClass). A developer writing a PVC for ShopSphere's database doesn't need to know or care whether it's backed by AWS EBS or something else — that's exactly the abstraction boundary this split is designed to provide.

### StorageClass and dynamic provisioning

**What it is.** A **StorageClass** defines *how* to dynamically provision storage when a PVC requests it — which underlying storage system to use, and with what parameters (disk type, replication behavior, and so on).

**Why it exists.** Without dynamic provisioning, someone would need to manually create a matching PersistentVolume by hand, every single time an application needed storage — precisely the kind of manual, error-prone process this entire book has been working to eliminate. With a StorageClass, a PVC simply references it by name, and the actual PV gets created automatically, on demand.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shopsphere-db-data
  namespace: shopsphere
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 20Gi
```

`accessModes` describes how the volume can be mounted: **ReadWriteOnce (RWO)** — mountable as read-write by a single node at a time (the normal case for a database, and specifically what a typical AWS EBS volume supports); **ReadOnlyMany (ROX)** — read-only, by many nodes simultaneously; **ReadWriteMany (RWX)** — read-write, by many nodes simultaneously (this requires storage like AWS EFS, since a standard EBS volume can't do this).

### Stateful applications and StatefulSet

We built ShopSphere's backend as a Deployment — completely appropriate, because its Pods are interchangeable: any replica can handle any request, and none of them individually holds unique data. A database is fundamentally different — replicas (if you run more than one) are *not* interchangeable; each has its own identity and its own data. This is what **StatefulSet** exists for: it provides stable, predictable Pod names (`shopsphere-db-0`, `shopsphere-db-1`, rather than random suffixes), stable per-replica storage (each replica keeps its *own* PVC across restarts, rather than sharing one), and ordered, sequential startup/shutdown. We're deliberately not going deep on StatefulSet mechanics in this book, for an honest, important reason we cover next.

### AWS EBS and EFS, and — importantly — the production choice this book actually recommends

**Amazon EBS (Elastic Block Store)** provides block storage — durable, but attachable to only one node at a time (matching `ReadWriteOnce`). **Amazon EFS (Elastic File System)** provides shared network file storage, attachable to many nodes simultaneously (matching `ReadWriteMany`) — used for things like shared upload directories, not typically for a primary relational database.

Here's the honest, important production guidance this book promised in its very first chapter, under "production vs. local development": **for ShopSphere's actual production PostgreSQL database, this book recommends Amazon RDS — a fully managed database service — rather than running PostgreSQL as a StatefulSet inside Kubernetes at all.** We'll cover RDS properly in Part XI. The reasoning is worth stating precisely, because "should we run our database in Kubernetes?" is a genuinely common real-world debate:

- Running a production-grade, highly-available database *inside* Kubernetes (correct StatefulSet configuration, replication, automated backups, failover, patching) is a substantial, ongoing operational burden — one that a managed service like RDS already solves, thoroughly, as its entire product.
- Kubernetes-native storage concepts — PV, PVC, StorageClass — are still genuinely important to understand deeply (they're used constantly for other stateful needs: file uploads, caches with persistence requirements, internal tooling), and they're a very real part of the Kubernetes interview and certification body of knowledge, which is exactly why this book covers them properly, in depth, above.
- But "we could run it in Kubernetes" and "we *should* run it in Kubernetes" are different questions — and for ShopSphere's actual production database, this book comes down clearly on using a managed service instead.

We'll still use a PostgreSQL *container* with a PVC in the **local development / personal lab** context throughout this book — exactly the "Local vs. Production" progression promised from Chapter 1 onward — but production gets RDS.

| | Production | Personal Lab / Local Dev |
|---|---|---|
| Database | Amazon RDS (managed, automated backups, Multi-AZ) | PostgreSQL container + PVC (or plain Docker volume locally) |
| Reasoning | Operational burden of self-managed stateful workloads outweighs the control gained | Fast, free/cheap, perfectly adequate for learning and local iteration |

**Interview question (advanced, and a genuinely common real one):** "Would you run your production database as a Kubernetes StatefulSet, or use a managed service like RDS?" — There's no single universally correct answer, and a strong candidate acknowledges the tradeoff explicitly: StatefulSets can absolutely work for production databases and some organizations do run them successfully, but it demands significant operational expertise (replication, backup automation, failover, patching) that a managed service already provides out of the box — for most teams, especially smaller ones, a managed service like RDS is the pragmatic default, reserving self-managed StatefulSet databases for cases with specific requirements a managed service genuinely can't meet.

---

## Chapter 7.4 — Checkpoint

**Beginner:**
1. What's the difference between a PersistentVolume and a PersistentVolumeClaim?
2. What does an Ingress Controller actually do, that an Ingress object alone doesn't?

**Intermediate:**
3. Why can a Pod reach any other Pod's IP directly, cluster-wide, but external internet traffic can't reach a Pod IP directly?
4. Explain host-based vs. path-based Ingress routing, with a ShopSphere-specific example of each.

**Advanced:**
5. Why does Kubernetes separate the concerns of "PVC" and "StorageClass" instead of having a Pod request storage directly from a specific underlying provider?
6. Make the case for and against running ShopSphere's production PostgreSQL database as a Kubernetes StatefulSet instead of Amazon RDS.

**Scenario:**
7. Customers report ShopSphere's storefront is unreachable from the public internet, but internal `kubectl exec` tests show the frontend Service responds correctly from inside the cluster. Where would you look, in order, based on everything in this chapter?

---

### Hands-On Lab 7.1 — Expose ShopSphere with Ingress, locally

**Objective:** Get external, browser-reachable routing working for the backend Deployment from Part VII, using a local Ingress Controller.

**Prerequisites:** The kind cluster and `shopsphere` namespace resources from Part VII's lab.

**Steps:**

1. Install the NGINX Ingress Controller into your kind cluster (kind has a documented setup for this — since installation specifics can shift, check the current kind documentation for the exact manifest URL rather than relying on a possibly-stale one here):
   ```bash
   kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
   kubectl wait --namespace ingress-nginx \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/component=controller \
     --timeout=90s
   ```

2. Create an Ingress pointing at the backend Service from Part VII:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: shopsphere-ingress
     namespace: shopsphere
     annotations:
       kubernetes.io/ingress.class: nginx
   spec:
     rules:
       - http:
           paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: shopsphere-backend
                   port:
                     number: 8000
   ```
   ```bash
   kubectl apply -f ingress.yaml
   ```

3. Confirm the Ingress picked up an address:
   ```bash
   kubectl get ingress -n shopsphere
   ```

4. Test it (kind typically maps the Ingress Controller to localhost, depending on your cluster config):
   ```bash
   curl http://localhost/
   ```

**Expected result:** the Ingress shows an assigned address; `curl` reaches the backend through the Ingress Controller, proving external-style traffic now flows all the way through Ingress → Service → Pod.

**Verification:** `kubectl logs -n ingress-nginx <controller-pod>` shows the incoming request being routed, giving you direct visibility into the routing decision, not just a successful response.

**Troubleshooting:** if `curl` fails, first confirm the Ingress Controller Pod itself is Running and Ready (`kubectl get pods -n ingress-nginx`) before suspecting the Ingress object's rules — this is the same "work outward from the most fundamental layer" debugging instinct we'll formalize properly in Part IX.

**Cleanup:**
```bash
kubectl delete namespace shopsphere
kubectl delete namespace ingress-nginx
```

**Challenge:** add a second path rule routing `/health` specifically to a different Service (or the same one, for the exercise), and confirm with `curl` that both paths route correctly and independently.

---

*End of Part VIII. Part IX covers resource requests and limits, QoS classes and OOMKilled, health probes (liveness/readiness/startup), and scheduling (node affinity, taints and tolerations) — the configuration that turns a working Deployment into one that behaves reliably under real production load.*
