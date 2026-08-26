# The ShopSphere DevOps Book
## Part XIII — AWS Foundations

---

### Where we left off

Everything so far — Docker, Kubernetes, Helm, Jenkins — has run on your laptop or a local kind cluster. It's time to build the real infrastructure ShopSphere's production system will actually live on. Before EKS (Part XIV) makes any sense, we need the AWS networking and identity foundations it depends on.

---

## Chapter 12.1 — A Personal Lab Budget Strategy, Upfront

Before touching AWS at all, let's be direct about cost, exactly as promised at the start of this book.

**Resources you can practice cheaply or entirely locally**, and have already been doing so throughout this book: Docker, Kubernetes (via kind), Helm, Jenkins, and everything conceptual in this chapter (VPC design, IAM policy structure) can genuinely be learned and even hand-drawn/planned without AWS costing you anything.

**Resources that require real AWS usage, and real awareness of cost:** ECR (already covered — genuinely low-cost, Chapter 4.3), EKS (a real hourly control-plane charge, covered fully in Part XIV), NAT Gateways (charge per hour *and* per GB processed — one of the more commonly underestimated costs in a personal AWS learning account), Load Balancers, EBS/EFS volumes, and RDS.

**The strategy for the remainder of this book:** create → practice → destroy. We are not going to leave a running EKS cluster, NAT Gateway, or RDS instance sitting idle "just in case" between reading sessions. Every AWS-touching lab from here on ends with an explicit cleanup step and a "what could still be costing me money" check — treat this discipline itself as one of the real skills this book is teaching you, not a formality.

---

## Chapter 12.2 — VPC and Networking

### VPC

**Simple explanation:** a VPC is your own private, isolated slice of the AWS network — your own logically separated space to put servers, databases, and everything else, with full control over how it's internally structured and what can reach it from outside.

**Proper definition:** a **VPC (Virtual Private Cloud)** is a logically isolated virtual network within AWS, with its own private IP address range (recall CIDR notation from Part I), which you define and control.

**Why it exists.** This is the AWS-scale version of exactly the private-network concept from Part I — private IPs, routing, and controlled internet access — now applied to an entire cloud environment rather than one machine or one Docker bridge network.

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```

Recall from Part I: `/16` gives roughly 65,536 addresses — plenty of room for subnets, nodes, and Pods across ShopSphere's eventual EKS cluster.

### Subnets

**What they are.** A **subnet** is a subdivision of a VPC's address range, tied to a specific Availability Zone, used to logically (and physically) separate resources — most importantly, into **public** and **private** subnets.

```text
                   VPC: 10.0.0.0/16
     +---------------------------------------------+
     |                                                 |
     |   Public Subnet          Private Subnet          |
     |   10.0.1.0/24              10.0.101.0/24           |
     |   (AZ: us-east-1a)         (AZ: us-east-1a)          |
     |                                                 |
     |   Public Subnet          Private Subnet          |
     |   10.0.2.0/24              10.0.102.0/24           |
     |   (AZ: us-east-1b)         (AZ: us-east-1b)          |
     +---------------------------------------------+
```

**Public subnets** have a route to the internet via an **Internet Gateway**, and resources in them can have a public IP — appropriate for a load balancer, but *not* appropriate for a database or, generally, EKS worker nodes. **Private subnets** have no direct inbound route from the internet — this is where ShopSphere's EKS worker nodes and RDS database should live.

**Why spreading subnets across multiple Availability Zones matters.** Recall Chapter 8.3's Pod anti-affinity discussion — the exact same high-availability reasoning applies one layer up: if every subnet lived in a single AZ, an entire AZ outage would take the whole application down, regardless of how well the Kubernetes-level anti-affinity was configured.

### Route tables

A **route table** determines where network traffic from a subnet is directed — "traffic to this CIDR range goes this way." A public subnet's route table sends `0.0.0.0/0` (everything, i.e., "any destination not otherwise matched") traffic to the Internet Gateway; a private subnet's route table instead sends `0.0.0.0/0` traffic to a NAT Gateway (next).

### Internet Gateway

An **Internet Gateway** attaches to a VPC and provides the actual path for traffic between public subnets and the internet — the concrete AWS implementation of the "public IP reachable from the internet" concept from Part I.

### NAT Gateway

**Simple explanation:** a NAT Gateway lets resources in a private subnet — with no public IP of their own — still reach *out* to the internet (to pull a Docker image from ECR, for example), without allowing anything from the internet to reach *in* to them.

**Proper definition:** a **NAT Gateway** performs Network Address Translation (the exact concept introduced in Part I) for outbound-only internet access from a private subnet, sitting in a public subnet itself, with an Elastic IP.

```text
   Private Subnet                 Public Subnet
   EKS Worker Node                 NAT Gateway              Internet
   10.0.101.5   -------outbound------->  |  -------------->  (e.g. ECR)
        ^                                 |
        |                                 v
        +-------- response only ---------+
   (no inbound connection from the
    internet can ever reach 10.0.101.5
    directly, unlike the NAT Gateway itself)
