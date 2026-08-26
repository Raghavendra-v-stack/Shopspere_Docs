# The ShopSphere DevOps Book
## Part XX — Final Interview Bank: Kubernetes

---

## Kubernetes (80 Questions)

**1. What is Kubernetes, in one or two sentences?**
*Tests:* Baseline clarity.
*Strong:* A container orchestration platform that automates deployment, scaling, and lifecycle management of containerized applications across a cluster, using a declarative, desired-state model.
*Weak:* "It manages containers."

**2. What does it mean for Kubernetes to be declarative?**
*Tests:* The single most important conceptual question in this whole bank.
*Strong:* You declare the desired end state; Kubernetes continuously reconciles actual state toward it, automatically, including recovering from failures.
*Weak:* "You write YAML instead of commands" (surface-level, misses the reconciliation loop entirely).
*Senior take:* Connects this to Helm and Terraform as the same pattern applied at different layers.

**3. What is the smallest deployable unit in Kubernetes, and why isn't it a container?**
*Strong:* A Pod — because some workloads need tightly-coupled containers sharing network/storage, scheduled and managed as one atomic unit.
*Weak:* "A container" (factually wrong — a common but real mistake).

**4. Name the control plane components and what each does.**
*Strong:* API Server (front door/validation), etcd (state store), Scheduler (assigns Pods to nodes), Controller Manager (runs reconciliation loops), Cloud Controller Manager (cloud-provider integration).
*Weak:* Names one or two, vaguely.

**5. Why does every control plane component communicate only through the API Server?**
*Strong:* Provides one consistent, authenticated, validated entry point for all cluster state changes, keeping the system consistent and auditable.
*Weak:* "For organization."

**6. What is etcd, and why is it critical?**
*Strong:* A distributed key-value store holding all cluster state; losing it without backup means losing the entire record of the cluster.
*Weak:* "A database Kubernetes uses."

**7. What does the Scheduler actually do — and not do?**
*Strong:* Decides *which node* a Pod should run on, based on resource availability and constraints; it does not itself start the container.
*Weak:* "It runs the Pods" (incorrect — that's the kubelet's job).

**8. What is the kubelet's role?**
*Strong:* A per-node agent ensuring assigned Pods are actually running, by instructing the local container runtime, and reporting status back.
*Weak:* "It manages the node" (too vague).

