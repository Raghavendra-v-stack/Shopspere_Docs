# The ShopSphere DevOps Book
## Part V — Docker Security and Container Registries

---

### Where we left off

ShopSphere runs locally as a full stack via Compose. Before this goes anywhere near production, we need to take container security seriously — and we need a way to actually distribute images beyond your own laptop. Both of these are prerequisites for Kubernetes, which is coming next in Part VI.

---

## Chapter 4.1 — Docker Security

Let's ground this the same way we grounded containers themselves in Part II: not as a checklist, but as a consequence of what a container actually is. A container shares the host's kernel. That single fact is the source of nearly every meaningful Docker security concern — so we'll work outward from it.

### Root vs. non-root, revisited properly

We touched this in Part III's Dockerfile fixes; now let's go deeper into *why* it matters so much.

**Simple explanation:** if a container runs as root and something goes wrong — a vulnerability in your app gets exploited — the attacker now has root privileges *inside* that container, which is a much stronger starting position than if the process had been running as a limited, unprivileged user.

**Proper definition:** by default, unless a Dockerfile specifies otherwise, the container's main process runs as **UID 0 (root)** — the same root user ID that exists on the host's kernel, because remember, there's no separate kernel per container (Part II). Root *inside* the container is a real, powerful identity to the one shared kernel underneath everything.

**Why it matters concretely:** many container escape vulnerabilities that have been found over the years specifically require root privileges inside the container to exploit. A non-root container doesn't eliminate the risk of a kernel vulnerability, but it closes off an entire category of privilege-escalation paths — the attacker would need to escalate to root *inside* the container first, adding a whole extra obstacle.

```dockerfile
RUN useradd --create-home appuser
USER appuser
```

We covered this fix in Part III — the point now is understanding *why* it's not just a "best practice checkbox," but a real, mechanistic reduction of attack surface tied directly to the shared-kernel architecture from Part II.

### The Docker socket: a genuinely dangerous footgun

**Simple explanation:** the Docker socket is the "phone line" the Docker CLI uses to talk to the Docker daemon. If a container gets access to that phone line, it can ask the daemon to do absolutely anything — including starting a brand-new container with full access to the host.

**Proper definition:** `/var/run/docker.sock` is the Unix socket the Docker daemon listens on for API requests. Mounting it into a container (`-v /var/run/docker.sock:/var/run/docker.sock`) gives that container the ability to control the Docker daemon itself.

**Why this is a serious risk:** a container with access to the Docker socket can, for example, request the daemon to start a new container with `--privileged` and a bind mount of the entire host filesystem — which is effectively full host takeover, entirely bypassing whatever isolation the *original* container had. This pattern shows up surprisingly often in the wild — usually because a CI/CD tool running inside a container needs to build or run other Docker images ("Docker-in-Docker" style workflows), and the Docker socket mount is the quick-and-easy way people wire that up.

**How you'd encounter this at a real company:** a Jenkins agent running as a container that needs to build Docker images is a completely realistic, common case (we'll build exactly this in Part X) — and it's exactly the situation where teams either mount the Docker socket (convenient, but means that Jenkins container can control the *entire host's* Docker daemon) or use a safer, more isolated approach (a dedicated Docker-in-Docker container, or rootless builders like `img`/`buildkit`/`kaniko`). Knowing this tradeoff exists — and being able to articulate it — is a genuine interview differentiator.

**Interview question (advanced):** "Why is mounting the Docker socket into a container considered a security risk, even though the mounting container itself might be trusted?" — Access to the Docker socket is equivalent to root access on the host, because it lets a container instruct the daemon to start new, arbitrarily privileged containers — including ones that mount the entire host filesystem — completely sidestepping any isolation the original container had.

### Privileged containers

**Simple explanation:** a privileged container has essentially all of the host's kernel capabilities available to it — it's about as close to "not actually contained" as a container can get.

**Proper definition:** running a container with `--privileged` disables most of the security restrictions Docker normally applies, granting the container nearly all Linux capabilities and direct access to host devices.

