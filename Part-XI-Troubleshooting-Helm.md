# The ShopSphere DevOps Book
## Part XI — Kubernetes Troubleshooting and Helm

---

### Where we left off

ShopSphere is reliable, scalable, and secured. This Part has two goals: build real, systematic troubleshooting instinct — the actual day-to-day job of a DevOps engineer, arguably more than anything else in this book — and then package everything we've built by hand across Parts VII–X into a single, reusable, versioned Helm chart.

---

## Chapter 10.1 — A Systematic Troubleshooting Methodology

Before we go incident by incident, let's establish the *process*, because that's the actually transferable skill — memorizing this book's specific incidents matters far less than internalizing how to approach one you've never seen before.

```text
        "The application isn't working"
                      |
                      v
          Is the Pod even Running?
           (kubectl get pods)
                      |
        +-------------+-------------+
       No                          Yes
        |                            |
        v                            v
   Check Events                 Is it Ready?
 (kubectl describe pod)       (READY column, e.g. 1/1)
        |                            |
        v                    +-------+-------+
   Check Logs                No               Yes
 (kubectl logs, and            |                |
  --previous if it              v                v
  restarted)                Check probe      Check the Service
        |                   config/logs      (kubectl get endpoints)
        v                                          |
   Root cause found?                      +--------+--------+
        |                                No match          Has endpoints
        v                                    |                  |
   Fix, verify,                              v                  v
   document                          Check label selectors   Check DNS
                                       match (Deployment           |
                                       template vs Service)        v
                                                              Check Ingress
                                                                    |
                                                                    v
                                                            Check NetworkPolicy
                                                                    |
                                                                    v
                                                              Check the Node
```

The core discipline here: **work outward from the smallest, most fundamental layer first.** Don't jump straight to "maybe it's a networking issue" before confirming the Pod is even Running. Almost every real incident resolves faster once you resist the urge to guess and instead walk this chain in order.

---

## Chapter 10.2 — Common Failure Modes, One at a Time

For each of these, we'll deliberately break a piece of ShopSphere, show you the symptom, and walk the diagnosis.

### CrashLoopBackOff

**Symptom:**
```bash
kubectl get pods -n shopsphere
# NAME                    READY   STATUS             RESTARTS
# shopsphere-backend-xyz   0/1    CrashLoopBackOff   5
```

**What it actually means.** The container starts, then exits (crashes, or exits cleanly with a nonzero code) — and Kubernetes, following the reconciliation loop from Chapter 5.1, keeps trying to restart it, with an exponential backoff delay between attempts (hence "back-off").

**Diagnosis:**
```bash
kubectl logs shopsphere-backend-xyz -n shopsphere --previous
```
The `--previous` flag is essential here — by the time you're looking, the container may already be in a fresh crash-restart cycle, and the *current* logs might be empty or unhelpful; `--previous` gets you the logs from the last attempt that actually crashed.

**Common root causes, ranked by real-world frequency:** a genuine application startup error (a missing environment variable, a bad config value causing an unhandled exception); a missing dependency the app expects to reach immediately at startup with no retry logic; an incorrect `CMD`/`ENTRYPOINT` (recall Part III) that doesn't correctly start the actual application process; or — often overlooked — a liveness probe (Chapter 8.2) that's misconfigured to fail immediately, restarting an otherwise-healthy container repeatedly.

### ImagePullBackOff / ErrImagePull

**Symptom:** the Pod never reaches Running at all; `STATUS` shows `ImagePullBackOff` or `ErrImagePull`.

**Diagnosis:**
```bash
kubectl describe pod shopsphere-backend-xyz -n shopsphere
```
The Events section names the exact problem directly — this is one of the most reliably self-explanatory failure modes, if you actually read the Events output instead of guessing.

**Common root causes:** a typo in the image name or tag (recall Part III/V's tagging discussion — this is exactly why explicit, correct tags matter); the image genuinely doesn't exist at that tag in the registry (maybe it was never pushed, or was pushed under a different tag); or — a very common one connecting directly to Part V's ECR authentication — missing or expired registry credentials (an `imagePullSecrets` reference missing from the Pod spec, or a credential that's expired).

### Pending

