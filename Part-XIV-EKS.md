# The ShopSphere DevOps Book
## Part XIV — Amazon EKS

---

### Where we left off

You understand the AWS networking and identity foundations. Now we actually put ShopSphere's Kubernetes cluster on AWS — moving from kind on your laptop to a real, managed Kubernetes control plane.

```text
Local Docker
     ↓
Docker Compose
     ↓
Local Kubernetes (kind)
     ↓
AWS ECR
     ↓
AWS EKS
     ↓
Production architecture
```

---

## Chapter 13.1 — EKS Architecture

**Simple explanation:** EKS is Kubernetes, with AWS running and operating the control plane for you — you get the same `kubectl`, the same YAML, the same everything you've already learned, but you're no longer responsible for keeping etcd alive, patching the API Server, or any of the control-plane operational burden from Part VI.

**Proper definition:** **Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes service — AWS operates and scales the control plane (API Server, etcd, Scheduler, Controller Manager, all from Chapter 5.3) across multiple Availability Zones for you, while you manage the worker nodes (or use a managed option for those too, covered next).

**Why this directly answers Chapter 5.3's etcd concern.** Recall the specific warning from Part VI: etcd backups and control-plane operational overhead are a serious, real responsibility for self-managed Kubernetes. EKS's core value proposition is precisely removing that burden — you never `ssh` into an etcd node, never manually manage API Server upgrades, and AWS handles control-plane high availability across AZs by default.

```text
                    AWS-MANAGED                          YOU MANAGE
     +------------------------------+      +--------------------------------+
     |         EKS Control Plane      |      |         Worker Nodes             |
     |  (API Server, etcd, Scheduler,  |      |   (or AWS-managed via a         |
     |   Controller Manager — spread    |      |    Managed Node Group)           |
     |   across multiple AZs)            |      |                                   |
     +------------------------------+      +--------------------------------+
                    |                                        |
                    +-----------------  kubectl  -------------+
                             (same tool, same YAML,
                              everything from Parts VI-X
                              applies completely unchanged)
```

**Interview question (beginner):** "What's the actual difference between running Kubernetes yourself on EC2 versus using EKS?" — With self-managed Kubernetes on EC2, you're responsible for the entire control plane — API Server availability, etcd backups and consistency, version upgrades, and high availability across AZs. EKS removes that burden by having AWS operate and guarantee the control plane's availability, while you still manage (or partially manage, via managed node groups) the worker nodes running your actual workloads.

### Managed Node Groups

**What they are.** A **Managed Node Group** is AWS's way of provisioning and lifecycle-managing the EC2 instances that serve as your EKS worker nodes — handling instance provisioning, joining them to the cluster, and rolling updates/replacements, without you manually running `kubeadm join` or hand-managing individual EC2 instances the way self-managed Kubernetes would require.

```bash
eksctl create nodegroup \
  --cluster shopsphere-cluster \
  --name shopsphere-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

`--nodes-min`/`--nodes-max` here is exactly where the Cluster Autoscaler concept from Chapter 9.1 connects concretely to real infrastructure — the Cluster Autoscaler adjusts the actual node count within this range, automatically, in direct response to Pending Pods (Chapter 10.2) that need more capacity than currently exists.

### EKS networking: the AWS VPC CNI

Recall Chapter 7.1's mention that Pod-to-Pod networking is implemented by a pluggable CNI plugin. On EKS, that's typically the **AWS VPC CNI**, which has a genuinely distinctive behavior worth knowing: it assigns Pods real IP addresses directly from the VPC's own address space (Chapter 12.2), rather than a separate overlay network — meaning Pods are natively routable within the VPC, but also meaning your VPC's subnet sizing (recall CIDR from Part I) needs to account for a meaningfully larger number of addresses than you might initially expect, since every Pod consumes a real VPC IP, not just every node.

---

## Chapter 13.2 — The AWS Load Balancer Controller

Recall Chapter 7.2's Ingress chapter: Ingress objects need an Ingress Controller to actually do anything. On EKS, the standard choice is the **AWS Load Balancer Controller** — this is the concrete, real-world implementation of the Cloud Controller Manager pattern introduced back in Chapter 5.3.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=shopsphere-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Notice `serviceAccount.create=false` — this controller needs real AWS permissions (to actually create and manage an ALB via the AWS API), and the correct, secure way to grant that is exactly IRSA (Chapter 12.3) — a pre-created ServiceAccount bound to a scoped IAM Role, rather than broad, unscoped node-level permissions. We set this up properly in Chapter 13.4.

Once installed, the exact same Ingress YAML from Chapter 7.2 works unchanged — apply it, and the controller automatically provisions a real Application Load Balancer:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shopsphere-ingress
  namespace: shopsphere
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
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
```