**Why it (rarely) exists:** some legitimate workloads genuinely need deep hardware or kernel access — certain low-level networking tools, or containers that need to manage other containers' cgroups/namespaces directly (like a Docker-in-Docker builder). It's a narrow, specific need, not a general-purpose flag.

**The mistake to avoid:** using `--privileged` as a lazy fix for a permissions error ("just run it privileged, that'll fix it") instead of understanding and granting the *specific* capability actually needed.

### Linux capabilities: the properly scoped alternative

**Simple explanation:** instead of the all-or-nothing choice of "root vs. non-root" or "privileged vs. not," Linux capabilities let you grant a process one narrow, specific superpower — like "the ability to bind to a low-numbered network port" — without granting full root.

**Proper definition:** **Linux capabilities** split up what used to be the single monolithic "root can do anything" privilege into dozens of distinct, individually grantable permissions (`CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`, `CAP_CHOWN`, and many more).

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE shopsphere-backend:v1
```

This drops every capability, then grants back exactly one — the ability to bind to a privileged port (below 1024) — instead of leaving the container with the full default capability set, or the extreme of `--privileged`.

**Interview question (intermediate):** "What's the difference between running a container as non-root and using `--cap-drop`?" — `USER` controls *which user identity* the process runs as; `--cap-drop`/`--cap-add` controls *which specific kernel-level privileges* are available to the process regardless of user, offering much finer-grained control than the binary root/non-root distinction. Production hardening typically uses both together.

### Read-only filesystems

**Simple explanation:** if a container's filesystem doesn't need to be writable at runtime, making it explicitly read-only removes an entire category of attack (an attacker who gets code execution can't drop malicious files onto disk if there's no writable disk to drop them onto).

```bash
docker run --read-only --tmpfs /tmp shopsphere-backend:v1
```

The `--tmpfs /tmp` here gives the application the one small writable, in-memory scratch space it likely still needs (many apps expect to write temp files somewhere), while everything else stays locked down.

### Secrets, done properly

We flagged this warning back in Part III: never put secrets in `ENV` or `ARG`. Let's now actually cover the right approach.

**The core problem:** anything baked into an image — an environment variable, a file copied in via `COPY` — is visible to anyone who can pull that image or run `docker history`. Images frequently end up in registries that more people than you'd expect have some form of access to.

**Better approaches, in rough order of how you'll encounter them:**

- **Runtime environment variables**, injected at `docker run` / `docker compose up` time rather than baked into the image — better than a hardcoded `ENV`, but still visible via `docker inspect` to anyone with access to the running container, so still not ideal for genuinely sensitive values.
- **Docker secrets** (in Swarm mode) or **mounted secret files** — the secret is provided as a file inside the container's filesystem at runtime, not as an environment variable, and not part of the image layers at all.
- **External secret managers** — AWS Secrets Manager, HashiCorp Vault — where the application fetches the secret at startup, over an authenticated API call, and the secret never touches a Dockerfile, an image layer, or version control at all.

We'll build the full production version of this — Kubernetes Secrets combined with AWS Secrets Manager — in Part VII. The principle to internalize now: **the image itself should never contain a secret.**

### Image scanning and vulnerability management

**Simple explanation:** an image scanner looks inside your image's layers and compares every installed package against databases of known vulnerabilities, so you find out about a dangerous, outdated library *before* it ships to production, not after.

**Proper definition:** **image scanning** analyzes a container image's filesystem and installed packages against vulnerability databases (like the CVE database) to identify known security issues, typically categorized by severity.

```bash
docker scout cves shopsphere-backend:v1
```

(Other common tools you'll encounter in real companies: Trivy, Grype, Snyk — and AWS ECR has built-in scanning, which we'll use directly in Chapter 4.3.)

**Where this fits in a real workflow:** image scanning is almost always wired into CI/CD, so a build with a critical vulnerability can be automatically blocked from being pushed to a registry or deployed — we'll build exactly this gate into ShopSphere's Jenkins pipeline in Part X.

### Resource limits, as a security tool too

We covered `--memory` and `--cpus` in Part II as a reliability tool (stopping one container from starving others). They're also a security tool: without limits, a compromised or buggy container can be used to exhaust host resources and degrade or take down every other workload on that machine — a form of denial-of-service that doesn't require breaking out of the container at all.

### Minimal base images, revisited

We covered `slim` and `distroless` images in Part III for size and build hygiene. From a security lens specifically: fewer installed packages means fewer things that can have a vulnerability in the first place, and no shell/package manager (distroless) means an attacker with code execution has meaningfully fewer tools available to them even if they succeed.

### Common security mistakes, named directly

- Running as root with no justification.
- Mounting the Docker socket into an application container "just to make something work."
- Using `--privileged` instead of scoped capabilities.
- Baking secrets into images via `ENV`, `ARG`, or `COPY`.
- Using `latest` tags in production, making it unclear exactly what's running and impossible to reliably roll back to (we covered the reproducibility angle of this in Part III — here's the security angle: you also can't be sure which known-vulnerable version you might silently be running).
- Never scanning images, and finding out about a critical vulnerability only after an incident.

**Interview question (scenario, advanced):** "You inherit a Dockerfile that runs as root, mounts the Docker socket, and stores an API key in an `ENV` line. Walk through how you'd fix each issue and why." — Strong answer: switch to a non-root `USER`; remove the Docker socket mount and replace whatever workflow needed it with a purpose-built, isolated build approach (or a scoped alternative like a rootless builder); move the API key out of the Dockerfile entirely, injecting it at runtime via a secret manager or mounted secret file instead of an image layer; and add image scanning to CI so future regressions are caught automatically rather than relying on manual review.

---

## Chapter 4.2 — Container Registries: The Concept

### Why they exist

Right now, ShopSphere's `shopsphere-backend:v1` image exists only on your laptop. It needs to get to other machines — a teammate's laptop, a CI server, and eventually production servers — without everyone rebuilding it from source individually (which, remember from Part III, isn't even guaranteed to produce an identical result unless the Dockerfile is airtight).

**Simple explanation:** a container registry is a storage-and-distribution service for container images — you push an image there once, and anyone with the right access can pull an identical copy of exactly that image.

**Proper definition:** a **container registry** stores container images, organized into repositories, addressed by a name and tag (or digest), and made available for `docker pull` over a defined API (the OCI Distribution Spec, which relates directly to the OCI Image Spec from Part II — same standards body, same "different tools all agree on one format" motivation).

```text
Dockerfile
   |
