# The ShopSphere DevOps Book
## Part XVII — Production Operations

---

### Where we left off

ShopSphere is deployed, automated, secured, and observable. This Part is about the operational discipline that turns "we built a working system" into "we can run this system reliably, indefinitely, without it slowly degrading or catastrophically failing." This is arguably the least glamorous part of the job, and arguably the most important.

---

## Chapter 16.1 — High Availability, Properly Defined

**Simple explanation:** high availability means the system keeps working even when individual pieces of it fail — not "it never fails," but "no single failure takes the whole thing down."

**Proper definition:** **High Availability (HA)** is a design property where a system continues operating, without unacceptable disruption, despite the failure of individual components — achieved through redundancy and the elimination of single points of failure.

**Tracing HA all the way back through this book, layer by layer — this is worth doing explicitly, because it shows HA isn't one feature, it's the cumulative effect of many individual decisions made throughout:**

- **Application layer:** multiple Deployment replicas (Chapter 6.4), so one crashed Pod doesn't take down the service.
- **Node layer:** Pod anti-affinity (Chapter 8.3), so those replicas aren't all on the same node.
- **Availability Zone layer:** anti-affinity with `topology.kubernetes.io/zone`, plus multi-AZ subnets (Chapter 12.2), so an entire AZ outage doesn't take down every replica.
- **Control plane layer:** EKS's own multi-AZ control plane (Chapter 13.1) — AWS's responsibility, not yours, but worth knowing it's there.
- **Database layer:** RDS Multi-AZ (introduced properly below), so a database instance failure doesn't mean data loss or extended downtime.
- **Traffic layer:** a load balancer (Chapter 7.2) that automatically stops routing to unhealthy targets.

**High availability is not a checkbox you enable — it's the sum of redundancy decisions made at every single layer of the stack**, and a gap at any one layer can undermine all the careful redundancy built into the others (recall Chapter 8.3's exact incident: three replicas, zero real HA, because anti-affinity was missing).

### Multi-AZ, specifically for RDS

Recall Chapter 7.3's recommendation to use Amazon RDS for ShopSphere's production database. **RDS Multi-AZ** maintains a synchronously-replicated standby copy of the database in a second Availability Zone, and automatically fails over to it if the primary becomes unavailable — a managed, largely-automatic implementation of exactly the redundancy principle above, applied to the one component (the database) that's hardest to make redundant correctly yourself, which is precisely why Chapter 7.3 recommended a managed service for it in the first place.

**Interview question (intermediate):** "You have 3 backend replicas spread across 3 AZs, but your database is a single RDS instance with no Multi-AZ. Is your application actually highly available?" — No — the database is a single point of failure regardless of how well the stateless application tier is redundant; an AZ outage or instance failure affecting that single RDS instance would take down the entire application even with a perfectly redundant backend, illustrating that HA has to be evaluated end-to-end, not layer by layer in isolation.

---

## Chapter 16.2 — Backups and Disaster Recovery

### The distinction, precisely

**A backup** protects against data loss. **Disaster recovery (DR)** is the broader plan for restoring *service* after a major failure — which might involve backups, but also involves infrastructure recreation, failover procedures, and defined recovery targets.

### RPO and RTO

Two terms worth knowing precisely, because they're how DR requirements actually get specified and discussed:

- **RPO (Recovery Point Objective)** — how much data loss is acceptable, measured in time. "RPO of 1 hour" means: in the worst case, you might lose up to the last hour of data.
- **RTO (Recovery Time Objective)** — how long the system is allowed to be down before it's restored. "RTO of 4 hours" means: recovery must complete within 4 hours of the incident.

**Why these numbers actually matter, concretely.** They directly determine the backup strategy: an RPO of 1 hour requires backups (or continuous replication) at least every hour; an RPO of 5 minutes requires something closer to continuous replication, not periodic snapshots. A tighter RTO/RPO is more expensive and operationally complex to achieve — this is a genuine business tradeoff, not just a technical one, and a real DevOps engineer should be able to translate "the business needs 15-minute RPO" into "here's what that actually costs and requires."

### Backing up RDS

