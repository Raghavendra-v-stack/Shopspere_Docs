# The ShopSphere DevOps Book
## Part XV — Terraform

---

### Where we left off

You manually built ShopSphere's VPC (Part XIII) and EKS cluster (Part XIV) using the AWS CLI and `eksctl`. This worked — but it's not repeatable, not reviewable, and not safe: if you needed to recreate this exact environment for a second engineer, or for a disaster recovery scenario, you'd be retyping commands from memory, hoping you remembered every flag correctly. This Part fixes that permanently.

---

## Chapter 14.1 — Why Terraform, Precisely

**Simple explanation:** Terraform lets you describe your infrastructure — the VPC, the EKS cluster, everything — as code, the same way a Kubernetes Deployment YAML describes your desired application state. You declare what you want; Terraform figures out how to make the real AWS environment match it.

**Proper definition:** **Terraform** is an **Infrastructure as Code (IaC)** tool that uses a declarative configuration language (HCL) to define cloud resources, and manages their full lifecycle — creation, modification, and destruction — by comparing your declared configuration against a tracked record of real-world state.

**This is Chapter 5.1's declarative, desired-state model, one more time, at the infrastructure layer.** You've now seen this exact pattern three times across this book: Kubernetes reconciling actual Pods against a desired replica count; Helm reconciling a set of Kubernetes objects against a chart's values; and now Terraform reconciling real AWS resources against your `.tf` configuration. Recognizing this as the *same idea*, applied at three different layers of the stack, is genuinely one of the most valuable patterns to walk into an interview with.

**Why this specifically fixes Part XIII and XIV's manual process.** Every `aws ec2 create-vpc`, every `eksctl create cluster` flag, becomes a reviewable, version-controlled file. A second engineer can read the `.tf` files and understand exactly what infrastructure exists, without needing to reverse-engineer it from the AWS Console or trust someone's memory of what commands were run in what order.

---

## Chapter 14.2 — Terraform Core Concepts

### Providers

**What it is.** A **provider** is a plugin that lets Terraform manage a specific platform's resources — AWS, Kubernetes, Helm, and many others.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Resources

**What it is.** A **resource** block declares one specific piece of infrastructure you want to exist.

```hcl
resource "aws_vpc" "shopsphere" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "shopsphere-vpc"
  }
}
```

Compare this directly to Chapter 12.5's manual `aws ec2 create-vpc --cidr-block 10.0.0.0/16` — same underlying action, but now declared, named (`shopsphere`, referenceable elsewhere in the configuration), and permanently recorded in version control rather than existing only in your shell history.

### Variables