```

**COST WARNING, stated exactly as this book promised it would be:** NAT Gateways are billed **per hour they exist, plus per GB of data processed through them** — this is one of the single most common sources of unexpected cost in a personal AWS learning account, precisely because it's easy to create one for a lab, forget about it, and have it silently accruing hourly charges for days. Do not create a NAT Gateway casually; understand this cost before doing so, and always tear it down when a lab session ends.

**Interview question (intermediate):** "Why would EKS worker nodes typically live in a private subnet rather than a public one?" — Worker nodes generally don't need to accept direct inbound connections from the internet at all — external traffic should arrive through a load balancer/Ingress instead (Part VIII) — so placing them in a private subnet, reachable outbound via a NAT Gateway but not directly reachable inbound from the internet, meaningfully reduces the attack surface, consistent with the defense-in-depth principle from Part X.

### Security Groups

**What they are.** A **Security Group** is a stateful, resource-attached firewall — recall the firewall concept from Part I, and specifically recall Chapter 3.1's Docker network-isolation-as-security-tool discussion and Chapter 9.3's NetworkPolicy: this is the same underlying principle (allow-list traffic by rule), now implemented at the AWS network layer, attached directly to specific resources (an EC2 instance, an RDS database, a load balancer) rather than to a subnet or a Kubernetes Pod.

```bash
aws ec2 create-security-group \
  --group-name shopsphere-db-sg \
  --description "Allow Postgres from backend nodes only" \
  --vpc-id vpc-xxxxxxxx

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxxxxx \
  --protocol tcp --port 5432 \
  --source-group sg-backend-nodes
```

This is, precisely, the AWS-layer equivalent of Chapter 9.3's NetworkPolicy example — "only the backend may reach the database, on port 5432, and nothing else" — expressed one layer further out in the stack. **Stateful** means a response to an allowed inbound request is automatically allowed back out, without needing a separate matching outbound rule — a genuinely practical distinction from Network ACLs (stateless, and applied at the subnet level rather than the resource level), which are a related but less commonly hand-tuned AWS networking concept worth knowing exists, though we won't build one out directly in this book.

**Why security groups matter as one more layer, not a replacement for NetworkPolicy.** This is worth being explicit about, because it directly extends Part X's defense-in-depth conclusion: a compromised backend Pod is constrained by Kubernetes RBAC, NetworkPolicy, *and* the Security Group governing the underlying EC2 worker node it happens to be running on — multiple independent layers, at different points in the stack, each one narrowing what's actually possible if an earlier layer is ever bypassed.

---

## Chapter 12.3 — IAM

### The core model

**Simple explanation:** IAM controls who — a person, or an AWS resource acting on someone's behalf — is allowed to do what, to which specific AWS resources.

**Proper definition:** **IAM (Identity and Access Management)** manages authentication (who are you) and authorization (what are you allowed to do) for AWS itself — the exact same two concepts distinguished carefully back in Chapter 9.3, now applied to AWS's own API rather than the Kubernetes API.

### Users, Roles, and Policies

- **IAM User** — a long-lived identity, typically representing a specific human (or, less ideally, a specific application using static, long-lived credentials — generally discouraged for anything that can use a Role instead, for exactly the reasons Chapter 4.3 flagged about ECR's short-lived tokens).
- **IAM Role** — an identity that can be **assumed temporarily** by a person or, critically for everything coming in Part XIV, by an AWS *service* or resource (an EC2 instance, an EKS Pod) — granting temporary, short-lived credentials rather than a permanent, static secret sitting somewhere waiting to leak.
- **IAM Policy** — a JSON document defining exactly which actions are allowed or denied, on which resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage"],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/shopsphere-backend"
    }
  ]
}
```

