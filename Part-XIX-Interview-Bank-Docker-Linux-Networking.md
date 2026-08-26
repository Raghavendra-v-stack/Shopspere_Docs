# The ShopSphere DevOps Book
## Part XIX — Final Interview Bank: Docker, Linux, and Networking

---

### How to use this bank

For each question: what the interviewer is actually testing, a strong answer, a common weak answer (and why it falls short), a natural follow-up, and what a genuinely experienced engineer might add. Don't just read these — cover the answer and try each one yourself first.

---

## Docker (55 Questions)

**1. What is a Docker container?**
*Tests:* Fundamental understanding, not memorized definitions.
*Strong:* A normal Linux process, isolated via namespaces and constrained via cgroups, running from a layered, portable image.
*Weak:* "A lightweight virtual machine."
*Follow-up:* How is that different from a VM?
*Senior take:* Ties the answer directly to sharing the host kernel, and the security implications of that.

**2. What's the difference between an image and a container?**
*Tests:* Basic vocabulary precision.
*Strong:* An image is a read-only, layered blueprint; a container is a running instance of it with a writable layer added on top.
*Weak:* "They're basically the same thing."
*Follow-up:* What happens to the writable layer when the container is removed?
*Senior take:* Notes multiple containers can share the same underlying image layers via OverlayFS.

**3. What is the Docker daemon?**
*Tests:* Architecture knowledge.
*Strong:* The background service (`dockerd`) that does the actual building/running work; the CLI is just a client talking to it over an API.
*Weak:* "It's the same as the CLI."
*Follow-up:* What happens if the daemon isn't running?

**4. Explain the difference between CMD and ENTRYPOINT.**
*Tests:* One of the most common Docker questions; tests real hands-on experience.
*Strong:* CMD is fully replaceable at `docker run` time; ENTRYPOINT is not — arguments passed at runtime are appended to it instead.
*Weak:* "They both set the startup command" (technically true, but misses the override behavior).
*Follow-up:* Show an example combining both.
*Senior take:* Mentions the exec-vs-shell form distinction and its effect on signal handling.

**5. Why prefer COPY over ADD?**
*Strong:* COPY does one predictable thing; ADD has surprising extra behaviors (archive extraction, remote URL fetch) that are rarely what's actually wanted.
*Weak:* "They're interchangeable."
*Follow-up:* When would ADD actually be the right choice?

**6. What's the difference between ARG and ENV?**
*Strong:* ARG is build-time only, not present in the final container; ENV persists into the running container.
*Weak:* "ARG is for arguments, ENV is for environment variables" (circular, no real understanding shown).
*Follow-up:* Why should neither hold a secret?

**7. Does EXPOSE actually publish a port?**
*Strong:* No — it's documentation/metadata; `-p` at `docker run` time is what actually publishes a port.
*Weak:* "Yes, it opens the port to the network."
*Follow-up:* What does `-p 8080:80` actually mean?

**8. Why does layer ordering in a Dockerfile matter?**
*Strong:* Docker caches layers; instructions that change rarely (dependency installs) should come before instructions that change often (copying source code), so unrelated code changes don't invalidate expensive cached layers.
*Weak:* "It doesn't really matter, just makes the file more organized."
*Follow-up:* What's a common Dockerfile mistake that breaks this?

**9. What is a multi-stage build, and why use one?**
*Strong:* Uses one image to build (with heavy tools), and copies only the final artifacts into a clean, minimal final image, keeping build-only tooling out of production.
*Weak:* "It's when you have more than one FROM statement" (true but misses the *why*).
*Follow-up:* Give a concrete example where this matters.

**10. What's a distroless image?**
*Strong:* A minimal base image with no shell, package manager, or unnecessary OS tooling — reduces size and attack surface, at the cost of harder in-container debugging.
*Weak:* "A really small image."
*Follow-up:* How would you debug a distroless container that's misbehaving?

**11. Why run a container as non-root?**
*Strong:* Limits the blast radius of a compromise — root inside a container (which shares the host kernel) is a stronger starting position for privilege escalation than a limited user.
*Weak:* "It's a best practice."
*Follow-up:* What Dockerfile instructions actually enforce this?