docker build
   |
Local Image
   |
docker push
   |
Registry
   |
docker pull  (from anywhere with access)
   |
Identical Image
```

### Docker Hub

**Docker Hub** is the default, most widely-used public registry — where official images like `postgres`, `redis`, and `python` (all of which you've used throughout this book) actually live. It supports both public repositories (anyone can pull) and private ones (access-controlled).

```bash
docker login
docker tag shopsphere-backend:v1 yourdockerhubuser/shopsphere-backend:v1
docker push yourdockerhubuser/shopsphere-backend:v1
```

**Why a company wouldn't just use Docker Hub for everything:** rate limits on pulls (which have caused real production outages at companies that didn't realize they were hitting them), less control over network access and compliance requirements, and generally less integration with a specific cloud provider's identity and access system. This is exactly why cloud-native private registries exist.

### Private registries and authentication

A **private registry** requires authentication to pull or push images — critical for anything containing proprietary code (which, remember from Part III's caching-friendly Dockerfile, includes your actual application source, copied directly into the image).

```bash
docker login registry.shopsphere.example
docker push registry.shopsphere.example/shopsphere-backend:v1
```

Authentication typically works via a token or credential stored locally (`~/.docker/config.json`) after `docker login`, and in CI/CD systems, via a dedicated service credential rather than a human's personal login — a distinction that matters a great deal once we build the Jenkins pipeline in Part X, where Jenkins itself needs registry credentials, scoped narrowly and stored securely, not a developer's personal password.

### Image tags and image versioning

**Why explicit tags matter, tied back to Part III:** we already flagged that `latest` is a moving target, not a fixed version. In a registry context this becomes even more consequential, because now *multiple systems* — your laptop, CI, and production — all need to agree on exactly which image is running.

**A common, effective real-world tagging strategy:**

```text
shopsphere-backend:1.4.2        <- semantic version, for humans
shopsphere-backend:a1b2c3d      <- git commit SHA, for exact traceability
shopsphere-backend:latest       <- convenience only, NEVER used in production deploys
```

**Image digests**, as an even stronger guarantee than tags: a **digest** is a cryptographic hash of the image's exact content (`sha256:e3b0c4...`). Unlike a tag, which can technically be *moved* to point at a different image later (someone could re-push `v1` with different content — bad practice, but possible), a digest is immutable by definition — it's mathematically tied to the exact bytes of that image. Production deployment manifests sometimes pin to a digest specifically for this reason, when absolute certainty matters more than human readability.

**Interview question (intermediate):** "Why shouldn't a production deployment use the `latest` tag?" — `latest` is not a fixed version — it's just a conventional default tag that gets reassigned to whatever was most recently pushed without one, so a production system referencing `latest` could silently start running different code after a routine build elsewhere, with no clear record of exactly what version is actually deployed, and no reliable way to roll back to a known-good state.

### Image lifecycle and image scanning at the registry level

Registries aren't just dumb storage — production-grade registries typically provide:

- **Lifecycle policies** — automatically expiring old, untagged, or unused images so storage costs and clutter don't grow unbounded forever.
- **Vulnerability scanning**, built directly into the registry (we'll use AWS ECR's version of this directly in the next chapter) — scanning images automatically on push, rather than relying on a separate manual step.
- **Access control**, so only specific services or people can push or pull specific repositories.

---

## Chapter 4.3 — AWS ECR

This is where ShopSphere's registry story actually lands, since we're using AWS as our cloud platform throughout this book (Part XI covers AWS properly — this is a focused first look, specifically at the registry piece).

**Simple explanation:** **Amazon ECR (Elastic Container Registry)** is AWS's own private container registry — it works like the private registry concept above, but it's tightly integrated with AWS's identity system (IAM) and with EKS, which we'll deploy to in Part XI.

**Why ShopSphere uses it instead of Docker Hub:** IAM-based access control (instead of a separate registry-specific login), no public-registry rate limits, images staying inside AWS's network when pulled by EKS nodes (lower latency, no extra egress cost for that traffic), and built-in vulnerability scanning integrated with the rest of AWS's tooling.

```bash
# Authenticate Docker to ECR (short-lived token, not a permanent password)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag the image for this specific ECR repository
docker tag shopsphere-backend:v1 \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1

