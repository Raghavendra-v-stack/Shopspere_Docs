# The ShopSphere DevOps Book
## Part XXII — The Final Capstone and Job-Readiness Checklist

---

### Before you begin

Everything in this closing Part assumes you've worked through Parts I–XXI, including their labs. This capstone is deliberately **not** a step-by-step tutorial. You've joined ShopSphere as a DevOps engineer, and this is your actual assignment. Business requirements are given; the implementation is yours to build, using everything this book has taught you. A complete reference solution follows afterward — but attempt it first. That struggle is where the actual learning happens.

---

## Chapter 21.1 — The Assignment

**The situation.** ShopSphere has outgrown its current setup and is preparing for its first major promotional sale in six weeks. Leadership has asked you to design and build the production infrastructure and deployment pipeline from scratch, on your own AWS account (as a stand-in for the company's real account, for this exercise), following the personal-lab-budget discipline from this book throughout.

### Business Requirements

- The storefront (frontend) and API (backend) must be reachable over HTTPS at a custom domain.
- The system must survive the failure of any single node or Availability Zone without customer-visible downtime.
- Order data must never be lost, even in a database failure — define and justify your actual RPO/RTO targets.
- The system must automatically handle a traffic surge of at least 5x normal load during the sale, and scale back down afterward.
- Every code change must be automatically tested, security-scanned, and deployed — no manual deployment steps in the final pipeline.
- No secret (database password, API key) may ever appear in source control, a Docker image, or plaintext Kubernetes configuration.
- The team must have real visibility into system health — metrics, logs, and alerting — before the sale, not discovered for the first time during it.
- Infrastructure must be fully defined as code, destroyable and reproducible, not manually clicked together.

### Security Requirements

- All workloads run as non-root, with dropped capabilities and read-only filesystems where feasible.
- Network access between components follows least privilege — the frontend should have no direct path to the database.
- All AWS access from within the cluster uses scoped, temporary credentials — no static keys stored anywhere.
- RBAC is scoped per role — an on-call engineer's read access should not equal a deploy pipeline's write access.

### Availability Requirements

- Minimum 3 backend replicas, spread across at least 2 Availability Zones, with no more than one replica per node.
- Database: justify your Multi-AZ decision explicitly, including the tradeoff you're accepting either way.
- A safe, budgeted autoscaling range for both Pods and nodes, sized for the expected 5x surge — with your reasoning for the specific numbers chosen.

### CI/CD Requirements

- Pipeline stages: test → build → scan → push → deploy → smoke test, with each stage a genuine gate.
- A documented, and ideally automated, rollback path.

### Monitoring Requirements

- Dashboards covering, at minimum, the Four Golden Signals for the backend.
- At least one alerting rule, tuned to avoid false-positive noise from brief, transient blips.
- A plan for where logs go, and for how long they're retained.

**Your task:** design the architecture, then build it — using Docker, Kubernetes, Helm, Jenkins, Terraform, and AWS, exactly as covered throughout this book. Stop reading here, and attempt it, before continuing to the reference solution below.

---

## Chapter 21.2 — Reference Solution: Architecture

```text
                              USERS
                                |
                                v
                            Route 53
                    (shop.example.com -> ALB)
                                |
                                v
                    AWS Application Load Balancer
              (provisioned by AWS Load Balancer Controller,
               TLS terminated here via ACM certificate)
                                |
                                v
                       Kubernetes Ingress
                     (host/path routing rules)
                                |
                                v
                      Kubernetes Services (ClusterIP)
                  +-------------+-------------+
                  |                           |
           Frontend Pods                 Backend Pods
        (3 replicas, 2+ AZs,          (3 replicas, 2+ AZs,
         anti-affinity)                anti-affinity, HPA
                                        3-15 replicas)
                                              |
                              +---------------+---------------+
                              |               |               |
                            Redis          Worker Pods      RDS PostgreSQL
                       (ElastiCache,     (Deployment,        (Multi-AZ,
                        or in-cluster     HPA via KEDA        automated
                        for lab)          on queue depth)     backups,
                                                               point-in-time
                                                               recovery)
```

**Justifications, as the assignment requires:**

- **RDS Multi-AZ: yes, for production.** The tradeoff accepted is cost (roughly double a single-instance RDS cost) in exchange for automatic failover on instance/AZ failure — justified given the explicit "no order data loss" requirement and the sale's high-stakes timing; for the personal lab version, a single-AZ instance is an acceptable, explicitly cost-conscious deviation, following this book's "don't fake production scale in the lab" principle from Part I.
- **RPO/RTO targets:** RPO of 5 minutes (met by RDS's continuous transaction-log-based point-in-time recovery, not just daily snapshots), RTO of 30 minutes (met by Multi-AZ automatic failover, which is typically much faster than 30 minutes, giving real margin).
- **Autoscaling range:** backend HPA `minReplicas: 3, maxReplicas: 15` (5x the baseline of 3, with headroom); Cluster Autoscaler/node group sized to comfortably support 15 backend replicas plus frontend and worker capacity, pre-emptively raised (per Chapter 16.4's capacity planning guidance) ahead of the sale rather than relying solely on reactive autoscaling from a cold start.

---

## Chapter 21.3 — Reference Solution: Implementation Map

This maps each requirement directly to the specific chapters and concepts that satisfy it — confirming, concretely, that this book's curriculum genuinely covers everything the assignment demands.

| Requirement | Satisfied by |
|---|---|
| HTTPS at custom domain | Ingress + AWS Load Balancer Controller + ACM certificate + Route 53 alias record (Chapters 7.2, 13.2) |
| Survive node/AZ failure | Pod anti-affinity across `topology.kubernetes.io/zone`, multi-AZ subnets, RDS Multi-AZ (Chapters 8.3, 12.2, 16.1) |
| No order data loss | RDS automated backups + point-in-time recovery, tested restore, explicit RPO/RTO (Chapters 7.3, 16.2) |
| 5x traffic surge handling | HPA + Cluster Autoscaler together, pre-emptively raised bounds ahead of the sale (Chapters 9.1, 13.5, 16.4) |
| No manual deploy steps | Full Jenkins pipeline: test → build (kaniko) → scan (trivy) → push (ECR) → deploy (Helm) → smoke test (Part XII) |
| No secrets in source/images | Kubernetes Secrets + IRSA + AWS Secrets Manager for genuinely sensitive values, never baked into images (Chapters 6.8, 9.3, 12.3, 13.4) |
| Least-privilege network access | NetworkPolicy restricting frontend from reaching the database directly; Security Groups as a second, independent layer (Chapters 9.3, 12.2) |
| Scoped AWS credentials in-cluster | IRSA for both the AWS Load Balancer Controller and any application needing AWS access (Chapter 13.4) |
| Scoped RBAC per role | Separate Role/RoleBinding for on-call read access vs. the CI/CD pipeline's deploy-scoped ServiceAccount (Chapter 9.3) |
| Metrics/logs/alerting | Prometheus + Grafana (Four Golden Signals dashboard), centralized logging via Loki or CloudWatch, Alertmanager with a sustained-duration threshold (Part XVI) |
| Infrastructure as Code | Terraform modules for VPC and EKS, remote state with locking (Part XV) |
| Non-root, dropped capabilities, read-only filesystem | SecurityContext on every Deployment (Chapter 9.3) |

---

## Chapter 21.4 — What a Strong Implementation Additionally Demonstrates

Beyond simply satisfying each checkbox, a genuinely strong solution to this capstone shows judgment in a few specific places worth calling out explicitly, since they're exactly what separates a junior implementation from a senior one:

- **The RDS Multi-AZ decision is justified, not just stated.** A weak solution says "I used Multi-AZ because it's more available." A strong solution names the specific cost tradeoff being accepted and ties it back to the explicit business requirement that justifies paying for it.
- **PodDisruptionBudgets are present**, protecting the backend during node maintenance/upgrades in the run-up to the sale — an easy requirement to miss, since it's not explicitly named in the assignment, but is a direct, necessary consequence of the "survive node failure" requirement combined with the reality that nodes sometimes need planned maintenance too (Chapter 16.3).
- **The alerting threshold has a sustained duration**, not a single-data-point trigger — directly avoiding the false-alarm churn problem named repeatedly since Chapter 8.2's probe-flapping caution.
- **A DR drill was actually performed**, not just configured — an actual timed restore, proving the stated RTO is realistic rather than theoretical (Chapter 16.2).
- **The pipeline's IAM role is scoped to deploy, not destroy** — reflecting the least-privilege reasoning from the AWS interview bank's Question 31, applied concretely rather than just recited.

---

## Chapter 21.5 — The Job-Readiness Checklist

You should be able to do every item below **without following a tutorial**. Work through this list honestly — it's the actual, practical test of everything this book set out to teach.

```text
DOCKER
[ ] Build a production-quality Dockerfile from scratch, unaided
[ ] Explain and apply multi-stage builds
[ ] Debug a container that crashes immediately on start
[ ] Configure custom networking between multiple containers
[ ] Configure volumes and explain volume vs. bind mount tradeoffs
[ ] Build a working multi-service Docker Compose file
[ ] Push and pull images to/from a private registry (ECR)
[ ] Harden an image: non-root, dropped capabilities, read-only filesystem
[ ] Explain what's actually happening at the kernel level (namespaces, cgroups)

KUBERNETES
[ ] Write a Deployment, Service, ConfigMap, and Secret from scratch
[ ] Configure Ingress with host- and path-based routing, and TLS
[ ] Configure ConfigMaps and Secrets, and explain Secrets' real limitations
[ ] Configure PersistentVolumeClaims and explain the PV/PVC/StorageClass chain
[ ] Configure liveness, readiness, and startup probes correctly and distinctly
[ ] Configure resource requests/limits and explain OOMKilled precisely
[ ] Configure HPA and explain why it needs the Metrics Server
[ ] Configure Pod anti-affinity and explain why replicas alone aren't HA
[ ] Troubleshoot CrashLoopBackOff, ImagePullBackOff, and Pending Pods independently
[ ] Troubleshoot a Service that isn't routing traffic correctly
[ ] Perform a rollback and explain what's actually happening underneath it
[ ] Package an application as a Helm chart with environment-specific values
[ ] Configure RBAC for a specific, scoped role
[ ] Configure a NetworkPolicy and verify it's actually being enforced

AWS
[ ] Explain VPC, subnets, and the public/private distinction, unaided
[ ] Explain IAM Roles vs. Users, and articulate why Roles are generally preferred
[ ] Push and manage images in ECR, including lifecycle cleanup
[ ] Deploy and tear down a real EKS cluster, including cost verification
[ ] Configure the AWS Load Balancer Controller and get a working ALB from Ingress
[ ] Configure IRSA for a workload needing AWS access
[ ] Explain the RDS Multi-AZ tradeoff and justify a real decision either way

CI/CD
[ ] Build a Jenkins pipeline from scratch: test, build, scan, push, deploy, smoke test
[ ] Explain why each stage is ordered the way it is
[ ] Explain the Docker-socket risk and articulate a safer CI build alternative
[ ] Wire up an automated or clearly documented rollback path

TERRAFORM
[ ] Write a reusable module, parameterized by variables
[ ] Explain state, and why remote state with locking matters for a team
[ ] Run a full plan -> apply -> destroy cycle, reviewing the plan output first
[ ] Explain a real scenario where manual and Terraform-managed infrastructure could drift

OBSERVABILITY AND OPERATIONS
[ ] Stand up Prometheus + Grafana and build a Four-Golden-Signals dashboard
[ ] Configure an alert with a sustained-duration threshold
[ ] Explain RPO/RTO and translate a business requirement into a concrete target
[ ] Perform and time an actual backup restore
[ ] Explain a blameless postmortem's purpose, not just its process
```

**A final honest note on this checklist.** If any single box gives you real pause, that's not a failure — it's a map of exactly where to go back and re-read, re-lab, and re-practice. The goal was never to memorize this book; it was to build the kind of understanding that lets you walk into a system you've never seen before — at ShopSphere, or anywhere else — and reason about it with genuine confidence, the same way you now can with everything covered across these twenty-two parts.

---

## Closing

You started this book with ShopSphere running as a handful of unmanaged processes on a single server, one engineer manually SSHing in and hoping nothing broke. You're finishing it with ShopSphere running as a secured, observable, autoscaling, multi-AZ production system, deployed automatically from every `git push`, on infrastructure that's entirely defined as code and can be rebuilt from scratch on demand.

Every technology in between was introduced because ShopSphere genuinely needed it, in the order a real engineering team would actually encounter that need — not as a disconnected tour of tools. That's the shape of the actual job. Good luck.

---

*End of Part XXII. End of The ShopSphere DevOps Book.*
