# The ShopSphere DevOps Book
## Complete Table of Contents and Reading Guide

A complete, book-length Docker + Kubernetes + DevOps production guide, taught through one evolving company — **ShopSphere**, an e-commerce platform — from a single unmanaged server to a secured, autoscaling, multi-AZ production system on AWS, with full CI/CD, Infrastructure as Code, observability, and interview preparation.

---

### How the book is organized

Each Part below is a separate file. They're meant to be read in order — every Part assumes everything before it, and the ShopSphere application, infrastructure, and pipeline all evolve continuously across the whole book rather than resetting per chapter.

---

## Foundations and Containers

| Part | File | Covers |
|---|---|---|
| I | `Part-I-Foundations.md` | ShopSphere's origin story; Linux essentials; networking essentials |
| II | `Part-II-Container-Fundamentals.md` | Virtualization vs. containers; namespaces, cgroups, OverlayFS; OCI |
| III | `Part-III-Docker.md` | Docker architecture; full command set; building a real production Dockerfile |
| IV | `Part-IV-Networking-Storage-Compose.md` | Docker networking, volumes/bind mounts, Docker Compose |
| V | `Part-V-Security-Registries.md` | Docker security hardening; Docker Hub, private registries, AWS ECR |

## Kubernetes

| Part | File | Covers |
|---|---|---|
| VI | `Part-VI-Kubernetes-Fundamentals.md` | Why Kubernetes; control plane and worker node architecture |
| VII | `Part-VII-Core-Objects.md` | Pod, ReplicaSet, Deployment, Service, Namespace, ConfigMap, Secret |
| VIII | `Part-VIII-Networking-Ingress-Storage.md` | Cluster networking, CoreDNS, Ingress, PV/PVC/StorageClass |
| IX | `Part-IX-Resources-Probes-Scheduling.md` | Requests/limits, OOMKilled, liveness/readiness/startup probes, scheduling |
| X | `Part-X-Scaling-Releases-Security.md` | HPA/Cluster Autoscaler, release strategies, RBAC, NetworkPolicy |
| XI | `Part-XI-Troubleshooting-Helm.md` | Systematic troubleshooting methodology; Helm charts |

## CI/CD and Cloud Infrastructure

| Part | File | Covers |
|---|---|---|
| XII | `Part-XII-CICD-Jenkins.md` | Jenkins pipeline: test → build → scan → push → deploy → smoke test |
| XIII | `Part-XIII-AWS-Foundations.md` | VPC, subnets, NAT Gateway, Security Groups, IAM |
| XIV | `Part-XIV-EKS.md` | EKS architecture, Load Balancer Controller, EBS/EFS, IRSA |
| XV | `Part-XV-Terraform.md` | Infrastructure as Code, state, modules, plan/apply/destroy |

## Running It in Production

| Part | File | Covers |
|---|---|---|
| XVI | `Part-XVI-Observability.md` | Metrics (Prometheus/Grafana), logs, traces (OpenTelemetry) |
| XVII | `Part-XVII-Production-Operations.md` | High availability, backups/DR, upgrades, SLO/SLA/SLI |
| XVIII | `Part-XVIII-Incident-Bank.md` | All 10 real-world incident walkthroughs |

## Interview Preparation

| Part | File | Covers |
|---|---|---|
| XIX | `Part-XIX-Interview-Bank-Docker-Linux-Networking.md` | 55 Docker + 27 Linux + 26 Networking questions |
| XX | `Part-XX-Interview-Bank-Kubernetes.md` | 80 Kubernetes questions |
| XXI | `Part-XXI-Interview-Bank-AWS-Jenkins-Terraform-Scenarios.md` | 42 AWS + 26 Jenkins + 28 Terraform + 32 troubleshooting scenarios |

## Capstone

| Part | File | Covers |
|---|---|---|
| XXII | `Part-XXII-Capstone-and-Checklist.md` | Final capstone project; job-readiness checklist |

---

### The throughlines to watch for while reading

A few ideas deliberately recur across almost every Part, at a different layer each time — noticing them is genuinely one of the most valuable outcomes of reading this book straight through rather than piecemeal:

- **Declarative, desired-state reconciliation** — appears in Kubernetes controllers (Part VI), Helm releases (Part XI), and Terraform (Part XV).
- **"Give it a stable name because the address changes"** — DNS (Part I), Docker Compose service names (Part IV), Kubernetes Services and CoreDNS (Part VII–VIII).
- **Least privilege, enforced at every layer** — non-root containers (Part V), Kubernetes RBAC/SecurityContext (Part X), IAM/IRSA (Part XIII–XIV).
- **Defense in depth, never one layer alone** — Docker security (Part V), NetworkPolicy + Security Groups together (Part X, XIII), the compromised-Pod interview question (Part XX).
- **Fail fast, gate every stage** — Dockerfile layer caching (Part III), CI/CD pipeline stages (Part XII), Terraform's plan-before-apply (Part XV).
- **Local → AWS → Production, cost-consciously** — introduced in the original spec, applied concretely from Part V's ECR lab through Part XIV's EKS lab, with every AWS-touching lab carrying its own Cost Warning and Cleanup section.

---

### Suggested pacing

This is a genuinely large curriculum. A reasonable pace for someone with some software development background, studying part-time:

- **Weeks 1–2:** Parts I–V (Foundations through Docker Security) — do every lab.
- **Weeks 3–5:** Parts VI–XI (all of Kubernetes) — this is the largest, most important stretch; don't rush it.
- **Weeks 6–7:** Parts XII–XV (CI/CD, AWS, EKS, Terraform) — budget real AWS lab time and cost-awareness here specifically.
- **Week 8:** Parts XVI–XVIII (Observability, Production Ops, Incidents).
- **Week 9:** Parts XIX–XXI (Interview Bank) — review actively, don't just read; cover answers and test yourself.
- **Week 10:** Part XXII — attempt the capstone cold before reading the reference solution.

---

*This index file has no independent content of its own — it exists purely to help you navigate between the 22 parts. Start with Part I.*