**Symptom:** the Pod stays in `Pending` and never gets scheduled at all.

**Diagnosis:**
```bash
kubectl describe pod shopsphere-backend-xyz -n shopsphere
# Look at the Events section specifically for a Scheduler message
```

**Common root causes, all directly traceable to concepts from Chapter 5.3 and Part IX:** insufficient cluster resources for the Pod's requests (no node has enough available CPU/memory) — exactly the "HPA scaled up, but Pods are stuck Pending" scenario flagged in Part X's checkpoint; a `nodeSelector` or required node affinity rule (Chapter 8.3) that no existing node actually satisfies; an unresolved PersistentVolumeClaim (Chapter 7.3) that can't be bound to any available storage; or a taint (Chapter 8.3) on every available node with no matching toleration on the Pod.

### ContainerCreating (stuck)

**Symptom:** the Pod sits in `ContainerCreating` for an unusually long time, rather than the brief, normal transition through this phase.

**Common root causes:** a slow or stuck image pull (large image, slow registry, or network issues reaching the registry — worth checking `describe pod` Events for pull progress); a volume that can't be mounted (a PVC bound to storage in the wrong Availability Zone from the node trying to mount it — a genuinely common real EBS-specific gotcha we'll return to in Part XII); or a ConfigMap/Secret referenced by the Pod that doesn't actually exist (Chapter 6.7/6.8's "silent mismatch" mistake, showing up here as a stuck mount instead).

### OOMKilled

Covered thoroughly in Chapter 8.1 — the summary version for this troubleshooting reference: `kubectl describe pod` shows `Reason: OOMKilled` under the container's Last State. **Fix:** raise the memory limit if the usage is legitimate, or find and fix a genuine memory leak if the usage is not — `kubectl top pod` over time, and application-level memory profiling, are how you tell the difference between the two.

### FailedScheduling

This is really the same root cause category as Pending above — `FailedScheduling` is the specific Event *reason* string you'll see in `describe pod` output that explains *why* a Pod is Pending. Worth knowing as the literal text to search for.

### Readiness probe failure / Liveness probe failure

Covered thoroughly in Chapter 8.2. Diagnosis reference: `kubectl describe pod` shows probe failure events directly, including the specific HTTP status code or timeout that caused the failure — always start there before guessing at the cause.

### Service not reachable

Already given a full dedicated treatment in Chapter 6.5's troubleshooting section — the summary chain: check `kubectl get endpoints` first (empty means a selector/label mismatch); if populated, check whether Pods are actually Ready (not just Running); then check the Service's `port`/`targetPort` mapping.

### DNS failure

**Symptom:** an application error indicating it couldn't resolve a hostname like `shopsphere-backend` or `db`.