# Push it
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
```

Notice `get-login-password` — this generates a **short-lived authentication token**, not a fixed password, following the same "avoid long-lived static credentials" principle we'll return to repeatedly across the AWS and Terraform chapters.

### Cost Warning — ECR

**Is this likely to cost money?** ECR has a genuinely useful **Free Tier**: (at the time of writing) a certain amount of storage free per month for the first year for new accounts, and beyond that, storage is billed per GB per month — genuinely low-cost for a handful of small images, but not literally free indefinitely.

**What resource causes the cost?** Storage (GB-months of image layers) and, in some configurations, data transfer.

**How can I minimize the cost?** Use lifecycle policies to automatically delete old/untagged images (exactly the "image lifecycle" concept above), and don't push every single experimental build — clean up test repositories when you're done with the personal-lab exercises in this book.

**How do I verify I've deleted everything afterward?**

```bash
aws ecr describe-repositories
aws ecr list-images --repository-name shopsphere-backend
aws ecr delete-repository --repository-name shopsphere-backend --force
```

`--force` deletes the repository even if it still contains images — useful for lab cleanup, but obviously not something to reach for casually against a real production repository.

**Personal Lab vs. Production, side by side:**

| | Production | Personal Lab |
|---|---|---|
| Registry | Amazon ECR, multiple repositories, lifecycle policies, scan-on-push enforced | Amazon ECR — genuinely fine to use directly; low storage cost for a few small learning images |
| Access control | IAM roles scoped per service/team | Your own IAM user, broader access acceptable for solo learning |
| Cleanup | Automated lifecycle policies | Manually `aws ecr delete-repository` when done with each lab |

Unlike EKS or NAT Gateways (which we'll cost-warn heavily about starting in Part XI), ECR is one of the AWS services in this book you can reasonably leave around during your learning process without much financial concern — just don't forget it exists indefinitely; check in on it occasionally with `aws ecr describe-repositories` and prune what you don't need.

---

## Chapter 4.4 — Checkpoint

**Beginner:**
1. What does a container registry actually store, and what's the relationship between an image, a tag, and a digest?
2. Why shouldn't production deployments reference the `latest` tag?

**Intermediate:**
3. Why is mounting the Docker socket into a container considered dangerous, even for a "trusted" internal tool?
4. What's the difference between running a container as non-root and using `--cap-drop=ALL --cap-add=...`?

**Advanced:**
5. Explain why an image digest provides a stronger guarantee than an image tag, and describe a real scenario where that difference matters.
6. Why does AWS ECR's `get-login-password` generate a short-lived token instead of using a fixed, permanent password?

**Scenario:**
7. A production incident review finds that a critical vulnerability shipped to production three weeks after it was publicly disclosed. What process gaps would you investigate, based on everything covered in this chapter?

---

### Hands-On Lab 4.1 — Harden the ShopSphere backend image and push it to ECR

**Objective:** Apply real security hardening to the Part III Dockerfile, scan it, and push it to a private registry.

**Prerequisites:** The `shopsphere-backend:v1` image from Part III; an AWS account with the AWS CLI configured (`aws configure`).

**Cost Warning:** this lab uses AWS ECR. As covered above, ECR has a genuine Free Tier and low storage costs for small images, but is not unconditionally free forever — review the pricing page for current numbers, and follow the cleanup step at the end regardless.

**Steps:**

1. Run the image with hardening flags and confirm it still works:
   ```bash
   docker run -d --name backend-hardened \
     --read-only --tmpfs /tmp \
     --cap-drop=ALL \
     -p 8000:8000 \
     shopsphere-backend:v1
   curl http://localhost:8000/
   ```

2. Scan the image for known vulnerabilities:
   ```bash
   docker scout cves shopsphere-backend:v1
   ```

3. Create an ECR repository and push:
   ```bash
   aws ecr create-repository --repository-name shopsphere-backend
   aws ecr get-login-password --region us-east-1 | \
     docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com
   docker tag shopsphere-backend:v1 <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
   docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/shopsphere-backend:v1
   ```

4. Confirm ECR's own scan results:
   ```bash
   aws ecr describe-image-scan-findings \
     --repository-name shopsphere-backend --image-id imageTag=v1
   ```

**Expected result:** the hardened container still serves requests despite `--read-only` and dropped capabilities (proving it didn't actually need root-level privileges or a writable root filesystem in the first place); the scan reports produce a list of findings (ideally few or none critical, for a `slim` base image); the image appears in ECR.

**Verification:** `aws ecr list-images --repository-name shopsphere-backend` shows your pushed tag.

**Troubleshooting:** if the hardened container fails to start, it likely needs a writable path somewhere the app doesn't expect — add a targeted `--tmpfs` mount for that specific path rather than abandoning `--read-only` altogether.

**Cleanup:**
```bash
docker rm -f backend-hardened
aws ecr delete-repository --repository-name shopsphere-backend --force
```

**What could still be costing me money?** Only the ECR repository itself if you skip the cleanup step — confirm it's gone with `aws ecr describe-repositories` (it should return an error naming the repository as not found).

**Challenge:** modify the Dockerfile to add a non-root `USER` (from Part III) *and* run it with `--cap-drop=ALL`, then figure out, using `docker logs`, whether it still starts successfully — and if not, use `--cap-add` to grant back only the one specific capability it actually needs.

---

*End of Part V. Part VI begins Kubernetes: what problem it actually solves that Compose and manual `docker run` fundamentally can't, followed by Kubernetes's full architecture — the control plane and worker nodes — and tracing exactly what happens internally, step by step, when you run your very first `kubectl create deployment`.*