**12. What's dangerous about mounting the Docker socket into a container?**
*Strong:* It grants that container effective control over the entire host's Docker daemon — equivalent to root on the host — because it can instruct the daemon to start arbitrarily privileged containers.
*Weak:* "It could let the container access other containers."
*Follow-up:* What's a safer alternative for build-in-container workflows?
*Senior take:* Names kaniko or a similar daemonless builder specifically.

**13. What are Linux capabilities, and how do they relate to `--privileged`?**
*Strong:* Capabilities split root's monolithic power into individually grantable permissions; `--cap-drop`/`--cap-add` lets you grant only what's needed, versus `--privileged`'s all-or-nothing approach.
*Weak:* "Privileged mode gives more access."
*Follow-up:* Name one specific capability and what it grants.

**14. What's the difference between a Docker volume and a bind mount?**
*Strong:* A volume is Docker-managed, portable storage; a bind mount ties directly to a specific host filesystem path.
*Weak:* "They're both ways to persist data" (true but no real distinction shown).
*Follow-up:* Which would you use for local dev vs. production, and why?

**15. Why are containers described as disposable?**
*Strong:* The writable layer is meant to be thrown away — replacing a broken container with a fresh one from the same image is the expected recovery pattern, not repairing it in place.
*Weak:* "Because they're small."
*Follow-up:* What does this imply about where persistent data should live?

**16. What's the default Docker bridge network's limitation for multi-container apps?**
*Strong:* No automatic DNS-based name resolution between containers — a user-defined bridge network provides that.
*Weak:* "It's slower."
*Follow-up:* How does Compose handle this automatically?

**17. What happens if you forget `-p` when running a container that needs external access?**
*Strong:* The container's network namespace isolates it — nothing outside Docker can reach it without an explicit published port mapping.
*Weak:* "It just won't work" (no explanation of why).

**18. Explain `docker ps` vs `docker ps -a`.**
*Strong:* `-a` shows all containers including stopped ones; without it, a crashed container is invisible, which is a common early debugging trap.
*Weak:* "`-a` shows more containers."

**19. What's the difference between `docker stop` and `docker kill`?**
*Strong:* `stop` sends SIGTERM and waits a grace period before SIGKILL, allowing graceful shutdown; `kill` sends SIGKILL immediately.
*Weak:* "They both stop the container" (misses the graceful-vs-immediate distinction).
*Follow-up:* Why does this matter for an in-flight database transaction?

**20. Why might the shell form of CMD cause problems with graceful shutdown?**
*Strong:* It runs the app as a child of a shell process, so the app isn't PID 1 and may not receive the SIGTERM directly — exec form avoids this.
*Weak:* "It's slower to start."
*Senior take:* Connects this directly to why `docker stop` can seem to "not work" for some images.

**21. What is OverlayFS, and why does Docker use a layered filesystem?**
*Strong:* A union filesystem combining multiple read-only layers plus one writable layer — enables layer sharing/caching across images and containers, saving disk and speeding builds.
*Weak:* "It's how Docker stores files."

**22. What's the difference between a container runtime, a container engine, and containerd?**
*Strong:* The engine (Docker Engine) is the user-facing tool; containerd manages the container lifecycle and image storage; runc is the low-level runtime that actually creates namespaces/cgroups.
*Weak:* "They're all the same as Docker."
*Follow-up:* Does Kubernetes need Docker Engine specifically?
*Senior take:* Explains the CRI and why Docker Engine was dropped from kubelet's direct dependency chain.

**23. What is the OCI, and why does it matter?**
*Strong:* A standards body defining image and runtime specs, so images built by one tool can run on any compliant runtime — enables portability across the ecosystem.
*Weak:* "It's a Docker thing."

**24. What does `docker exec -it` actually do?**
*Strong:* Runs a new process inside a running container, with `-i` keeping STDIN open and `-t` allocating a pseudo-terminal — effectively an interactive shell into the container.
*Weak:* "It logs into the container."

**25. Why should you avoid `latest` tags in production?**
*Strong:* It's a moving target, not a fixed version — deployments referencing it can silently run different code over time, with no clear rollback target.
*Weak:* "It's just convention to avoid it."
*Follow-up:* What's an image digest, and how is it stronger than a tag?

**26. What is an image digest?**
*Strong:* A cryptographic hash of the image's exact content — immutable, unlike a tag which can technically be reassigned.
*Weak:* "Another name for the tag."