**Diagnosis:**
```bash
kubectl run -n shopsphere dns-test --rm -it --image=busybox --restart=Never -- \
  nslookup shopsphere-backend
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

**Common root causes:** CoreDNS Pods themselves unhealthy or under resource pressure (check their own status and logs, just like any other workload); a typo in the service name being queried; or, less commonly, a custom `dnsPolicy` on the Pod that isn't using the cluster's DNS at all.

### Ingress failure

**Diagnosis chain, building directly on Chapter 7.2:** confirm the Ingress Controller Pods themselves are Running first (`kubectl get pods -n ingress-nginx` or equivalent); then check `kubectl describe ingress` for rule-parsing errors; then check the Ingress Controller's own logs for the specific request in question — they usually log routing decisions explicitly, which is often the fastest way to see exactly which backend Service a request was (or wasn't) routed to.

### NetworkPolicy failure

**Symptom:** a Pod that *should* be able to reach another Pod suddenly can't, right after a NetworkPolicy was introduced.

**Diagnosis:** review the NetworkPolicy's `podSelector` and `ingress.from` rules for a labeling mismatch — this is a very similar class of mistake to the Service-selector mismatch in Chapter 6.5, just applied to network access instead of traffic routing. Also recall Chapter 9.3's caveat: confirm the CNI plugin actually enforces NetworkPolicy at all — if it doesn't, the policy silently does nothing, which can just as easily present as "why didn't blocking this even work," the inverse failure mode.

### Storage/PVC failure

**Symptom:** a Pod stuck in `Pending` or `ContainerCreating`, with Events referencing volume binding or mounting.

**Diagnosis:**
```bash
kubectl get pvc -n shopsphere
kubectl describe pvc shopsphere-db-data -n shopsphere
```

**Common root causes:** no StorageClass available to satisfy the PVC's request; a capacity request exceeding what's available; or — the AZ-mismatch issue flagged above — a PV bound in one Availability Zone while the Pod got scheduled onto a node in a different one, a real, common EBS-specific production gotcha we'll revisit properly with EKS in Part XII.

---

## Chapter 10.3 — Real-World Incident Walkthroughs

Let's run through several of this book's promised incident scenarios properly, in the "symptom → what would you check → diagnosis → root cause → fix → prevention" format.

### Incident: "New deployment keeps crashing"

**Symptoms:** after `kubectl apply`, backend Pods cycle through CrashLoopBackOff.
**What would you check?** `kubectl logs --previous`, first, always.
**Diagnosis:** logs show `KeyError: 'DATABASE_URL'` — the application expects an environment variable that isn't present.
**Root cause:** the new Deployment YAML was updated, but the ConfigMap `envFrom` reference (Chapter 6.7) was accidentally left out of this revision.
**Fix:** restore the `envFrom` block, reapply.
**Prevention:** a CI validation step (we'll build exactly this in Part XIII) that diffs manifests and flags removed `envFrom`/`env` references before they reach production, plus a startup-time explicit check-and-fail-loudly pattern in the application itself, rather than a bare `KeyError`.

### Incident: "Pods are stuck in Pending"

**Symptoms:** an HPA scale-up event created new Pods; they never leave `Pending`.
**What would you check?** `kubectl describe pod` Events, immediately.
**Diagnosis:** Events show `0/3 nodes are available: 3 Insufficient cpu`.
**Root cause:** the cluster's existing nodes are already at capacity, and Cluster Autoscaler (Chapter 9.1) isn't configured — so the HPA is correctly trying to add Pods, but nothing is adding the node capacity to actually host them.
**Fix:** enable and correctly configure Cluster Autoscaler (or a managed node group with autoscaling, on EKS — Part XII).
**Prevention:** this is precisely the "Scaling Pods vs. Scaling Nodes" distinction from Chapter 9.1 — treat HPA and Cluster Autoscaler as a matched pair from the start, not HPA alone.

### Incident: "Application works inside the Pod but not from the internet"

**Symptoms:** `kubectl exec` into the Pod and `curl localhost:8000` succeeds; external requests through the real domain fail.
**What would you check, in order?** Service endpoints, then Ingress, then the Ingress Controller's own health, then DNS for the actual public hostname.
**Diagnosis:** `kubectl get endpoints` is populated correctly, `kubectl describe ingress` shows no errors — but the Ingress Controller's logs show zero incoming requests at all for that hostname.
**Root cause:** the public DNS record for the domain was never actually pointed at the load balancer's address — a pure Part I-level DNS misconfiguration, entirely outside the cluster itself.
**Fix:** correct the DNS record (we'll cover Route 53 specifically in Part XII).
**Prevention:** this incident is a genuinely good illustration of why the troubleshooting chain in Chapter 10.1 works outward layer by layer — jumping straight to "maybe it's the Ingress config" would have wasted time on a component that was actually fine.

### Incident: "Deployment succeeded but users still see the old version"

**Symptoms:** `kubectl rollout status` reports success; the new version's Pods are Running and Ready; but the storefront still visibly shows old content.
**What would you check?** Whether the *browser* or a CDN/caching layer in front of the Ingress is serving a cached response — this one is often not a Kubernetes problem at all.
**Diagnosis:** confirming, via `curl -I` with cache-busting headers directly against the backend, that the new version genuinely is being served correctly at the cluster level.
**Root cause:** aggressive client-side or CDN caching, unrelated to the deployment itself.
**Fix:** appropriate cache-control headers on responses, and/or a cache invalidation step added to the deployment pipeline.
**Prevention:** a genuinely important, humbling lesson: **not every "deployment" symptom is actually a Kubernetes problem** — a disciplined troubleshooter verifies where in the full request path (browser → CDN → Ingress → Service → Pod) the stale behavior is actually coming from, rather than assuming the newest, most exciting technology in the stack is automatically the culprit.

---

## Chapter 10.4 — Helm

### The problem, precisely

Look back at Chapter 6.9's full manifest set — Namespace, ConfigMap, Secret, Deployment, Service — and now imagine maintaining separate near-duplicate copies of all of it for `dev`, `staging`, and `production`, each with slightly different replica counts, image tags, and resource limits. Hand-copying and manually editing YAML across environments is exactly the kind of repetitive, error-prone process this entire book has worked to eliminate at every previous layer.

**Simple explanation:** Helm lets you turn a set of Kubernetes manifests into a templated, reusable package — with the specific values that differ between environments pulled out into one small, easy-to-diff configuration file, instead of duplicated across many nearly-identical YAML files.

**Proper definition:** **Helm** is a package manager for Kubernetes; a **chart** is a Helm package — a directory of templated Kubernetes manifests plus a `values.yaml` file defining configurable defaults; a **release** is one deployed instance of a chart, with a specific set of values applied.

### Why Helm exists, tied directly to earlier chapters

- It solves the "same YAML, different values per environment" duplication problem directly, using Go templating.
- It gives deployments a name and a genuine, trackable revision history — `helm history`, `helm rollback` — conceptually similar to `kubectl rollout history` (Chapter 9.2), but covering an entire *set* of related objects together as one coordinated unit, rather than one Deployment at a time.
- It supports **chart dependencies** — ShopSphere's chart could depend on the official, community-maintained PostgreSQL or Redis charts for local/lab use, rather than reinventing that YAML from scratch.
- **Helm repositories** let charts be versioned, shared, and reused across teams and companies, the same fundamental idea as a container registry (Part V), applied to packaged sets of Kubernetes configuration instead of container images.

### Anatomy of a chart

```text
shopsphere-backend/
├── Chart.yaml           # chart metadata: name, version, description
├── values.yaml           # default configuration values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── _helpers.tpl      # reusable template snippets
```

**`Chart.yaml`:**
```yaml
apiVersion: v2
name: shopsphere-backend
description: ShopSphere backend API
version: 0.1.0
appVersion: "1.0"
```

**`values.yaml`:**
```yaml
replicaCount: 3
image:
  repository: <account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend
  tag: v1
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
config:
  logLevel: info