**9. What does kube-proxy do?**
*Strong:* Maintains network rules on each node implementing Service routing/load-balancing to backing Pods.
*Weak:* "Handles networking" (doesn't name the specific job).

**10. Does Kubernetes require Docker specifically?**
*Strong:* No — it requires an OCI-compliant, CRI-compatible runtime (e.g., containerd), not Docker Engine specifically.
*Weak:* "Yes" (a common, outdated misconception).

**11. Walk through what happens when you run `kubectl create deployment`.**
*Strong:* API Server validates and writes to etcd → Deployment controller creates a ReplicaSet → ReplicaSet controller creates Pod(s) → Scheduler assigns a node → kubelet instructs the runtime to pull the image and start the container → kubelet reports status back.
*Weak:* "It creates the deployment and starts pods" (skips every internal step being tested).

**12. What's the relationship between Deployment, ReplicaSet, and Pod?**
*Strong:* Deployment manages ReplicaSets (adding rollout/rollback capability); ReplicaSet maintains a fixed number of identical Pods.
*Weak:* "They're all the same kind of thing."

**13. Why doesn't a ReplicaSet alone provide safe rolling updates?**
*Strong:* It only maintains a replica count of *identical* Pods — it has no concept of gradually replacing an old version with a new one.
*Weak:* "It's simpler than a Deployment" (true but doesn't answer *why* that's insufficient).

**14. What problem does a Service actually solve?**
*Strong:* Pod IPs change every time a Pod is recreated; a Service provides one stable address and load-balances to whichever Pods currently match its selector.
*Weak:* "It exposes Pods" (vague — doesn't name the actual problem: Pod IP instability).

**15. What's the difference between ClusterIP, NodePort, and LoadBalancer?**
*Strong:* ClusterIP is internal-only; NodePort additionally exposes a static port on every node; LoadBalancer additionally provisions a real cloud load balancer.
*Weak:* Lists them with no distinction.

**16. `kubectl get endpoints` for a Service is empty. What's the most likely cause?**
*Strong:* The Service's label selector doesn't match any existing Pod's labels — a very common typo-driven failure.
*Weak:* "The Service is broken" (no diagnosis given).

**17. Why might a Service have populated endpoints but still fail to route traffic correctly?**
*Strong:* The Pods might be Running but not Ready (failing readiness probes) — Kubernetes excludes NotReady Pods from Service traffic even if they technically exist.
*Weak:* "Networking issue" (too vague to be useful).

**18. What is a Namespace, and what does it not provide by default?**
*Strong:* A logical partition for object names, RBAC scope, and quotas; it does not by itself provide network isolation between Pods in different namespaces.
*Weak:* "A way to organize things" (misses the important "doesn't isolate network traffic" caveat).

**19. Are Kubernetes Secrets encrypted by default?**
*Strong:* No — only base64-encoded, which is trivially reversible; real protection requires etcd encryption at rest, tight RBAC, and often an external secrets manager.
*Weak:* "Yes" (a very common, important misconception to correct).

**20. Why use a ConfigMap instead of baking config into the image?**
*Strong:* Decouples config from the image, letting the exact same tested artifact be promoted across environments unchanged.
*Weak:* "It's more organized."

**21. What's the difference between a resource request and a limit?**
*Strong:* A request is the guaranteed minimum, used by the Scheduler for placement; a limit is the hard ceiling, enforced by the kernel via cgroups.
*Weak:* "They're both about resources" (no distinction).

**22. What happens when a container exceeds its CPU limit vs. its memory limit?**
*Strong:* CPU: throttled, keeps running slower. Memory: OOMKilled — the kernel can't throttle memory the way it throttles CPU, so it terminates the process outright.
*Weak:* "It gets killed" (correct for memory only, misses the CPU-throttling distinction, which is the actual point).

**23. What are QoS classes, and how does a Pod get "Guaranteed" status?**
*Strong:* Guaranteed (requests == limits for all containers), Burstable (requests set but not equal to limits), BestEffort (no requests/limits at all) — determines eviction priority under node pressure.
*Weak:* Names the tiers with no explanation of how they're assigned or why they matter.

**24. What's the difference between a liveness and a readiness probe?**
*Strong:* Failed liveness → container restarted; failed readiness → Pod pulled from Service traffic, no restart. They should generally check different things.
*Weak:* "They both check health" (misses the differing consequences, which is the entire point of the question).

**25. Why shouldn't a liveness probe check an external dependency like a database?**
*Strong:* A shared dependency blip would fail every replica's liveness probe simultaneously, causing a self-inflicted mass restart/outage — that check belongs in readiness instead.
*Weak:* "It's not a good practice" (no mechanism explained).

**26. What's a startup probe for?**
*Strong:* Delays liveness/readiness checks until a slow-starting container finishes booting, preventing premature liveness-triggered restarts.
*Weak:* "For slow apps" (vague — doesn't explain the actual mechanism/interaction with liveness).

**27. Explain node affinity vs. a toleration.**
*Strong:* Affinity is expressed by a Pod, pulling it toward certain nodes; a taint pushes Pods away from a node by default, and a toleration is a Pod's explicit permission to land there anyway.
*Weak:* "They're both scheduling features" (doesn't distinguish direction/mechanism).

**28. Why is Pod anti-affinity necessary even with multiple replicas already running?**
*Strong:* Without it, the Scheduler could place all replicas on the same node, meaning one node failure takes down every replica — defeating the purpose of redundancy.
*Weak:* "For better distribution" (doesn't connect it to the actual failure scenario being tested).

**29. What's the difference between requiredDuringScheduling and preferredDuringScheduling affinity?**
*Strong:* Required is a hard constraint — the Pod won't be scheduled without it; preferred is a soft, weighted preference that can be overridden if unmet.
*Weak:* "One is stronger" (imprecise).

**30. Why does an HPA sometimes fail to scale even under real load?**
*Strong:* Most commonly, the Metrics Server isn't installed or functioning — the HPA has no usage data to act on.
*Weak:* "It's misconfigured" (too vague to be a real diagnosis).

**31. What's the difference between the HPA and the Cluster Autoscaler?**
*Strong:* HPA scales the number of Pod replicas; Cluster Autoscaler scales the number of nodes — both are needed together for genuine elastic scaling.
*Weak:* "They both do autoscaling" (misses the crucial Pods-vs-Nodes distinction).

**32. New Pods from an HPA scale-up are stuck Pending. Why?**
*Strong:* Likely no available node capacity, and Cluster Autoscaler either isn't configured or hasn't provisioned new nodes yet.
*Weak:* "Not enough resources" (correct direction, but a strong answer names the Cluster Autoscaler gap specifically).

**33. What's the difference between VPA and HPA?**
*Strong:* VPA adjusts a container's resource requests/limits over time; HPA adjusts the number of replicas.
*Weak:* "They're both autoscalers" (no distinction).

**34. Explain a rolling update's maxSurge and maxUnavailable.**
*Strong:* maxSurge controls how many extra Pods beyond desired count are allowed during rollout; maxUnavailable controls how many can be unavailable at once.
*Weak:* "They control the rollout speed" (imprecise — doesn't define either term).

**35. How does a Kubernetes rollback actually work mechanically?**
*Strong:* Deployments retain previous ReplicaSets; a rollback scales a prior ReplicaSet back up and the current one down, using the same rolling mechanism in reverse.
*Weak:* "It reverts to the old version" (doesn't explain the underlying mechanism, which is the actual test).

**36. What's the difference between a rolling update and a canary deployment, in terms of risk?**
*Strong:* Rolling update briefly mixes old/new for everyone; canary deliberately limits new-version exposure to a small traffic percentage first, containing blast radius if something's wrong.
*Weak:* "Canary is safer" (true but no explanation of the actual mechanism/reasoning).

**37. What's a blue/green deployment, and its main tradeoff?**
*Strong:* Two full separate environments, instant cutover; avoids ever mixing versions in front of live traffic, at the cost of double resource capacity during the transition.
*Weak:* "Two versions running" (incomplete).

**38. What's the difference between authentication and authorization in Kubernetes?**
*Strong:* Authentication establishes identity (who); authorization (RBAC) determines what that identity is allowed to do.
*Weak:* "They're related security concepts" (doesn't distinguish them).

**39. Explain Role vs. ClusterRole.**
*Strong:* Role is namespace-scoped; ClusterRole is cluster-wide, or usable for non-namespaced resources like Nodes.
*Weak:* "One is bigger" (imprecise).

**40. What is a ServiceAccount, and why does it matter?**
*Strong:* An identity for a process running inside the cluster to authenticate to the API — distinct from a human user; least-privilege scoping of ServiceAccounts is a key security practice.
*Weak:* "An account for a service" (circular).

**41. What does SecurityContext control, and how does it relate to Docker security concepts?**
*Strong:* Pod/container-level security settings (non-root, read-only filesystem, dropped capabilities) — the Kubernetes-native home for exactly the Docker security hardening concepts (USER, --read-only, --cap-drop).
*Weak:* "Security settings for Pods" (too vague to show the connection being tested).

**42. What are Pod Security Standards, and what replaced them (or rather, what did they replace)?**
*Strong:* Three tiers (Privileged, Baseline, Restricted) enforceable at the namespace level, replacing the deprecated PodSecurityPolicy.
*Weak:* "Security policies for Pods."

**43. What does a NetworkPolicy actually control, and what's the default behavior without one?**
*Strong:* Which Pods can communicate with which others; by default, with no NetworkPolicy, all Pods can reach all other Pods across the entire cluster.
*Weak:* "Network rules" (misses the crucial "insecure by default" fact being tested).

**44. Why is it dangerous to assume NetworkPolicy is enforced on any given cluster?**
*Strong:* Enforcement depends entirely on the CNI plugin supporting it — some don't, and would silently accept the policy object without actually restricting anything.
*Weak:* "It might not work" (doesn't explain why).

**45. What is a PersistentVolumeClaim, and how does it relate to a PersistentVolume?**
*Strong:* A PVC is a request for storage; a PV is the actual provisioned storage that satisfies it — separating the "what I need" concern from the "how it's provisioned" concern.
*Weak:* "They're both storage-related objects."

**46. What does a StorageClass do?**
*Strong:* Defines how to dynamically provision storage automatically when a PVC requests it, rather than requiring manual PV creation.
*Weak:* "A class of storage" (circular).

**47. What's the difference between ReadWriteOnce and ReadWriteMany?**
*Strong:* RWO is mountable read-write by one node at a time (typical for EBS); RWX allows many nodes simultaneously (requires something like EFS).
*Weak:* "Different access levels" (no specifics).

**48. Would you run a production database as a Kubernetes StatefulSet, or use a managed service?**
*Strong:* Acknowledges the real tradeoff — StatefulSets can work but demand significant operational expertise (replication, backup, failover); a managed service like RDS is the pragmatic default for most teams.
*Weak:* A flat "always use Kubernetes" or "never use Kubernetes" with no nuance (misses that this is a genuine, debated tradeoff).

**49. What does an Ingress object do, and why does it need an Ingress Controller?**
*Strong:* Defines HTTP routing rules; the Ingress object alone does nothing without a controller (like NGINX or AWS Load Balancer Controller) actually implementing those rules.
*Weak:* "It routes external traffic" (doesn't mention the controller dependency, which is the real point).

**50. Why doesn't Kubernetes give every Service its own dedicated cloud load balancer by default?**
*Strong:* Cost and manageability — Ingress consolidates external routing behind a smaller number of load balancers using host/path rules.
*Weak:* "It's more efficient" (vague).

**51. Explain host-based vs. path-based Ingress routing.**
*Strong:* Host-based routes by the request's hostname (e.g., api.example.com vs. shop.example.com); path-based routes by URL path under a shared host.
*Weak:* "Different ways to route traffic" (no actual distinction given).

**52. What's the systematic first step when a Pod isn't working, and why that one first?**
*Strong:* `kubectl get pods` to check basic status, then `kubectl describe pod` for Events — work outward from the smallest, most fundamental layer before assuming a networking or higher-layer issue.
*Weak:* "Check the logs" (a reasonable second step, but not the methodologically correct first one being tested).

**53. Why is `kubectl logs --previous` often more useful than plain `kubectl logs` for CrashLoopBackOff?**
*Strong:* By the time you're looking, the container may be in a fresh crash cycle with empty current logs; `--previous` retrieves logs from the actual crash that just happened.
*Weak:* "It shows older logs" (technically true but doesn't explain why that matters here specifically).

**54. What's the difference between ImagePullBackOff and Pending, diagnostically?**
*Strong:* ImagePullBackOff means the Pod was scheduled but can't pull its image; Pending (with no container state at all) usually means it hasn't even been scheduled yet — `describe pod` Events distinguishes them immediately.
*Weak:* "They're both failure states" (doesn't distinguish the actual diagnostic signal, which is the point).

**55. Give at least three genuinely different root causes for a Pod stuck in ContainerCreating.**
*Strong:* A slow/stuck image pull, a volume that can't mount (including AZ mismatch), or a missing referenced ConfigMap/Secret.
*Weak:* Names only one cause, or says "it's still starting up" with no real diagnosis.

**56. What does OOMKilled mean precisely, and how do you distinguish a real leak from legitimately needing more memory?**
*Strong:* The kernel terminated the container for exceeding its memory limit; distinguishing requires `kubectl top`/Grafana trends over time — a steady climb suggests a leak, a flat-but-high usage suggests genuinely needing a higher limit.
*Weak:* "The container ran out of memory" (correct but incomplete — doesn't address the diagnostic follow-up, which is the real test).

**57. Why might a Deployment show a mix of old and new Pods for a while after an image update — is that a bug?**
*Strong:* No — that's the rolling update strategy working correctly, gradually replacing Pods according to maxSurge/maxUnavailable rather than all at once.
*Weak:* "Something's wrong with the deployment" (incorrectly assumes normal behavior is a bug).

**58. What is a Helm chart, and what problem does it solve?**
*Strong:* A templated, versioned package of Kubernetes manifests with a values file — solves the problem of duplicated, near-identical YAML across environments.
*Weak:* "A way to package Kubernetes stuff" (vague).

**59. What's the difference between `helm install` and `helm upgrade --install`?**
*Strong:* Plain `install` fails if the release already exists; `--install` on upgrade is idempotent — installs if new, upgrades if it already exists — important for repeatable CI/CD pipeline stages.
*Weak:* "They both deploy the chart" (misses the idempotency distinction being tested).

**60. What does `helm rollback` actually change under the hood?**
*Strong:* Reverts the release to a previous recorded revision's rendered manifests, applying them — conceptually similar to a Deployment rollback, but covering the whole chart's object set together.
*Weak:* "It undoes the last change" (imprecise about mechanism).

**61. Why might a cluster's NodeController show a node as Unknown rather than immediately rescheduling its Pods?**
*Strong:* A deliberate grace/eviction timeout gives the node a chance to recover before Kubernetes commits to treating it as truly failed, avoiding unnecessary churn from a brief blip.
*Weak:* "It's checking the node" (vague, doesn't name the actual mechanism).

**62. What is a DaemonSet, and what's a common real use case?**
*Strong:* Ensures exactly one Pod copy runs on every node; a classic use case is a log-shipping agent that needs to run everywhere logs are generated.
*Weak:* "A different kind of Deployment" (misses the "one per node" defining property).

**63. What's the difference between a Deployment and a StatefulSet, fundamentally?**
*Strong:* Deployment Pods are interchangeable; StatefulSet Pods have stable identity, stable per-replica storage, and ordered startup — needed when replicas aren't interchangeable, like a database.
*Weak:* "StatefulSet is for stateful apps" (correct but circular — doesn't explain the actual mechanism difference).

**64. What is a PodDisruptionBudget, and how does it differ from maxUnavailable in a rollout?**
*Strong:* PDB protects capacity during *voluntary infrastructure disruption* (like node drains); maxUnavailable protects capacity during a *deployment rollout* — different triggers, related safeguard concept.
*Weak:* "They're similar settings" (doesn't distinguish the trigger, which is the actual point).

**65. Why might `kubectl drain` hang indefinitely?**
*Strong:* Often a PodDisruptionBudget that can't be satisfied at all — e.g., minAvailable equal to or exceeding total replica count, making any voluntary eviction mathematically impossible.
*Weak:* "The node isn't responding" (doesn't identify the actual common root cause being tested).

**66. What is RBAC least privilege, and how would you verify a ServiceAccount's actual permissions?**
*Strong:* Grant only the minimum necessary permissions; verify with `kubectl auth can-i <verb> <resource> --as=<identity>`.
*Weak:* "Give minimum access" (doesn't mention the actual verification command, which is a meaningful part of a strong answer).

**67. Explain CoreDNS's role and the naming convention for a Service's fully qualified name.**
*Strong:* Provides in-cluster DNS resolution; a Service is reachable at `<service>.<namespace>.svc.cluster.local`, and by just `<service>` within the same namespace.
*Weak:* "It's DNS for Kubernetes" (doesn't name the actual convention being tested).

**68. Why can any Pod reach any other Pod's IP directly, cluster-wide, without NAT?**
*Strong:* A deliberate Kubernetes networking design requirement, implemented by the CNI plugin — flat, routable Pod networking regardless of which node either Pod is on.
*Weak:* "Kubernetes networking allows it" (circular).

**69. What's the practical implication of the AWS VPC CNI assigning Pods real VPC IPs?**
*Strong:* Every Pod consumes a real VPC address, so subnet sizing needs to account for far more addresses than just node count — a common real EKS capacity-planning gotcha.
*Weak:* "Pods get real IPs" (doesn't connect it to the practical sizing consequence being tested).

**70. What does IRSA let you do, and why is it preferable to static AWS credentials in a Pod?**
*Strong:* Lets a specific Kubernetes ServiceAccount assume a scoped IAM Role, giving Pods short-lived, auto-rotating AWS credentials instead of a long-lived static key stored somewhere.
*Weak:* "It gives Pods AWS access" (misses the least-privilege and credential-lifecycle reasoning being tested).

**71. What's the difference between admission control and RBAC?**
*Strong:* RBAC governs whether an identity is allowed to perform an action at all; admission control intercepts requests after authz, before persistence, to validate or mutate them against additional policy (e.g., Pod Security Standards, OPA Gatekeeper).
*Weak:* "They're both security features" (doesn't distinguish where each sits in the request lifecycle).

**72. A frontend Pod is compromised. What layers of defense should limit the damage?**
*Strong:* Non-root/dropped-capabilities SecurityContext, least-privilege RBAC on its ServiceAccount, NetworkPolicy restricting what it can reach, and resource limits preventing resource exhaustion — no single layer relied on alone.
*Weak:* Names only one layer (e.g., "NetworkPolicy would stop it") — misses the defense-in-depth framing being explicitly tested.

**73. Why is `terraform`/Helm/Kubernetes's shared "declarative reconciliation" pattern worth recognizing across all three?**
*Strong:* Demonstrates genuine architectural understanding rather than memorized tool-specific commands — the same observe/compare/act loop recurs at the infrastructure, packaging, and application-deployment layers.
*Weak:* Treats the three tools as unrelated (misses the explicitly-tested cross-cutting insight).

**74. What's the real difference between kind/minikube and a production EKS cluster?**
*Strong:* Conceptually identical Kubernetes API and objects; the difference is entirely in the control plane's operational model (self-managed/local vs. AWS-managed, multi-AZ) and the underlying infrastructure, not in how you use `kubectl`.
*Weak:* "kind is for testing, EKS is for production" (true but shallow — doesn't explain *what specifically* differs).

**75. Why does EKS charge a flat hourly fee for the control plane, and why does this matter operationally for a personal learning account?**
*Strong:* It's billed regardless of usage, meaning an idle cluster racks up real cost with zero traffic — a genuinely common personal-account surprise, making "destroy after each session" a real operational discipline, not just a suggestion.
*Weak:* "EKS costs money" (misses the specific always-on billing behavior being tested).

**76. What's the difference between the traditional Cluster Autoscaler and Karpenter, conceptually?**
*Strong:* Cluster Autoscaler works within pre-defined node groups' min/max ranges; Karpenter directly provisions right-sized nodes on demand based on actual Pending Pod requirements, without pre-defined fixed groups.
*Weak:* "Karpenter is newer" (true but doesn't explain the actual mechanism difference).

**77. Why does `volumeBindingMode: WaitForFirstConsumer` matter on EKS specifically?**
*Strong:* Prevents a PV from being provisioned in a different AZ than the Pod that will actually use it — directly solving a real, common storage-Pending failure mode.
*Weak:* "It controls when the volume is created" (technically true, doesn't connect it to the specific AZ-mismatch problem being tested).

**78. What's the danger of an overly broad ClusterRoleBinding, even in a learning/lab environment?**
*Strong:* Builds bad habits and masks permission errors during learning that would surface (correctly) in a properly least-privileged production environment — practicing with intentionally scoped RBAC is itself part of learning the skill.
*Weak:* "It's less secure" (true but doesn't address the specific learning-context angle the question is probing).

**79. Why would a team choose to keep production database access entirely outside the Kubernetes RBAC model (e.g., via IAM/RDS auth) rather than only through Kubernetes Secrets?**
*Strong:* Kubernetes Secrets alone (base64, not encrypted by default) aren't a sufficient control boundary for the most sensitive credentials; layering IAM-based database authentication provides an independent, stronger control outside Kubernetes's own trust boundary.
*Weak:* "Kubernetes Secrets aren't secure enough" (correct instinct, but a strong answer explains the layered/independent-boundary reasoning).

**80. Summarize, in under a minute, how a request from a customer's browser actually reaches a running container in ShopSphere's production architecture.**
*Strong:* DNS resolves the hostname to the ALB → ALB (provisioned via Ingress + AWS Load Balancer Controller) routes to the correct Service based on host/path rules → kube-proxy load-balances to a healthy, Ready Pod's IP → the container, running under its SecurityContext-constrained, resource-limited configuration, handles the request.
*Weak:* "It goes through the load balancer to the pods" (correct at a high level, but omits essentially every specific mechanism this entire book covered, which is exactly what a strong answer demonstrates).

---

*End of Part XX. Part XXI closes the Final Interview Bank with AWS, Jenkins/CI-CD, Terraform, and Production Troubleshooting Scenarios.*