**27. How would you reduce the size of a Docker image?**
*Strong:* Use a minimal/slim or distroless base, multi-stage builds, combine RUN instructions to reduce layers where sensible, and add a `.dockerignore`.
*Weak:* "Delete unnecessary files at the end" (misses that deleted files in a later layer don't shrink earlier layers).
*Senior take:* Explains why deleting in a later layer doesn't reduce image size, due to how layers stack.

**28. What's a `.dockerignore` for?**
*Strong:* Excludes files from the build context, preventing bloat, slow builds, and accidental inclusion of secrets or local artifacts like `.git` or `.env`.
*Weak:* "Makes builds faster" (true but incomplete — misses the security angle).

**29. Why is `docker inspect` useful?**
*Strong:* Shows full container metadata — IP, mounts, environment variables, resource limits — essential for deep debugging beyond logs.
*Weak:* "Shows container details."

**30. Explain what happens internally when you run `docker run nginx`.**
*Strong:* Docker checks for the image locally, pulls if missing, creates a container with new namespaces and cgroup limits via containerd/runc, sets up networking, and starts the process.
*Weak:* Lists only "it downloads and runs nginx" with no internal detail.

**31. Why would you use `--read-only` on a container?**
*Strong:* Removes an entire category of attack — if there's no writable root filesystem, an attacker with code execution can't drop malicious files onto disk.
*Weak:* "For extra security" (no mechanism explained).
*Follow-up:* What do you do if the app genuinely needs to write somewhere?

**32. What's the difference between `docker rm` and `docker rmi`?**
*Strong:* `rm` removes a container; `rmi` removes an image — you generally can't remove an image still referenced by an existing container.
*Weak:* "They both delete things."

**33. How does Docker health-check work, and why use it?**
*Strong:* `HEALTHCHECK` runs a command periodically inside the container to determine actual application health, not just process liveness — surfaced in `docker ps` and usable by Compose's `condition: service_healthy`.
*Weak:* "It checks if the container is running" (conflates liveness with the process simply existing).

**34. Why is `depends_on` alone insufficient in Compose?**
*Strong:* It only controls container start order, not whether the dependency is actually ready to accept connections — a healthcheck is needed for that.
*Weak:* "It doesn't work well."
*Senior take:* Draws the direct parallel to Kubernetes readiness probes.

**35. What's the risk of baking a secret into a Docker image via ENV?**
*Strong:* It's visible via `docker inspect` or `docker history`/image layers to anyone with image or container access — not protected at all.
*Weak:* "Someone could see it if they had access to the server" (technically true but understates how trivially accessible it is).

**36. What is image scanning, and where does it fit in a real workflow?**
*Strong:* Analyzes image layers against known-vulnerability databases; typically integrated into CI/CD as a gate that blocks pushing/deploying vulnerable images.
*Weak:* "Checking if the image is safe."

**37. What's the difference between a registry, a repository, and a tag?**
*Strong:* A registry hosts many repositories; a repository holds versions of one image; a tag identifies a specific version within a repository.
*Weak:* "They're all the same concept."

**38. Why does AWS ECR use `get-login-password` instead of a fixed password?**
*Strong:* Generates a short-lived token rather than a permanent, static credential — reduces the risk window if it leaks.
*Weak:* "It's more secure" (no mechanism given).

**39. What's the difference between a bridge, host, and none Docker network?**
*Strong:* Bridge is the default isolated virtual network; host shares the host's network namespace directly (no isolation); none disables networking entirely except loopback.
*Weak:* "Different network modes" (no distinction given).

**40. What is tmpfs, and when would you use it?**
*Strong:* An in-memory-only mount that never touches disk and disappears when the container stops — good for short-lived sensitive data.
*Weak:* "A type of volume."

**41. Explain Docker's build cache and how to intentionally bust it.**
*Strong:* Docker reuses unchanged layers; a `RUN` instruction with a changing input (like a timestamp, or an explicit `--no-cache` build flag) forces a fresh execution.
*Weak:* "Docker remembers old builds."

**42. What's the security concern with `--privileged`?**
*Strong:* Grants nearly all Linux capabilities and host device access, effectively defeating most container isolation — should be reserved for genuinely narrow, specific needs, not general use.
*Weak:* "It gives extra permissions."

**43. How would you debug a container that exits immediately after starting?**
*Strong:* `docker logs`, then `docker ps -a` for the exit code, then possibly override the entrypoint with a shell to poke around interactively.
*Weak:* "Look at the logs" (incomplete — doesn't mention exit codes or entrypoint override).

**44. What does a nonzero exit code from a container typically indicate?**
*Strong:* The main process crashed or exited with an error — the specific code (e.g., 137 for SIGKILL/OOM, 1 for a general application error) narrows down the cause.
*Weak:* "Something went wrong."
*Follow-up:* What does exit code 137 specifically suggest?

**45. What's the purpose of a base image's `slim` variant?**
*Strong:* Strips docs, compilers, and other non-runtime-essential packages, reducing size and attack surface versus the full image.
*Weak:* "It's just smaller."

**46. How does Docker Compose handle service discovery?**
*Strong:* Automatically creates a shared network and DNS entries per service name, so services reach each other by name with zero manual config.
*Weak:* "It connects the containers."

**47. What's the practical difference between `docker compose down` and `docker compose down -v`?**
*Strong:* The former stops/removes containers and the default network but keeps named volumes; `-v` also removes volumes, meaning genuine data loss.
*Weak:* "One removes more stuff" (doesn't flag the data-loss risk).

**48. Why might you `--cap-drop=ALL` and then add back one specific capability?**
*Strong:* Applies least privilege precisely — grant only the exact kernel permission the process genuinely needs, nothing more.
*Weak:* "For security."

**49. What's the tradeoff of using a distroless or scratch-based image?**
*Strong:* Smaller, more secure, but harder to debug live (no shell to exec into) — often mitigated with a separate debug-variant image.
*Weak:* "It's harder to use."

**50. Explain what "container escape" means, at a high level.**
*Strong:* A vulnerability (often kernel-level) that lets a process break out of its container's namespace/cgroup isolation and affect the host or other containers — a real, if rare, risk of the shared-kernel architecture.
*Weak:* "When a container breaks."

**51. Why does Docker's build context matter for build performance?**
*Strong:* The entire build context is sent to the daemon before the build starts — a large, unfiltered context (no `.dockerignore`) slows every build unnecessarily.
*Weak:* "It's the files the Dockerfile uses" (correct but doesn't address the performance angle asked).

**52. What's a sensible way to handle configuration that differs between dev and production in a Docker-based workflow?**
*Strong:* Externalize it — environment variables or mounted config, injected at runtime, not baked into the image, so the same artifact is promoted across environments unchanged.
*Weak:* "Use different Dockerfiles per environment" (creates drift risk between environments).

**53. What does `docker stats` show, and when would you use it?**
*Strong:* Live CPU/memory/network usage per container — useful for real-time debugging of resource-related issues before reaching for Kubernetes-level tooling.
*Weak:* "Container statistics."

**54. Why is it a mistake to assume a healthy `docker ps` output means the application inside is actually working?**
*Strong:* `docker ps` only reflects process/container state, not application-level health — a hung or broken app can still show as "Up."
*Weak:* "It might not be accurate."

**55. What's the relationship between Docker and OCI regarding portability?**
*Strong:* Because Docker builds OCI-compliant images, those images can run on any OCI-compliant runtime (containerd, CRI-O, Podman) — Docker isn't a hard requirement downstream.
*Weak:* "Docker uses OCI standards" (vague, no portability implication given).

---

## Linux (27 Questions)

**1. What does `chmod 755` actually set?**
*Strong:* Owner: read/write/execute; group and others: read/execute — expressed as three octal digits mapped to rwx bits.
*Weak:* "It sets permissions" (no specifics).

**2. What's the difference between `chmod` and `chown`?**
*Strong:* `chmod` changes permission bits; `chown` changes the owning user/group of a file.
*Weak:* "They're similar commands."

**3. What is a process, and how does it differ from a thread?** *(Foundational, sometimes asked even in a DevOps context)*
*Strong:* A process is an independently executing program with its own memory space; threads share memory within one process.
*Weak:* "A process is a running program" (fine as far as it goes, but no thread distinction).

**4. What does `systemctl status` tell you that `ps aux` doesn't?**
*Strong:* Service-level state (active, failed, enabled-on-boot) and recent log excerpts for a managed service, not just raw process existence.
*Weak:* "It shows if the service is running" (same info as `ps`, misses the extra context).

**5. How would you find which process is using a specific port?**
*Strong:* `lsof -i :PORT` or `ss -tulnp | grep PORT`.
*Weak:* "Check the logs."

**6. What's the purpose of environment variables, in your own words?**
*Strong:* Pass configuration to a running process without hardcoding values into source code — the same mechanism underlying Docker ENV and Kubernetes env vars.
*Weak:* "They store values."

**7. What does `PATH` do?**
*Strong:* A list of directories the shell searches, in order, when resolving a bare command name.
*Weak:* "It's where programs are."

**8. Explain the difference between a hard link and a symbolic link.**
*Strong:* A hard link points directly to the same inode as the original file; a symlink is a separate file containing a path reference, and breaks if the target is moved.
*Weak:* "They're both shortcuts."

**9. What does piping (`|`) actually do?**
*Strong:* Feeds one command's stdout directly as the next command's stdin, letting small focused tools be chained together.
*Weak:* "Connects two commands."

**10. How would you search recursively for a text pattern across many files?**
*Strong:* `grep -r "pattern" ./directory`.
*Weak:* "Use grep."

**11. What's the difference between `>` and `>>` in shell redirection?**
*Strong:* `>` overwrites the target file; `>>` appends to it.
*Weak:* "They redirect output" (misses the overwrite-vs-append distinction, which is the actual point of the question).

**12. What does `kill -9` do differently from a plain `kill`?**
*Strong:* Plain `kill` sends SIGTERM (a request the process can catch and handle gracefully); `-9` sends SIGKILL, which can't be caught or ignored — an immediate, forceful termination.
*Weak:* "It kills the process harder."

**13. Why is it generally better to send SIGTERM before SIGKILL?**
*Strong:* Gives the process a chance to clean up (close connections, flush data) before being forcefully terminated — the same principle behind `docker stop`'s grace period.
*Weak:* "It's more polite."

**14. What's the significance of PID 1 in a Linux process tree?**
*Strong:* The first process started; in containers, whichever process is PID 1 is the one that directly receives signals like SIGTERM — a shell wrapping the real app can break this.
*Weak:* "It's the first process."

**15. How would you check disk usage by directory?**
*Strong:* `du -sh */` (or `du -h --max-depth=1`).
*Weak:* "Use `df`" (df shows filesystem-level usage, not per-directory — a common confusion worth catching).

**16. What's the difference between `df` and `du`?**
*Strong:* `df` reports filesystem-level free/used space; `du` reports actual disk usage of specific files/directories.
*Weak:* "They both show disk space" (true but misses the distinction being tested).

**17. Why might `df` show a filesystem as full even after you deleted large files?**
*Strong:* A process may still hold an open file handle to the deleted file, keeping the space allocated until that process closes it or is restarted.
*Weak:* "Deletion can be delayed" (vague, no mechanism given).
*Senior take:* A genuinely strong, non-obvious answer many candidates miss entirely.

**18. What does `awk '{print $1}'` do?**
*Strong:* Prints the first whitespace-delimited field of each input line.
*Weak:* "Processes text."

**19. What's the purpose of `sed`?**
*Strong:* A stream editor, commonly used for find-and-replace and other line-by-line text transformations, without opening an interactive editor.
*Weak:* "Edits files."

**20. What does `curl -I` do differently from a plain `curl`?**
*Strong:* Sends a HEAD request and shows only response headers, without downloading the body — fast way to check status/headers.
*Weak:* "Shows info about the request."

**21. Why would a DevOps engineer care about Linux file permissions in a container context specifically?**
*Strong:* Ties directly to non-root container users (Part V) — an application running as a non-root UID needs correctly-owned files, or it fails with permission errors at startup.
*Weak:* "Permissions are important for security" (true but not connected to the specific container scenario asked).

**22. What's the practical difference between a process being "zombie" and being "orphaned"?**
*Strong:* A zombie has exited but its exit status hasn't been reaped by its parent yet; an orphan's parent has exited, and it gets reparented (often to PID 1).
*Weak:* "They're both dead processes" (imprecise — a zombie is technically no longer running at all).

**23. How would you check what version of a package is installed?**
*Strong:* Depends on the package manager — `apt list --installed`, `dpkg -l`, `rpm -q`, etc. — the point is knowing to check the relevant manager, not one universal command.
*Weak:* "Check the package manager" (correct instinct, but a strong answer names the actual commands).

**24. What is `/etc` typically used for?**
*Strong:* System-wide configuration files.
*Weak:* "System files" (too vague — could describe almost any directory).

**25. Why might `find` be preferred over `ls` for locating files across a large directory tree?**
*Strong:* `find` recurses and supports rich filtering (name, type, modified time, size); `ls` is shallow and mostly for listing, not searching.
*Weak:* "find is more powerful" (true but no specifics).

**26. What does it mean for a shell script to "fail silently," and how would you prevent it?**
*Strong:* A command fails but the script continues as if nothing happened, because the exit code wasn't checked; `set -e` (exit on any command failure) is a common guard.
*Weak:* "The script doesn't show an error."

**27. Why does SSH matter conceptually, even once Kubernetes is in the picture?**
*Strong:* Establishes the same "securely execute commands on a remote system" need that `kubectl exec` later fulfills at the container level — same underlying concept, different layer.
*Weak:* "It's how you log into servers" (correct but doesn't connect it to the book's broader pattern, which is what the question is really testing).

---

## Networking (26 Questions)

**1. What's the difference between a public and private IP address?**
*Strong:* Public IPs are globally routable and reachable from the internet; private IPs are only reachable within a local network.
*Weak:* "Public is for the internet, private is for local" (correct but shallow — a strong answer adds routability as the actual mechanism).

**2. What does a port number actually do?**
*Strong:* Identifies a specific channel/service on a machine, letting multiple services share one IP address.
*Weak:* "It's a number for connections."

**3. What's the difference between TCP and UDP?**
*Strong:* TCP is connection-based and guarantees reliable, ordered delivery; UDP is connectionless, faster, but doesn't guarantee delivery or order.
*Weak:* "TCP is more reliable" (true but incomplete — doesn't explain the tradeoff or use cases).
*Follow-up:* Give an example of when UDP is the better choice.

**4. What is DNS, and why does it exist?**
*Strong:* Resolves human-readable names to IP addresses; exists because IPs change and are hard to remember, while a stable name solves both.
*Weak:* "It's the internet's phonebook" (fine as an analogy, but a strong answer adds the *why*).

**5. What is CIDR notation, and what does a smaller number after the slash mean?**
*Strong:* Describes a range of IP addresses via a fixed-bits prefix; a smaller number after the slash means fewer fixed bits, hence a *larger* address range.
*Weak:* "It's how you write IP ranges" (doesn't demonstrate understanding of the actual size relationship, which is the real test).

**6. What does NAT do?**
*Strong:* Translates private IP addresses to a shared public one at a network boundary, letting private-network devices reach the internet without individual public IPs.
*Weak:* "It changes IP addresses" (too vague).

**7. What's the role of a NAT Gateway specifically in an AWS VPC?**
*Strong:* Lets resources in a private subnet reach the internet outbound (e.g., to pull a container image) without being directly reachable inbound.
*Weak:* "It's for networking in AWS."

**8. Why would a load balancer perform health checks?**
*Strong:* To stop routing traffic to backend targets that are down or unhealthy, instead of returning errors to users.
*Weak:* "To check if things are healthy" (circular).

**9. What's the difference between a firewall and a router, conceptually?**
*Strong:* A router directs traffic toward its destination; a firewall decides whether traffic is *allowed* to proceed at all, based on rules.
*Weak:* "They both handle network traffic" (doesn't distinguish their actual jobs).

**10. What is a subnet, and why divide a network into them?**
*Strong:* A subdivision of an IP address range, often tied to a specific physical/logical boundary (like an Availability Zone) — used to segment traffic and apply different access rules to different groups of resources.
*Weak:* "A smaller network."

**11. Why does spreading subnets across multiple Availability Zones matter?**
*Strong:* Protects against a single AZ failure taking down all resources — the same high-availability principle applied at the network layer.
*Weak:* "For redundancy" (correct instinct, underdeveloped explanation).

**12. What's the difference between a Security Group and a Network ACL, at a conceptual level?**
*Strong:* Security Groups are stateful and attached to individual resources; NACLs are stateless and applied at the subnet level.
*Weak:* "They're both firewalls" (misses the stateful/stateless and resource/subnet distinctions, which is the actual point).

**13. What does "stateful" mean in the context of a firewall rule?**
*Strong:* A response to an allowed request is automatically permitted back through, without needing a separate matching rule for the return traffic.
*Weak:* "It remembers things" (imprecise).

**14. Why is HTTPS preferred over HTTP for production traffic?**
*Strong:* Encrypts traffic in transit, preventing eavesdropping and tampering — HTTP sends everything, including credentials, in plaintext.
*Weak:* "It's more secure" (true but no mechanism given).

**15. What does DNS resolution actually involve, at a high level?**
*Strong:* A client queries a resolver, which (if not cached) walks the DNS hierarchy — root, TLD, authoritative nameservers — to find the IP for a given name.
*Weak:* "It looks up the IP address" (correct outcome, no process described).

**16. What's a load balancer's role beyond just distributing traffic evenly?**
*Strong:* Also performs health checks, can terminate TLS, and can implement routing algorithms beyond simple round-robin (least-connections, weighted).
*Weak:* "It spreads out traffic."

**17. What's the difference between Layer 4 and Layer 7 load balancing, at a conceptual level?**
*Strong:* Layer 4 routes based on IP/port (transport layer) without inspecting content; Layer 7 (e.g., Ingress/ALB) can route based on HTTP-level details like hostname or path.
*Weak:* "One is more advanced" (doesn't explain what's actually different).

**18. What does it mean for a route table entry to send `0.0.0.0/0` traffic somewhere?**
*Strong:* It's the default/catch-all route — any destination not matched by a more specific route goes this way (e.g., to an Internet Gateway or NAT Gateway).
*Weak:* "It's a rule for all traffic" (correct but doesn't explain the catch-all/default nature clearly).

**19. Why would a database server typically have no public IP?**
*Strong:* It has no legitimate reason to be directly reachable from the internet; removing that path entirely reduces attack surface, consistent with defense-in-depth.
*Weak:* "For security" (too generic — a strong answer explains the specific reasoning).

**20. What's an Elastic IP, and why can it cost money even when "not doing anything"?**
*Strong:* A static public IP address; AWS charges for EIPs that are allocated but *not* attached to a running resource, to discourage IP address hoarding.
*Weak:* "A fixed IP address" (misses the specific billing behavior being asked about).

**21. What is an Internet Gateway's role in a VPC?**
*Strong:* Provides the actual path for traffic between a VPC's public subnets and the internet.
*Weak:* "It connects to the internet" (vague, doesn't distinguish it from a NAT Gateway).

**22. Why might two engineers disagree about whether a resource "needs" a public IP?**
*Strong:* A legitimate tension between operational convenience (direct access for debugging) and security best practice (minimizing exposed surface) — the correct answer depends on context, not a universal rule.
*Weak:* "It should always have one" or "It should never have one" (either extreme misses the nuance being tested).

**23. What does "east-west" vs. "north-south" traffic mean in networking discussions?**
*Strong:* North-south is traffic entering/leaving the network (client to server); east-west is traffic between services *within* the network (e.g., Pod-to-Pod).
*Weak:* "Different traffic directions" (doesn't define which is which).

**24. Why is DNS caching both helpful and occasionally a source of confusing bugs?**
*Strong:* Caching reduces lookup latency and load, but a cached stale record (e.g., after a service's IP changes) can cause a client to keep hitting an outdated destination until the TTL expires.
*Weak:* "Caching can cause problems" (no mechanism given).

**25. What's the practical difference between a CNAME and an A record?**
*Strong:* An A record maps a name directly to an IP address; a CNAME maps a name to *another name*, which then gets resolved — useful for pointing at something whose IP might change, like a load balancer.
*Weak:* "They're both DNS record types."

**26. Why can't a CNAME be used at the root/apex of a domain, and what's the common AWS workaround?**
*Strong:* DNS specification restrictions prevent a CNAME from coexisting with other records required at the zone apex; Route 53's alias record solves this by behaving like a CNAME but being usable at the apex.
*Weak:* "You just can't do that" (doesn't explain why, or name the actual workaround).

---

*End of Part XIX. Part XX continues the Final Interview Bank with Kubernetes — the largest single section, at 75+ questions.*