```

**`templates/deployment.yaml`** — this is Chapter 6.9's Deployment manifest, converted directly into a template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-backend
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-backend
    spec:
      containers:
        - name: backend
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
```

Every `{{ .Values.* }}` reference pulls from `values.yaml` — or from an override file supplied at install/upgrade time, which is exactly how per-environment differences get handled cleanly.

### Functions and variables in templates

Helm templates use Go's templating language, with Helm-specific built-in objects and functions layered on top — `.Release.Name`, `.Release.Namespace`, `.Values.*` (as seen above), `.Chart.*` for chart metadata, plus functions like `toYaml`, `nindent` (for correct YAML indentation when inserting a nested block, as used above), `quote`, and conditionals (`{{- if .Values.ingress.enabled }}`) for optionally including entire blocks of a manifest based on a value.

### Installing, upgrading, rolling back

```bash
helm install shopsphere-prod ./shopsphere-backend \
  --namespace shopsphere \
  --values values-production.yaml

helm upgrade shopsphere-prod ./shopsphere-backend \
  --namespace shopsphere \
  --values values-production.yaml \
  --set image.tag=v2

helm rollback shopsphere-prod 1 --namespace shopsphere

helm history shopsphere-prod --namespace shopsphere
```

`values-production.yaml` overrides only what genuinely differs from the chart's defaults — for example:

```yaml
replicaCount: 6
image:
  tag: v1.4.2
```

— while `values-staging.yaml` might override differently (fewer replicas, a different tag), with both sharing the exact same underlying chart and templates, guaranteeing staging and production are structurally identical, differing only in the specific values that are genuinely meant to differ.

