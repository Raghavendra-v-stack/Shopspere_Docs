# The ShopSphere DevOps Book
## Part XVIII — The Real-World Incident Bank

---

### How to use this Part

This collects all ten of the incident scenarios promised at the start of this book, in one place, as a dedicated troubleshooting drill set. Four were already walked through fully in Part XI (Chapter 10.3) — they're referenced here rather than repeated, so this Part reads as one complete, non-redundant set of ten. For each, try to reason through **symptoms → what you'd check → diagnosis → root cause → fix → prevention** yourself before reading the walkthrough — that's the actual skill being built.

---

## Incident 1 — "The website is down."

**Symptoms:** Customers report the storefront is completely unreachable. No page loads at all.

**What would you check?** Work outward from Chapter 10.1's methodology, starting from the *outside* this time, since that's where the report originated: is DNS resolving at all? Is the load balancer/Ingress reachable? Then work inward toward Pods.

```bash
dig shop.example.com
curl -v https://shop.example.com
kubectl get ingress -n shopsphere
kubectl get pods -n shopsphere
```

**Diagnosis:** `dig` resolves fine; `curl` times out entirely (not even an HTTP error — a connection-level failure); `kubectl get pods` shows all backend and frontend Pods `Running` and `Ready`.

**Root cause:** the AWS Load Balancer Controller (Chapter 13.2) had silently failed to reconcile after an unrelated IAM policy change narrowed its IRSA (Chapter 13.4) permissions — the ALB itself had stopped receiving target-group health updates, and AWS had marked all targets unhealthy, taking the ALB itself out of service even though every Pod behind it was completely healthy.

**Fix:** restore the IRSA policy's required permissions; the controller reconciled and re-registered healthy targets within minutes.

**Prevention:** this is a strong, real argument for Part XVI's monitoring covering the *infrastructure* layer, not just the application layer — an alert on ALB target health specifically (not just backend Pod readiness) would have caught this independently of, and likely faster than, application-level readiness probes, since the actual failure was entirely outside the application.

---

## Incident 2 — "New deployment keeps crashing."

Covered fully in Part XI, Chapter 10.3. Summary: `kubectl logs --previous` is always the first move for CrashLoopBackOff; that incident's specific root cause was a missing `envFrom` ConfigMap reference lost during a manifest edit.

---

## Incident 3 — "Pods are stuck in Pending."

Covered fully in Part XI, Chapter 10.3. Summary: `kubectl describe pod` Events named `Insufficient cpu`; root cause was HPA scaling Pods with no matching Cluster Autoscaler configured to add node capacity — the "Scaling Pods vs. Scaling Nodes" gap from Chapter 9.1.

---

## Incident 4 — "Database connection suddenly stopped working."

**Symptoms:** Backend Pods are Running and Ready, but application logs show repeated `connection refused` or `timeout` errors specifically for database calls.

**What would you check?** Whether this is a Kubernetes-side problem or a database-side problem — work outward from the backend toward the database, testing connectivity independently of the application code itself.

```bash
kubectl exec -n shopsphere deploy/shopsphere-backend -- nc -zv <db-host> 5432
kubectl get networkpolicy -n shopsphere
aws rds describe-db-instances --db-instance-identifier shopsphere-db
```

**Diagnosis:** the raw TCP connectivity check itself times out — this rules out an application-level bug and points squarely at network or database-availability layers; `aws rds describe-db-instances` shows the instance status as `storage-full`.

**Root cause:** the RDS instance ran out of allocated storage — a slow, gradual growth in order history data that crossed the allocated threshold, at which point RDS stops accepting write connections to protect itself.

**Fix:** increase the RDS instance's allocated storage (a straightforward, low-risk operation RDS supports online, without downtime); confirm connections recover.

**Prevention:** enable **RDS Storage Autoscaling** so allocated storage grows automatically ahead of this threshold, and add a CloudWatch (or Prometheus, via the `postgres_exporter`) alert on free storage space specifically — trending toward exhaustion is a classic slow-burn incident that a good capacity-planning dashboard (Chapter 16.4) should have surfaced days in advance, not as a sudden outage.

---

## Incident 5 — "CPU usage has increased dramatically."

**Symptoms:** Grafana shows backend CPU usage climbing sharply over the last hour, with no corresponding deploy or obvious traffic spike in the request-rate metric.

**What would you check?** Whether this correlates with traffic (a legitimate, healthy scaling trigger) or is disconnected from traffic (a red flag for a code-level issue — an inefficient query, an infinite retry loop, a runaway background job).

```
# In Grafana / PromQL:
rate(http_requests_total[5m])       # is traffic actually up?
container_cpu_usage_seconds_total   # confirm which specific Pods are affected
```

```bash
kubectl top pods -n shopsphere
kubectl logs -n shopsphere <high-cpu-pod> | tail -100
```

**Diagnosis:** request rate is flat — this CPU spike is *not* traffic-driven; logs show a specific backend replica repeatedly retrying a call to an external payment-processing API that's returning errors, with no backoff between retries.

**Root cause:** a recently deployed retry-handling code path had no exponential backoff or retry limit — a downstream vendor outage turned into a self-inflicted CPU/traffic storm against that vendor, and against ShopSphere's own CPU budget.

**Fix:** roll back the offending deploy (Chapter 9.2) immediately as the fast mitigation; the proper fix is adding exponential backoff and a maximum retry count before rolling forward again.

**Prevention:** this is a direct, concrete example of Part XVI's "traces would show exactly where time/CPU is going" argument — a code review checklist item requiring backoff/limits on any external call, and an HPA (Chapter 9.1) that would otherwise have masked this by simply scaling out more Pods to absorb the runaway retries, at real, avoidable cost, rather than actually fixing anything.