```bash
aws rds describe-db-snapshots --db-instance-identifier shopsphere-db
```

RDS supports automated daily snapshots plus point-in-time recovery (restoring to any specific moment within the retention window, using transaction logs between snapshots) — genuinely important to know as a distinct, more precise capability than "restore from yesterday's snapshot."

### Backing up etcd

Recall Chapter 5.3's warning: etcd is the cluster's entire source of truth. On EKS, this is AWS's responsibility (part of what you're paying the control-plane fee for), but it's worth knowing this remains a critical, real concern for anyone running self-managed Kubernetes — `etcdctl snapshot save` is the actual command, and losing etcd without a recent snapshot means losing the record of your entire cluster's state.

### Disaster recovery drills

**A genuinely important, often-skipped practice:** a backup you've never tested restoring is not a verified backup — it's a hope. Real DR planning includes periodically actually performing a restore, in a non-production environment, and confirming it works and measuring how long it actually takes against your stated RTO. This is exactly the same discipline as Part XI's troubleshooting labs — proving something works, rather than assuming it does because the configuration looks correct on paper.

**Interview question (advanced):** "Your company sets an RPO of 15 minutes for the production database. What does that actually require, and how would you verify it's being met?" — An RPO of 15 minutes rules out simple daily snapshots alone — it requires continuous or near-continuous replication (RDS's transaction-log-based point-in-time recovery genuinely supports this), and verifying it means actually performing a test restore to a specific point in time and confirming the data loss window is genuinely within 15 minutes, not just trusting that the underlying feature is theoretically capable of it.

---

## Chapter 16.3 — Rolling Upgrades and Cluster Maintenance

### Node maintenance and draining

**The problem.** A node sometimes needs maintenance — an OS patch, a planned replacement. Simply terminating it outright would abruptly kill every Pod running on it, with no graceful handling.

**`kubectl drain`:**

```bash
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
```

**What this actually does, precisely.** It cordons the node (marks it unschedulable, so no *new* Pods land there — `kubectl cordon` does this step alone, without evicting anything), then evicts existing Pods gracefully — respecting **PodDisruptionBudgets** (next) and normal graceful termination — so replacement Pods can be scheduled elsewhere *before* the node is actually taken down, rather than after.

### PodDisruptionBudget (PDB)

**What it is.** A **PodDisruptionBudget** limits how many Pods of a given application can be voluntarily disrupted (drained, evicted) at once — protecting against a well-intentioned maintenance operation accidentally taking down too much capacity simultaneously.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: shopsphere-backend-pdb
  namespace: shopsphere
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: shopsphere-backend
```

With `minAvailable: 2` and 3 replicas total, `kubectl drain` will evict at most 1 backend Pod at a time from any node being drained — waiting for a replacement to become Ready before evicting the next, rather than potentially evicting 2 or 3 simultaneously across multiple nodes being drained at once and dropping the application below safe capacity. **This is the direct operational-maintenance counterpart to Chapter 9.2's `maxUnavailable`** — that setting protects capacity during a *deployment*; a PDB protects capacity during *infrastructure* maintenance, a genuinely different trigger, using a conceptually related safeguard.

### Kubernetes version upgrades

EKS control-plane upgrades are largely AWS-managed (recall Chapter 13.1), but worker node upgrades — moving Pods off old-version nodes onto new ones — follow exactly the drain pattern above, typically automated by rolling a Managed Node Group (Chapter 13.1) through a new launch template, one node at a time, letting `drain` and PDBs protect availability throughout.

**A genuinely important practical note:** Kubernetes has a defined API deprecation policy — APIs get deprecated and eventually removed across versions (recall Chapter 6.1's `apiVersion` field) — meaning an upgrade can break manifests still referencing a removed API version. Checking manifests against the target version's deprecation notes *before* upgrading (tools like `kubectl-convert` or `pluto` can help scan for this) is a real, standard pre-upgrade step, not an edge case.

---

## Chapter 16.4 — Capacity Planning

**Simple explanation:** capacity planning means having a reasonable, evidence-based answer to "will we have enough resources for the traffic we expect," before that traffic actually arrives and finds out the hard way.

**What it actually draws on, connecting directly to earlier chapters:** historical metrics from Part XVI's Prometheus/Grafana setup (actual observed CPU/memory usage patterns, actual observed traffic growth over time) inform realistic resource requests/limits (Chapter 8.1) and realistic HPA/Cluster Autoscaler bounds (Chapter 9.1/13.5) — capacity planning isn't a separate discipline bolted on top of everything else in this book, it's the practice of actually *using* the observability data this book built to make informed infrastructure decisions, rather than guessing.

**A concrete, realistic ShopSphere example:** before a known, planned promotional sale, a responsible team reviews Grafana dashboards from the previous comparable sale, calculates the actual traffic multiplier observed, and pre-emptively raises the HPA's `maxReplicas` and the node group's `max_size` (Chapter 13.1) to comfortably accommodate it — rather than trusting autoscaling alone to react fast enough from a cold start, which it may not, depending on how quickly new nodes can actually be provisioned relative to how fast the traffic spike itself arrives.

---

## Chapter 16.5 — Incident Response

### A basic incident response structure

- **Detection** — an alert fires (Chapter 15.2), or a customer/monitoring system reports an issue.
- **Triage** — assess severity and impact; is this affecting all customers, or a subset? Is data at risk?
- **Mitigation** — stop the bleeding first, even before fully understanding root cause — a rollback (Chapter 9.2/10.4) is very often the fastest, safest first move, precisely *because* it's a well-understood, low-risk operation you've already practiced.
- **Resolution** — the actual underlying fix, once mitigated and there's room to investigate properly rather than under acute pressure.
- **Postmortem** — a blameless review of what happened, why, and what changes (technical or process) would prevent recurrence.

**Why "mitigate first, root-cause later" is the right default, not a shortcut.** Recall the entire structure of Chapter 10.3's incident walkthroughs — in every one, a fast rollback or fix restored service *before* the full root cause was necessarily understood in complete depth. Under real incident pressure, restoring service quickly, safely, and reversibly should almost always take priority over a slower, more thorough root-cause investigation that could be done properly afterward, with the pressure off.

### Blameless postmortems

**Why "blameless" specifically, and why this matters practically, not just culturally.** A postmortem culture that looks for a person to blame teaches engineers to hide mistakes, obscure timelines, and avoid flagging near-misses — which directly *reduces* the organization's ability to actually learn and prevent recurrence. A blameless postmortem instead treats an incident as a signal that the *system* (technical, or process) allowed a mistake to become an outage, and focuses entirely on what would prevent that class of incident happening again, regardless of who was involved.

**Interview question (intermediate, and it rewards a genuinely thoughtful answer over a purely technical one):** "Why do many engineering organizations insist on blameless postmortems?" — Blame-focused reviews discourage honest disclosure of mistakes and near-misses, which reduces the organization's ability to actually learn from incidents; blameless postmortems focus on systemic and process factors that allowed an incident to occur, producing genuinely actionable prevention measures rather than discouraging the openness needed to identify them in the first place.

---

## Chapter 16.6 — SLO, SLA, and SLI

These three terms are used constantly in production operations discussions and are frequently confused with each other — worth being precise.

- **SLI (Service Level Indicator)** — an actual, measured metric of some aspect of service behavior — for example, "the percentage of requests that complete successfully in under 300ms."
- **SLO (Service Level Objective)** — an internal *target* for an SLI — "99.9% of requests should complete successfully in under 300ms, measured over a rolling 30 days."
- **SLA (Service Level Agreement)** — an external, often contractual, *promise* to customers — typically a looser target than the internal SLO, with defined consequences (service credits, for example) if it's not met.

**Why the SLO is deliberately stricter than the SLA.** This is a genuinely important, commonly-tested nuance: if your SLA promises 99.9% to customers, your internal SLO target should be tighter than that — say, 99.95% — so that normal, expected operational variance doesn't routinely put you at risk of actually breaching the customer-facing contractual promise. The SLO is your own internal early-warning threshold, deliberately set with margin against the SLA.

### Error budgets

**Simple explanation:** if your SLO allows 0.1% of requests to fail, that 0.1% is your **error budget** — a deliberately allowed amount of imperfection, which exists specifically so the team doesn't over-invest in reliability at the expense of ever shipping anything new.

**Why this connects directly back to Chapter 9.2's release strategies.** A genuinely useful, common practice: if the error budget for a given period is already exhausted (too many incidents/errors already occurred), a team might deliberately slow down or pause risky releases (favoring, for instance, more cautious canary rollouts over full rolling updates) until the budget resets — a concrete, practical link between the reliability numbers this chapter defines and the actual day-to-day release decisions covered back in Part X.

**Interview question (advanced):** "What is an error budget, and how might it actually change a team's day-to-day engineering decisions?" — It's the amount of unreliability an SLO permits before breaching target — if a team has already spent most of its error budget for the period (through incidents or elevated error rates), it's a legitimate, data-driven signal to prioritize stability work and favor lower-risk release strategies over shipping new, riskier features, until the budget resets; conversely, an unspent error budget is itself a signal that the team may be over-investing in caution relative to their actual reliability target.

---

## Chapter 17.7 — Checkpoint

**Beginner:**
1. What's the difference between a backup and a disaster recovery plan?
2. What does `kubectl drain` actually do, step by step?

**Intermediate:**
3. Explain RPO and RTO, and why a tighter RPO generally costs more to achieve.
4. Why should an internal SLO typically be stricter than the external SLA?

**Advanced:**
5. Trace high availability through every layer of ShopSphere's stack covered in this book — where could a single remaining point of failure still exist, even after everything covered in Chapter 16.1?
6. What is an error budget, and how does it connect concretely to release strategy decisions from Part X?

**Scenario:**
7. During a node upgrade, `kubectl drain` on one node is unexpectedly blocking, unable to proceed. Using this chapter's concepts alone, what's a plausible cause, and how would you investigate it?

---

### Hands-On Lab 17.1 — Practice a Safe Node Drain and a Real Restore

**Objective:** Configure a PodDisruptionBudget, safely drain a node without dropping capacity, and perform (and time) an actual database restore.

**Prerequisites:** A kind cluster (multi-node — `kind create cluster --config` with a multi-node config) or your EKS lab cluster from Part XIV; ShopSphere's backend Deployment with 3+ replicas.

**Steps:**

1. Apply the PodDisruptionBudget from Chapter 16.3.

2. In one terminal, continuously curl the backend Service in a loop, logging failures.

3. In another terminal, drain a node hosting at least one backend replica:
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```

4. Confirm zero (or minimal, PDB-bounded) failed requests in your curl loop throughout the drain.

5. Uncordon the node afterward:
   ```bash
   kubectl uncordon <node-name>
   ```

6. Separately, if you have an RDS instance from earlier labs (or a local PostgreSQL container with a volume): create a snapshot/backup, deliberately modify or delete some data, then restore from the backup and time exactly how long the full restore takes.

**Expected result:** the curl loop shows continuous (or PDB-bounded, brief) availability throughout the drain, proving the PDB and graceful eviction worked together correctly; the restore step recovers the deliberately-deleted data, with a concrete, measured restore time you can compare against a hypothetical RTO target.

**Verification:** compare your curl loop's failure count against `minAvailable` in the PDB — it should never drop below what the PDB guarantees.

**Troubleshooting:** if `drain` hangs indefinitely, check for a PDB that can't be satisfied at all (for example, `minAvailable` set equal to or greater than the total replica count, making any voluntary eviction mathematically impossible) — this is a genuinely common, self-inflicted PDB misconfiguration.

**Cleanup:**
```bash
kubectl delete pdb shopsphere-backend-pdb -n shopsphere
```

**Challenge:** deliberately misconfigure the PDB with `minAvailable` equal to your replica count, attempt a drain, and confirm it blocks — then explain, in your own words, exactly why that specific configuration makes any voluntary disruption impossible.

---

*End of Part XVII. Part XVIII collects the remaining real-world incident scenarios from this book's original list into one focused troubleshooting drill set, immediately followed by the Final Interview Bank — organized by domain, with 270+ questions in the question / what's-being-tested / strong-answer / weak-answer / follow-up format promised at the start of this book.*