This is genuinely satisfying to see, if you've followed the book this far: **every YAML concept from Part VIII is unchanged.** Only the `ingress.class` annotation and a couple of ALB-specific annotations differ from what you already wrote for the local NGINX-based Ingress Controller lab.

### Route 53

**Amazon Route 53** is AWS's managed DNS service — recall the "Ingress failure" incident from Chapter 10.3, where the actual root cause was a DNS record never being pointed at the load balancer. In production, you'd create a Route 53 record pointing `shop.example.com` at the ALB's DNS name (using an **alias record**, Route 53's AWS-integrated equivalent of a CNAME, which additionally supports pointing directly at the root domain, which a plain CNAME cannot).

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch file://route53-change.json
```

**Cost Warning:** Route 53 charges a small monthly fee per hosted zone, plus a small per-query charge — genuinely low cost for a personal lab, but not zero; and note that a **registered domain name** itself (if you don't already own one) is a separate, ongoing annual cost through a domain registrar, unrelated to Route 53's own hosting fee.

---

## Chapter 13.3 — Storage on EKS: EBS and EFS CSI Drivers

Recall Chapter 7.3's PV/PVC/StorageClass concepts — on EKS, dynamic provisioning requires installing the appropriate **CSI (Container Storage Interface) driver** for whichever AWS storage backend you want.

```bash
eksctl create addon --cluster shopsphere-cluster --name aws-ebs-csi-driver
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
```

**`volumeBindingMode: WaitForFirstConsumer` deserves a specific, important explanation, because it directly resolves the AZ-mismatch storage failure flagged back in Chapter 10.2.** With the default immediate binding mode, a PV could be provisioned in one AZ *before* Kubernetes knows which AZ the Pod will actually be scheduled into — leading exactly to that "PV bound in the wrong AZ" stuck-`Pending` failure. `WaitForFirstConsumer` delays provisioning until a Pod that will actually use the PVC is scheduled, so the volume is guaranteed to be created in the *same* AZ as that Pod. This single setting is a genuinely common, genuinely important real-world EKS storage gotcha to know by name.

**EFS**, for anything genuinely needing `ReadWriteMany` (Chapter 7.3) — shared file storage accessible from multiple nodes simultaneously — uses a separate CSI driver (`efs.csi.aws.com`) and works similarly, though EFS itself is priced differently (and generally costs more per GB than EBS, reflecting its different performance and sharing characteristics) — worth checking current pricing before provisioning any meaningful amount of it.

---

## Chapter 13.4 — IRSA: Connecting Kubernetes to IAM, For Real

This is the payoff of everything flagged since Chapter 9.3 and formalized in Chapter 12.3 — let's actually wire it up.

**The problem, precisely.** ShopSphere's backend might need to read a secret from AWS Secrets Manager (the "external secret manager" pattern flagged back in Part V and Part VII). It needs real AWS credentials to do that — but we specifically do **not** want to bake a long-lived AWS access key into the container image or a Kubernetes Secret (the exact anti-pattern flagged repeatedly since Part III).

**The solution: IRSA (IAM Roles for Service Accounts).** This lets a specific Kubernetes ServiceAccount (Chapter 9.3) assume a specific IAM Role (Chapter 12.3), with AWS automatically injecting short-lived, auto-rotating credentials into the Pod — no static secret, anywhere, ever.

```bash
eksctl create iamserviceaccount \
  --cluster shopsphere-cluster \
  --namespace shopsphere \
  --name shopsphere-backend-sa \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ShopSphereSecretsReadPolicy \
  --approve