**What it is.** A **variable** parameterizes your configuration, so the same `.tf` files can be reused across environments (recall Helm's `values.yaml` from Chapter 10.4 — this is the identical idea, one layer further out).

```hcl
variable "environment" {
  description = "Deployment environment name"
  type        = string
  default     = "lab"
}

variable "node_instance_type" {
  type    = string
  default = "t3.medium"
}
```

```hcl
resource "aws_vpc" "shopsphere" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name        = "shopsphere-vpc-${var.environment}"
    Environment = var.environment
  }
}
```

### Outputs

**What it is.** An **output** exposes a value from your Terraform configuration — for use by another Terraform configuration, a CI/CD pipeline (recall Part XII — Jenkins could read a Terraform output to know which cluster to deploy to), or just for your own visibility.

```hcl
output "vpc_id" {
  value = aws_vpc.shopsphere.id
}

output "eks_cluster_endpoint" {
  value = aws_eks_cluster.shopsphere.endpoint
}
```

### State

**Simple explanation:** Terraform state is Terraform's own record of what it actually created last time — this is precisely what lets it figure out, on the next run, what's changed and needs updating, versus what already matches and can be left alone.

**Proper definition:** **Terraform state** (`terraform.tfstate`) is a JSON file mapping your declared configuration to the real, actual IDs of the resources Terraform has created — it is Terraform's own "actual state" record, in the exact desired-state-vs-actual-state sense from Chapter 5.1, except here *Terraform itself* is both tracking the actual state and reconciling it, rather than a continuously-running control loop like Kubernetes's.

**A critical, genuinely important operational fact:** state should almost never be edited by hand, and losing it is a serious problem — without it, Terraform no longer knows which real AWS resources correspond to which parts of your configuration, and can't safely reconcile them.

### Remote state and state locking

**Why local state alone doesn't scale to a team.** If state lives only as a local file on your laptop, a second engineer running Terraform has no way to know what you've already created — they'd either recreate duplicate resources or, worse, have their own local state disagree with reality entirely.

**Remote state** stores this file centrally instead — commonly in an S3 bucket for a team working on AWS infrastructure — so everyone's Terraform runs read and write the same, single, shared source of truth.

```hcl
terraform {
  backend "s3" {
    bucket         = "shopsphere-terraform-state"
    key            = "eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "shopsphere-terraform-locks"
    encrypt        = true
  }
}
```

**State locking**, using the DynamoDB table referenced above, prevents two people (or two CI pipeline runs) from applying changes to the same infrastructure *simultaneously* — without it, two concurrent `terraform apply` runs could corrupt the state file or create conflicting, inconsistent real-world changes. This is a genuinely important, commonly-tested concept: state locking is what makes Terraform safe for a team to use concurrently at all.

**Interview question (advanced):** "Why is remote state with locking considered essential for any team using Terraform beyond a single person experimenting alone?" — Local state has no shared source of truth, so multiple people risk creating duplicate or conflicting resources; remote state (e.g., in S3) gives everyone the same view of what's actually deployed, and locking (e.g., via DynamoDB) prevents two concurrent applies from corrupting that shared state or making conflicting real-world changes at the same time.

### Modules

**What it is.** A **module** is a reusable, self-contained package of Terraform configuration — the direct Terraform equivalent of a Helm chart (Chapter 10.4): write the VPC configuration once, as a module, and reuse it across `dev`, `staging`, and `production`, each with different input variables.

```hcl
module "vpc" {
  source   = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  environment = var.environment
}

module "eks" {
  source     = "./modules/eks"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

Notice `module.vpc.vpc_id` — this is one module's output feeding directly into another module's input, letting Terraform automatically understand the dependency between them (it will always create the VPC before the EKS cluster that needs it, without you needing to specify that ordering explicitly).

### Plan, Apply, Destroy

```bash
terraform init      # download providers, initialize backend
terraform plan       # show exactly what would change, without changing anything
terraform apply       # actually make the changes
terraform destroy      # tear everything down
```

**`terraform plan` deserves special emphasis, because it's genuinely one of Terraform's most valuable safety properties.** It shows you a precise diff — resources to be created, changed, or destroyed — *before* anything actually happens. This is a direct, practical safety net against exactly the kind of "oops, I didn't realize that command would also delete the database" mistake that a purely imperative CLI-command workflow (Part XIII/XIV's manual approach) offers no protection against at all. **A disciplined Terraform workflow never runs `apply` without having read the preceding `plan` output carefully.**

### Workspaces and environment strategy

**Terraform workspaces** let you maintain multiple, separate state files from the same configuration — useful for lightweight environment separation. However, worth an honest, practical note: many real teams prefer **separate directories or separate root configurations per environment** (rather than workspaces) for genuinely distinct environments like `staging` and `production`, specifically because workspaces share the same underlying configuration files, and a mistake made while intending to target one workspace can too easily land on another. This isn't a hard rule — reasonable teams differ here — but it's worth knowing this is a real, debated tradeoff rather than a fully settled question, and being able to articulate both sides.

---

## Chapter 14.3 — ShopSphere's Infrastructure, in Terraform

Let's convert Part XIII and XIV's manual work into real modules.

```text
shopsphere-infra/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── eks/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**`modules/vpc/main.tf`** — converting Chapter 12.2's manual VPC, subnets, Internet Gateway, and NAT Gateway into code:

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  tags = { Name = "shopsphere-vpc-${var.environment}" }
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "shopsphere-public-${count.index}" }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]
  tags = { Name = "shopsphere-private-${count.index}" }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
}

resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "this" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags = { Name = "shopsphere-nat" }
}
```

Notice `count = length(var.public_subnet_cidrs)` — this creates one subnet *per entry* in a list variable, rather than hand-writing a separate resource block for each AZ, directly solving the exact repetition problem Terraform (and Helm, and every other templating tool in this book) exists to eliminate.

**`modules/eks/main.tf`** — converting Chapter 13.1's `eksctl create cluster`:

```hcl
resource "aws_eks_cluster" "this" {
  name     = "shopsphere-cluster-${var.environment}"
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}

resource "aws_eks_node_group" "workers" {
  cluster_name    = aws_eks_cluster.this.name
  node_group_name = "shopsphere-workers"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = var.subnet_ids

  scaling_config {
    desired_size = var.desired_node_count
    min_size     = var.min_node_count
    max_size     = var.max_node_count
  }

  instance_types = [var.node_instance_type]
}
```

Every value here — `desired_node_count`, `min_node_count`, `node_instance_type` — is a variable, meaning the *exact same module* produces ShopSphere's small, cost-conscious personal lab cluster with one set of input values, and a full, larger production cluster with another — precisely the "Local vs. Production, side by side" table promised throughout this book, now expressed as literal Terraform input differences rather than prose.

```hcl
# environments/lab/terraform.tfvars
environment        = "lab"
desired_node_count = 1
min_node_count     = 1
max_node_count     = 2
node_instance_type = "t3.medium"

# environments/production/terraform.tfvars
environment        = "production"
desired_node_count = 3
min_node_count      = 3
max_node_count      = 10
node_instance_type  = "m5.large"
```

```bash
terraform plan -var-file="environments/lab/terraform.tfvars"
terraform apply -var-file="environments/lab/terraform.tfvars"
```

---

## Chapter 14.4 — The Personal Lab Terraform Workflow, Explicitly

This is where Terraform pays off enormously for exactly the cost-conscious personal lab approach this book has emphasized throughout.

```bash
# Start of a study session:
terraform apply -var-file="environments/lab/terraform.tfvars"