### Helm repositories and chart dependencies

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install shopsphere-db bitnami/postgresql --namespace shopsphere
```

For local/lab use specifically, pulling a well-maintained community chart for PostgreSQL or Redis is often far more practical than writing that StatefulSet/PVC configuration by hand — recall Chapter 7.3's own honest recommendation that production ShopSphere uses RDS anyway, so this community chart is genuinely a **personal-lab-only** convenience, not something we're suggesting for the production database.

**Interview question (intermediate):** "Why would a team use Helm instead of maintaining raw Kubernetes YAML directly?" — Helm eliminates duplicated, near-identical YAML across environments by extracting the values that genuinely differ into a small, reviewable `values.yaml` (or environment-specific override files), while keeping the underlying templates identical — and it adds release versioning and rollback for a coordinated *set* of objects together, not just one object at a time the way `kubectl rollout` does.

---

## Chapter 10.5 — Checkpoint

**Beginner:**
1. What's the first command you should run when a Pod isn't working, and why that one first?
2. What is a Helm chart?

**Intermediate:**
3. Why is `kubectl logs --previous` often more useful than `kubectl logs` for diagnosing CrashLoopBackOff?
4. Explain the diagnostic difference between `ImagePullBackOff` and `Pending` — what's the fastest way to tell which one you're actually looking at?

**Advanced:**
5. A Pod is stuck in `ContainerCreating`. List at least three genuinely different root causes, and how you'd distinguish between them.
6. Explain how `values.yaml` and an environment-specific override file work together to keep staging and production structurally identical while allowing them to differ meaningfully.

**Scenario:**
7. Walk through, from first principles using Chapter 10.1's methodology, how you'd diagnose "customers report checkout is failing intermittently" — with no other information given.

---

### Hands-On Lab 10.1 — Break ShopSphere on purpose, then package it with Helm

**Objective:** Practice real diagnosis against intentionally broken manifests, then convert the fixed version into a working Helm chart.

**Prerequisites:** A kind cluster; Helm installed; the manifests from Chapter 6.9.

**Steps:**

1. Deploy three intentionally broken variants, one at a time, and diagnose each using only `kubectl describe` and `kubectl logs` — resist looking at the "planted bug" until you've formed a hypothesis:
   - Variant A: an image tag that doesn't exist in your registry.
   - Variant B: a Service `selector` that doesn't match the Deployment's Pod labels.
   - Variant C: a container command that exits immediately (e.g., `command: ["false"]`).

2. For each, write down: the exact `STATUS`/Events output you saw, your hypothesis, and the actual root cause — then fix it and confirm recovery.

3. Convert the fixed Chapter 6.9 manifests into a Helm chart following the structure in Chapter 10.4, with `replicaCount`, `image.tag`, and `config.logLevel` all templated.

4. Install it twice, as two independent releases with different values, to prove template reuse:
   ```bash
   helm install shopsphere-dev ./shopsphere-backend -n shopsphere-dev --create-namespace --set replicaCount=1
   helm install shopsphere-staging ./shopsphere-backend -n shopsphere-staging --create-namespace --set replicaCount=2
   ```

**Expected result:** each broken variant produces a distinct, identifiable symptom you can name precisely; both Helm releases run independently, with different replica counts, from the exact same chart.

**Verification:** `helm list --all-namespaces` shows both releases; `kubectl get deployment -n shopsphere-dev` and `-n shopsphere-staging` show the different replica counts you specified, proving the same template correctly produced two different, valid outcomes.

**Troubleshooting:** if `helm install` fails with a template rendering error, run `helm template ./shopsphere-backend` locally first — it renders the final YAML without installing anything, which is by far the fastest way to catch a templating mistake.

**Cleanup:**
```bash
helm uninstall shopsphere-dev -n shopsphere-dev
helm uninstall shopsphere-staging -n shopsphere-staging
kubectl delete namespace shopsphere-dev shopsphere-staging
```

**Challenge:** add a `helm upgrade` step that changes `image.tag`, and use `helm rollback` to revert it — then explain, using Chapter 10.4's concepts, exactly what `helm rollback` actually changed under the hood.

---

*End of Part XI. Part XII moves into CI/CD with Jenkins — building the full pipeline that takes ShopSphere from a `git push` all the way to a deployed, smoke-tested Kubernetes release, automatically.*