```

This single command does several things worth naming explicitly: it creates the IAM Role, sets up the **OIDC trust relationship** that lets EKS Pods legitimately assume it (the specific mechanism underneath IRSA — worth knowing the name exists, without needing to memorize its full internals), attaches the specified least-privilege policy, and creates a matching Kubernetes ServiceAccount annotated to reference that Role.

```yaml
spec:
  template:
    spec:
      serviceAccountName: shopsphere-backend-sa
      containers:
        - name: backend
          # No AWS credentials anywhere in this spec —
          # the pod automatically receives short-lived credentials
          # scoped to exactly the ShopSphereSecretsReadPolicy permissions
```

**Pod Identity**, mentioned in Chapter 12.3, is a newer alternative to IRSA that simplifies this setup further (no OIDC provider configuration needed directly) — functionally solving the same problem, with a somewhat simpler setup path; worth knowing both names, since IRSA remains extremely widely deployed in existing production clusters even as Pod Identity gains adoption for new ones.

**Interview question (advanced, and a strong signal of real hands-on EKS experience):** "How would a Pod running on EKS securely access an AWS service like Secrets Manager, without static AWS credentials?" — Via IRSA (or the newer Pod Identity) — associating the Pod's Kubernetes ServiceAccount with a specific, narrowly-scoped IAM Role, so AWS automatically provides short-lived, auto-rotating credentials to the Pod at runtime, with no long-lived access key ever stored anywhere in the cluster or the image.

---

## Chapter 13.5 — Cluster Autoscaling on EKS

Recall Chapter 9.1's Cluster Autoscaler concept — on EKS specifically, you have two real options worth distinguishing:

**Cluster Autoscaler** — the traditional approach, working within a Managed Node Group's min/max range (exactly the `--nodes-min`/`--nodes-max` flags from Chapter 13.1), adding or removing whole EC2 instances of a pre-defined instance type as Pending Pods demand.

**Karpenter** — a newer, more flexible node-provisioning approach, which doesn't require pre-defining fixed node groups at all — instead, it directly analyzes the specific resource requirements of Pending Pods and provisions exactly-fitting EC2 instances (potentially different instance types for different workloads) on demand, often provisioning faster and more cost-efficiently than the traditional Cluster Autoscaler + fixed node group approach. Worth knowing this name as the modern direction the ecosystem has been moving, even if Cluster Autoscaler remains extremely common in existing production clusters.

---

## Chapter 13.6 — Personal Lab vs. Production, EKS Specifically

| Component | Production | Personal Lab |
|---|---|---|
| Control plane | EKS, Multi-AZ (AWS-managed regardless) | EKS — same control plane either way; this genuinely doesn't differ |
| Worker nodes | Managed Node Group, 3+ nodes, multiple AZs, autoscaling | Managed Node Group, 1–2 small instances (e.g., `t3.medium`), single AZ acceptable for learning |
| Load Balancer | AWS Load Balancer Controller, production ALB | Same — genuinely worth practicing for real; low relative cost |
| Database | Amazon RDS, Multi-AZ | PostgreSQL container in the cluster, or a small single-AZ RDS instance for practicing the real integration |
| Node count | Autoscaling, 3-10+ | Fixed at 1-2; destroy after the lab session |
| Uptime | Always on | Created → practiced → **destroyed** |

**COST WARNING, the single most important one in this entire book:** EKS charges a **flat hourly fee for the control plane itself**, regardless of whether you're actively using it — this is separate from, and in addition to, the cost of the worker node EC2 instances, any NAT Gateway, and any load balancer. **An idle EKS cluster left running for a week will cost real, non-trivial money, even with zero traffic and zero workloads deployed.** This is not a "maybe" — verify current EKS control-plane pricing before your first real cluster, and treat "did I delete my cluster" as a genuine, recurring checklist item, not an afterthought.

---

## Chapter 13.7 — Checkpoint

**Beginner:**
1. What does EKS manage for you that self-managed Kubernetes on EC2 wouldn't?
2. What is a Managed Node Group?

**Intermediate:**
3. Why does the AWS Load Balancer Controller need IRSA specifically, rather than broader node-level IAM permissions?
4. What problem does `volumeBindingMode: WaitForFirstConsumer` solve, and how does it connect to a failure mode covered in Part XI?

**Advanced:**
5. Explain, precisely, how IRSA lets a Pod access AWS services without ever storing a static AWS credential anywhere.
6. What's the practical difference between the traditional Cluster Autoscaler and Karpenter?

**Scenario:**
7. You created an EKS cluster to follow along with this Part, got interrupted, and didn't get back to it for eight days. Estimate, in general terms (without needing exact current pricing), the categories of cost that were likely accruing that whole time, even with nothing actively deployed.

---

### Hands-On Lab 13.1 — Deploy ShopSphere to a Real EKS Cluster

**Objective:** Take the Helm chart from Part XI and actually deploy it to a real, small EKS cluster, with a real ALB and IRSA — then tear the entire thing down completely.

**Cost Warning — read this before starting.** This lab creates a real EKS cluster (hourly control-plane charge), Managed Node Group EC2 instances (hourly), and an Application Load Balancer (hourly, plus data processed) — genuinely not free. Budget roughly one to two hours of AWS costs for this lab, verify current pricing for your region before starting, and follow the cleanup section immediately when finished — do not leave this running "to look at later."

**Prerequisites:** AWS CLI, `eksctl`, `kubectl`, `helm` installed; the VPC concepts from Part XIII (this lab uses `eksctl`'s own default VPC creation for simplicity, rather than the hand-built VPC from Part XIII's lab, to keep this lab focused specifically on EKS itself).

**Steps:**

1. Create a small, cost-conscious cluster:
   ```bash
   eksctl create cluster \
     --name shopsphere-cluster \
     --region us-east-1 \
     --nodegroup-name shopsphere-workers \
     --node-type t3.medium \
     --nodes 2 \
     --nodes-min 1 \
     --nodes-max 3 \
     --managed
   ```
   (This step alone typically takes 15-20 minutes — EKS control plane provisioning is not instant.)

2. Confirm cluster access:
   ```bash
   kubectl get nodes
   ```

3. Install the AWS Load Balancer Controller with IRSA, following Chapter 13.2 and 13.4.

4. Deploy ShopSphere's Helm chart from Part XI, using the ALB Ingress annotations from Chapter 13.2:
   ```bash
   helm upgrade --install shopsphere-prod ./shopsphere-backend \
     --namespace shopsphere --create-namespace \
     --set image.repository=<your-ecr-repo> \
     --set image.tag=v1
   kubectl apply -f alb-ingress.yaml
   ```

5. Confirm the ALB was actually provisioned, and reach it:
   ```bash
   kubectl get ingress -n shopsphere
   curl http://<the-alb-address-shown-above>/health
   ```

**Expected result:** nodes show `Ready`; the Ingress shows a real `*.elb.amazonaws.com` address; `curl` against it reaches ShopSphere's backend, genuinely running on real AWS infrastructure for the first time in this book.

**Verification:** `aws elbv2 describe-load-balancers` shows the real ALB that was provisioned on your behalf by the controller — direct proof of the Cloud Controller Manager pattern from Chapter 5.3 and 13.2 working end to end.

**Troubleshooting:** if the Ingress never gets an address, check the AWS Load Balancer Controller's own Pod logs (`kubectl logs -n kube-system deploy/aws-load-balancer-controller`) first — IRSA permission errors show up there explicitly and are the most common cause.

**Cleanup — do this immediately after finishing verification:**
```bash
kubectl delete ingress shopsphere-ingress -n shopsphere
kubectl delete namespace shopsphere
eksctl delete cluster --name shopsphere-cluster --region us-east-1
```
`eksctl delete cluster` tears down the control plane, node group, and the VPC/networking `eksctl` created for you — but always verify.

**What could still be costing me money?**
```bash
aws eks list-clusters
aws ec2 describe-instances --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=shopsphere-cluster"
aws elbv2 describe-load-balancers
aws ec2 describe-nat-gateways
aws ec2 describe-addresses
```
Every one of these should come back empty or show no ShopSphere-related resources. Genuinely run all five — this is not optional busywork; forgotten load balancers and NAT Gateways left over from a deleted cluster are a real, common source of surprise charges.

**Challenge:** before tearing anything down, deliberately trigger a Cluster Autoscaler scale-up by deploying enough replicas to exceed current node capacity, and watch a new EC2 instance actually join the cluster in real time via `kubectl get nodes --watch` — then scale back down and confirm the extra node is eventually removed automatically.

---

*End of Part XIV. Part XV covers Terraform — converting everything we just did manually with `eksctl` and the AWS CLI into version-controlled, repeatable Infrastructure as Code, including state management and the safe personal-lab plan/apply/destroy workflow.*