This policy grants exactly two narrow ECR read actions, on exactly one specific repository — nothing broader. This is **least privilege** (Chapter 9.3's principle), applied at the AWS identity layer.

**Why Roles are strongly preferred over long-lived User credentials for anything automated.** This is the same principle behind Chapter 4.3's `get-login-password` short-lived token, and behind Chapter 11.2's Jenkins Credentials store — long-lived static secrets sitting in a config file or environment variable are a standing risk (they can leak, and they don't expire on their own); a Role's temporary credentials are automatically time-limited and re-issued, dramatically shrinking the window in which a leaked credential is even useful to an attacker.

**Interview question (advanced):** "Why would you use an IAM Role instead of an IAM User with access keys for an application running on AWS infrastructure?" — A Role provides temporary, automatically-rotated credentials assumed dynamically at runtime, rather than a static, long-lived secret that has to be manually stored, rotated, and protected — significantly reducing the blast radius and lifetime of any credential exposure, and removing the operational burden of manual secret rotation entirely.

### Where this is heading: IRSA / Pod Identity

We flagged this back in Chapter 9.3 as a preview — here's the concept properly named. **IRSA (IAM Roles for Service Accounts)**, and its newer successor **EKS Pod Identity**, let a specific Kubernetes ServiceAccount (Chapter 9.3) assume a specific, narrowly-scoped IAM Role — meaning an individual Pod can be granted exactly the AWS permissions it needs (say, permission to read one specific S3 bucket, or one specific Secrets Manager secret), and nothing more, without AWS credentials ever being manually placed inside the container image or a Kubernetes Secret at all. This is precisely the "least privilege, extended all the way down to an individual Pod" idea, and we'll wire it up for real in Part XIV.

---

## Chapter 12.4 — The Rest of AWS, Named and Placed

A quick, honest map of where the remaining AWS services from your curriculum fit, and exactly which upcoming chapter covers each properly — so nothing feels unaddressed:

- **ECR** — already covered fully in Chapter 4.3.
- **EKS** — Part XIV, next.
- **Load Balancers (ALB)** — introduced conceptually in Chapter 7.2 (Ingress); provisioned for real via the AWS Load Balancer Controller in Part XIV.
- **EBS / EFS** — introduced conceptually in Chapter 7.3; used for real in Part XIV's storage lab.
- **Route 53** — AWS's managed DNS service; the real-world fix for the "Ingress failure" DNS incident in Chapter 10.3 — covered practically in Part XIV alongside the Ingress setup.
- **CloudWatch** — AWS's native monitoring and logging service; covered properly in Part XVI (Observability), alongside Prometheus and Grafana, with an honest comparison of when each is the right tool.

---

## Chapter 12.5 — Checkpoint

**Beginner:**
1. What's the difference between a public and a private subnet?
2. What does an IAM Policy actually define?

**Intermediate:**
3. Why does a NAT Gateway allow outbound internet access from a private subnet without allowing inbound access?
4. Why is an IAM Role generally preferred over an IAM User with static access keys for something automated, like a CI/CD pipeline or an application?

**Advanced:**
5. Explain how Security Groups relate to Kubernetes NetworkPolicy — are they redundant with each other, or complementary? Justify your answer.
6. Why does spreading subnets across multiple Availability Zones matter for the same underlying reason that Pod anti-affinity mattered in Part IX?

**Scenario:**
7. A NAT Gateway has been running in a personal AWS learning account for two weeks without anyone using it for anything. What's the actual financial impact of this, in general terms, and what would you check to confirm nothing else is similarly forgotten?

---

### Hands-On Lab 12.1 — Build the VPC foundation by hand, once, before Terraform

**Objective:** Understand the actual AWS CLI mechanics behind a VPC before Part XV automates all of it with Terraform — doing it manually once first makes the automated version make far more sense.

**Cost Warning:** the VPC, subnets, Internet Gateway, and route tables in this lab are free on their own. The NAT Gateway step specifically is **not** free — it bills hourly plus per-GB processed. This lab creates one briefly and tears it down immediately; do not leave it running.

**Steps:**

1. Create the VPC and two subnets:
   ```bash
   VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
   PUBLIC_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
     --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
   PRIVATE_SUBNET=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.101.0/24 \
     --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
   ```

2. Create and attach an Internet Gateway:
   ```bash
   IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
   aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
   ```

3. Create a NAT Gateway in the public subnet (briefly — this is the billable step):
   ```bash
   EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
   NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET \
     --allocation-id $EIP_ALLOC --query 'NatGateway.NatGatewayId' --output text)
   aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID
   ```

4. Confirm everything exists:
   ```bash
   aws ec2 describe-vpcs --vpc-ids $VPC_ID
   aws ec2 describe-nat-gateways --nat-gateway-ids $NAT_ID
   ```

**Expected result:** all four resources report as available/created; you can see, concretely, the exact objects Terraform will be creating on your behalf starting in Part XV.

**Verification:** `aws ec2 describe-nat-gateways` shows `State: available`.

**Cleanup — do this immediately after confirming step 4, not "later":**
```bash
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_ID
aws ec2 release-address --allocation-id $EIP_ALLOC
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-subnet --subnet-id $PUBLIC_SUBNET
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET
aws ec2 delete-vpc --vpc-id $VPC_ID
```

**What could still be costing me money?** An orphaned Elastic IP (EIPs are billed when *not* attached to anything — `release-address` above handles this, but it's worth explicitly verifying with `aws ec2 describe-addresses` afterward) and, obviously, a NAT Gateway you forgot to delete — `aws ec2 describe-nat-gateways` should show no gateways in `available` state when you're done.

**Challenge:** write out, in your own words (not code yet — that's Part XV), the route table entries a public subnet and a private subnet would each need, referencing the Internet Gateway and NAT Gateway respectively, based purely on Chapter 12.2's explanation.

---

*End of Part XIII. Part XIV deploys ShopSphere for real onto Amazon EKS — cluster architecture, managed node groups, the AWS Load Balancer Controller wiring up the Ingress from Part VIII, EBS/EFS CSI drivers, and IRSA/Pod Identity connecting Kubernetes ServiceAccounts to the IAM Roles introduced in this Part.*