# ... practice EKS, deploy ShopSphere, work through labs ...

# End of the study session — completely, reliably torn down:
terraform destroy -var-file="environments/lab/terraform.tfvars"
```

**Why this is genuinely better than Part XIV's manual `eksctl delete cluster` cleanup, for the cost-awareness reasons this book has stressed throughout:** `terraform destroy` tears down *everything* Terraform created — VPC, subnets, NAT Gateway, EKS cluster, node groups, IAM roles — as a single, complete, reliable operation, driven directly from the state file's record of everything that actually exists. There's no risk of manually forgetting one specific resource (a lingering Elastic IP, an orphaned NAT Gateway) the way there genuinely is with a manual, multi-step CLI cleanup checklist. This doesn't remove the need for the verification habit from Chapter 13.6's "what could still be costing me money" checks — always still verify — but it meaningfully reduces the *risk* of human error in cleanup, which is exactly the kind of repetitive, error-prone manual process this entire book has worked to eliminate, applied now to teardown instead of just creation.

**Interview question (beginner, but genuinely worth being able to answer confidently):** "Why is `terraform destroy` generally safer for full environment cleanup than manually deleting resources one at a time through the AWS CLI or Console?" — Terraform's state file tracks every single resource it created; `destroy` reliably removes all of them together in dependency order, eliminating the real risk of a human manually forgetting one specific resource (like an Elastic IP or a NAT Gateway) during a manual, multi-step cleanup process.

---

## Chapter 14.5 — Checkpoint

**Beginner:**
1. What does `terraform plan` do, and why is it good practice to always review it before `apply`?
2. What is Terraform state, in your own words?

**Intermediate:**
3. Why does remote state with locking matter for a team, specifically, in a way it doesn't for a single person working alone?
4. Explain how a Terraform module is conceptually similar to a Helm chart.

**Advanced:**
5. Trace the "declarative, desired-state" pattern across Kubernetes, Helm, and Terraform — in your own words, what's genuinely the same about all three, and what's genuinely different?
6. Why might a team choose separate directories per environment instead of Terraform workspaces, even though workspaces are a built-in feature specifically designed for this?

**Scenario:**
7. A teammate manually deleted an EKS node group directly through the AWS Console, without going through Terraform. What happens the next time someone runs `terraform plan`, and why?

---

### Hands-On Lab 14.1 — Rebuild Part XIII and XIV's Infrastructure as Code

**Objective:** Recreate the VPC and EKS cluster from Parts XIII and XIV entirely through Terraform, proving the full plan → apply → destroy workflow end to end.

**Cost Warning:** identical to Part XIV's lab — this creates real, billable EKS and networking resources. Budget accordingly, and do not skip the destroy step.

**Prerequisites:** Terraform installed; an S3 bucket and DynamoDB table created ahead of time for remote state (or start with local state for this first exercise, and add remote state as the Challenge below).

**Steps:**

1. Write the `vpc` and `eks` modules following Chapter 14.3's structure (or adapt the snippets above directly).

2. Create `environments/lab/terraform.tfvars` with small, cost-conscious values.

3. Initialize and review the plan carefully before applying anything:
   ```bash
   terraform init
   terraform plan -var-file="environments/lab/terraform.tfvars"
   ```
   Read the plan output in full — confirm it matches what you expect to be created, and only then proceed.

4. Apply:
   ```bash
   terraform apply -var-file="environments/lab/terraform.tfvars"
   ```

5. Confirm the cluster works exactly as it did in Part XIV's manual lab:
   ```bash
   aws eks update-kubeconfig --name shopsphere-cluster-lab --region us-east-1
   kubectl get nodes
   ```

**Expected result:** `terraform plan` shows a clear, complete list of resources to be created; `apply` succeeds; `kubectl get nodes` shows your Terraform-provisioned nodes Ready — functionally identical to Part XIV's manually-created cluster, but now fully defined in version-controlled code.

**Verification:** `terraform show` displays the complete current state, matching what `aws eks describe-cluster` reports independently — proof Terraform's state accurately reflects reality.

**Troubleshooting:** if `terraform apply` fails partway through, do **not** panic-delete resources manually — re-run `terraform plan` first; Terraform is designed to safely resume from a partial-apply state, and manual intervention outside Terraform is exactly what risks state/reality drift.

**Cleanup:**
```bash
terraform destroy -var-file="environments/lab/terraform.tfvars"
```
Then run the exact same "what could still be costing me money" verification commands from Chapter 13.6's lab regardless — Terraform's `destroy` is reliable, but independent verification is still the correct habit.

**Challenge:** migrate this configuration from local state to a remote S3 backend with DynamoDB locking, following Chapter 14.2's `backend "s3"` block — and explain, in your own words, exactly what problem this solves that local state doesn't, using a concrete two-person scenario.

---

*End of Part XV. Part XVI covers Observability — Prometheus and Grafana for metrics, centralized logging, and OpenTelemetry for tracing — giving ShopSphere real visibility into its own production behavior, followed immediately by Part XVII's Production Operations chapter.*