---

## Incident 6 — "Pods are being OOMKilled."

Mechanically covered fully in Chapter 8.1 and referenced again in Chapter 10.2 — the summary diagnosis chain: `kubectl describe pod` → `Last State: OOMKilled` → `kubectl top pod` over time to distinguish "genuinely needs more memory" (raise the limit) from "this is a real leak" (needs actual code investigation, ideally backed by a memory profiler and, per Part XVI, a Grafana panel showing the leak's shape over time — a slow, steady climb rather than a spike is the classic leak signature).

---

## Incident 7 — "Application works inside the Pod but not from the internet."

Covered fully in Part XI, Chapter 10.3. Summary: root cause was a public DNS record never pointed at the load balancer — a pure Part I-level DNS gap, entirely outside the cluster, and a strong illustration of why the troubleshooting chain (Chapter 10.1) must work all the way out to DNS, not stop at Ingress/Service internals.

---

## Incident 8 — "New Docker image cannot be pulled."

**Symptoms:** a fresh deployment shows Pods stuck in `ImagePullBackOff`, immediately after a Jenkins pipeline reported a successful build and push.

**What would you check?** Whether the image genuinely exists at the exact tag referenced, and whether the Pod's pull credentials are valid — recall Chapter 10.2's root-cause list for this exact failure mode.

```bash
kubectl describe pod <pod-name> -n shopsphere | grep -A5 Events
aws ecr describe-images --repository-name shopsphere-backend --image-ids imageTag=<tag>
kubectl get sa <service-account> -n shopsphere -o yaml
```

**Diagnosis:** the Events output shows `401 Unauthorized` specifically, not `not found` — ruling out a missing-image or typo problem and pointing at authentication; `aws ecr describe-images` confirms the image genuinely exists.

**Root cause:** the node's IAM role (or the relevant IRSA-based pull mechanism) had an ECR authentication token that had expired, combined with a misconfigured token-refresh interval — a real, specific EKS/ECR integration gotcha distinct from the more common "wrong tag" root cause.

**Fix:** correct the token refresh configuration; manually trigger a fresh pull to confirm recovery.

**Prevention:** an automated post-deploy smoke test (Chapter 11.3) checking that a *fresh* Pod can actually pull and start — not just that already-running Pods remain healthy — would catch this class of issue before it silently blocks the *next* deploy or scale-up event, rather than only being discovered then.

---

## Incident 9 — "Deployment succeeded but users still see the old version."

Covered fully in Part XI, Chapter 10.3. Summary: root cause was CDN/browser caching, entirely outside Kubernetes — a strong lesson that not every "deployment" symptom is actually a Kubernetes problem, and that the troubleshooting chain must include the full request path, not just the cluster.

---

## Incident 10 — "One Kubernetes node has failed."

**Symptoms:** monitoring shows one EKS worker node transitioned to `NotReady` and has stayed that way for several minutes; some Pods that were on it show as `Unknown` or are being rescheduled.

**What would you check?** Whether this is being handled automatically (the expected, healthy outcome, per Chapter 5.3's Node controller) or whether something is preventing proper recovery.

```bash
kubectl get nodes
kubectl describe node <failed-node>
kubectl get pods -n shopsphere -o wide --field-selector spec.nodeName=<failed-node>
```

**Diagnosis:** `kubectl describe node` shows the kubelet stopped reporting heartbeats several minutes ago (an underlying EC2 instance-level failure, confirmed separately via `aws ec2 describe-instance-status`); the Node controller's default **pod eviction timeout** hadn't yet elapsed, so affected Pods were still shown as `Unknown` rather than already rescheduled — this is expected, intentional behavior, not a bug, giving the node a defined grace period to recover before Kubernetes commits to treating it as truly gone.

**Root cause:** an underlying EC2 hardware/hypervisor issue (visible in the AWS instance status checks) — genuinely outside Kubernetes's control, and not something to "fix" at the Kubernetes layer at all.

**Fix:** once the eviction timeout elapses, Kubernetes automatically reschedules the affected Pods onto healthy nodes (assuming sufficient capacity — connecting directly back to Incident 3/Chapter 9.1's Cluster Autoscaler); if the node doesn't recover on its own, the Managed Node Group (Chapter 13.1) can be configured to automatically replace unhealthy instances, or an engineer can manually terminate the instance to accelerate replacement.

**Prevention:** this incident, more than any other in this bank, is the direct payoff of Chapter 8.3's Pod anti-affinity and Chapter 16.1's multi-AZ design — with those in place, a single node failure is a non-event from the *application's* perspective (traffic simply continues serving from the remaining, correctly-spread replicas) even while the infrastructure-level recovery plays out over the following minutes.

---

## Chapter 18.1 — What This Bank Should Leave You With

Notice the shape across all ten: roughly half of these incidents (1, 4, 8, 10) had root causes **entirely outside the application code** — IAM/IRSA, RDS storage, ECR token refresh, and raw hardware failure. This is a genuinely important, realistic lesson to close this Part with: **a large fraction of real production incidents are infrastructure and configuration problems, not application bugs** — which is exactly why this book spent as much time as it did on Docker internals, Kubernetes architecture, AWS networking, and Terraform, rather than treating "the app" as the only thing worth understanding deeply. The troubleshooting methodology from Chapter 10.1 works precisely because it doesn't assume where the fault lies — it works outward, layer by layer, letting the evidence lead.

---

*End of Part XVIII. Part XIX begins the Final Interview Bank — starting with Docker, Linux, and Networking.*
