# The ShopSphere DevOps Book
## Part I — Foundations

---

### How this book works

Before we start, a quick note on the approach, because it's different from most tutorials you've probably used.

Everything in this book happens inside one story: you've just joined a company called **ShopSphere** as a junior DevOps engineer. Every technology we introduce — Docker, Kubernetes, Terraform, Jenkins, all of it — gets introduced because ShopSphere needs it to solve a real problem, in the order a real engineer would encounter it. We're not going to jump around between unrelated demo apps. The same application grows in complexity chapter by chapter, the same way a real system does.

Whenever we hit a new technical term, we'll follow the same pattern every time:

1. **Simple explanation** — what it is, in plain language.
2. **Proper definition** — the term you'd actually use in a meeting or interview.
3. **Why it exists** — what problem it solves.
4. **Where it's used** — in ShopSphere and in the wider industry.
5. **A practical example.**
6. **How you'd encounter it at a real company.**
7. **An interview question about it.**

This is intentional. A lot of learning material either drowns you in jargon or protects you from it so much that you can't hold a technical conversation later. We want both: understanding *and* vocabulary.

---

## Chapter 0.1 — Meet ShopSphere

ShopSphere is a small e-commerce company. They sell physical products online — think a mid-sized online store, not quite Amazon, but big enough to have real engineering problems.

**What the application does:**

- Customers browse products and place orders (the **frontend** — a web UI).
- Orders, product data, and user accounts are handled by an API (the **backend**).
- Order and product data lives in a **database** (PostgreSQL).
- Frequently-read data (like product listings) is cached in **Redis** so the database isn't hit on every single request.
- When an order is placed, a **background worker** processes it asynchronously — for example, sending a confirmation email or updating inventory — so the customer isn't stuck waiting on the checkout button.

Here's the shape we're eventually building toward. Don't worry about understanding all of it yet — this is the destination, not the starting point:

```text
                    Internet
                       |
                       v
                  Load Balancer
                       |
                       v
                    Ingress
                       |
          +------------+------------+
          |                         |
       Frontend                  Backend API
                                    |
                     +--------------+--------------+
                     |              |              |
                 PostgreSQL       Redis         Worker
```

**What ShopSphere's infrastructure looks like on your first day:**

This is the important part, and it's realistic: it's a mess, in the way a lot of real early-stage companies are a mess.

- The backend, frontend, and worker all run as plain processes on a single EC2-like Linux server.
- One engineer manually SSHs in, pulls the latest code with `git pull`, restarts the process, and hopes nothing breaks.
- PostgreSQL runs directly on that same server.
- There's no automated deployment. There's no monitoring beyond "someone notices the site is down."
- When a new engineer joins, they spend two days trying to get the app running locally because the setup instructions are a stale wiki page.

**What problems does this create?**

- **"It works on my machine."** One developer has Python 3.11, another has 3.9. One has a system library installed globally that the app secretly depends on. The app behaves differently on different laptops.
- **Deploys are risky and manual.** There's no way to easily roll back a bad deploy. A typo during a manual server restart can take the site down.
- **No isolation.** If the worker process leaks memory, it can starve the backend API running on the same machine.
- **Scaling is a pain.** More traffic before a big promotional sale? Someone has to manually provision another server and copy the setup by hand.
- **New developer onboarding is slow**, because "how do I run this app" isn't a solved, repeatable process.

This is precisely the situation that leads companies to adopt containers, and eventually Kubernetes. We are going to solve these problems one at a time, in the order a real engineering team would.

**What will you, the DevOps engineer, actually be responsible for?**

Broadly: making it fast, safe, and repeatable to build, ship, and run this application — and making sure that when it breaks at 2 a.m., there's a clear path to figuring out why. That covers:

- Packaging the application consistently (Docker).
- Orchestrating and scaling it reliably (Kubernetes).
- Automating the path from code to production (CI/CD, Jenkins).
- Provisioning and managing the underlying infrastructure (Terraform, AWS).
- Making the system observable so problems are visible before customers complain (monitoring, logging).
- Keeping it secure and available.

We'll build toward all of it. But first — before Docker even enters the picture — there's a small set of Linux and networking fundamentals you need, because every explanation of *why* Docker and Kubernetes work the way they do rests on these ideas. We'll keep this tight. This isn't a full Linux course; it's exactly what you need for what's coming.

---

## Chapter 0.2 — Linux Essentials for DevOps

Every Docker container you'll ever run is, underneath, a Linux process. Every Kubernetes node is a Linux machine. If Linux is fuzzy, Docker and Kubernetes will always feel like magic instead of a system you understand. So let's ground this.

### The filesystem and permissions

Linux organizes everything — files, devices, configuration — as files in a single tree starting at `/` (the root). A few directories you'll see constantly:

- `/etc` — system configuration files.
- `/var/log` — log files.
- `/home` — user home directories.
- `/usr/bin`, `/bin` — where executable programs live.

**Users and groups.** Every file on Linux is owned by a **user** and a **group**. This matters enormously in containers and Kubernetes, because a huge category of production security issues comes down to "this process is running as a more privileged user than it needs to be."

**Permissions.** Every file has three permission sets — for the owner, the group, and everyone else — each of which can allow **r**ead, **w**rite, or e**x**ecute. You'll see this as a string like `-rwxr-xr--` or as octal numbers like `755`.

```bash
chmod 644 app.py      # owner can read/write, everyone else can only read
chown appuser:appuser app.py   # change the owner (and group) of a file
```

**Interview angle:** "Why would a Dockerfile create a non-root user and `chown` the application files to it?" — Because if an attacker compromises the app, running as a non-root user limits what they can do inside (and, with proper configuration, outside) the container. We'll cover this properly in the Docker security chapter — but notice that the underlying concept, ownership and permissions, is pure Linux.

### Processes and services

A **process** is a running instance of a program. Every container is, fundamentally, one or more Linux processes running with a restricted, isolated view of the system (we'll unpack exactly how in the Container Fundamentals chapter).

```bash
ps aux          # list running processes
kill 1234       # send a termination signal to process ID 1234
```

On a traditional Linux server, long-running programs (a database, a web server) are usually managed by `systemd`, using `systemctl`:

```bash
systemctl status postgresql
systemctl restart nginx
systemctl enable myapp     # start automatically on boot
```

**Why this matters later:** when we get to Kubernetes, the `kubelet` on each node is effectively doing a more sophisticated version of what `systemctl` does — starting, monitoring, and restarting things when they fail. Understanding "a supervisor process watches over other processes and restarts them" as a general Linux pattern makes Kubernetes's behavior much less mysterious.

### Environment variables and PATH

An **environment variable** is a named value available to a running process. This is the single most common way applications receive configuration — database URLs, API keys, feature flags — without hardcoding them into source code.

```bash
export DATABASE_URL=postgres://user:pass@localhost:5432/shopsphere
echo $DATABASE_URL
```

`PATH` is a special environment variable: a list of directories the shell searches through when you type a command name, so you can run `docker` instead of typing `/usr/bin/docker` every time.

**Where you'll see this constantly:** Docker `ENV` instructions, Kubernetes `env:` blocks in Pod specs, and CI/CD pipeline configuration are all, fundamentally, "set an environment variable for this process." It's the same concept, over and over, at every layer.

### SSH

**SSH (Secure Shell)** lets you securely log into and run commands on a remote machine. On day one at ShopSphere, this is literally how engineers deploy — SSH into the server, pull code, restart. Later, when we introduce Kubernetes, `kubectl exec` will feel very familiar — it's conceptually "get a shell inside a running container," the same underlying need, solved at a different layer.

### The command-line tools you'll actually use

You don't need mastery of these, but you need working fluency, because you'll use them constantly to inspect logs and debug running systems:

```bash
grep "ERROR" app.log             # search text for a pattern
grep -r "TODO" ./src             # search recursively through files

sed 's/foo/bar/' file.txt        # find-and-replace text (stream editor)

awk '{print $1}' access.log      # extract columns from structured text

find / -name "*.log"             # search the filesystem for files

curl https://api.shopsphere.com/health     # make an HTTP request from the command line
wget https://example.com/file.tar.gz       # download a file

cat /var/log/syslog | tail -n 50           # view the last 50 lines of a log file
```

A pattern worth internalizing now, because you'll use it in every troubleshooting chapter later:

```bash
docker logs backend | grep "ERROR" | tail -n 20
```

That's a **pipe** (`|`) — it takes the output of one command and feeds it as input to the next. `grep` filters, `tail` trims. Chaining small, focused tools together is the core Unix philosophy, and it's exactly how you'll investigate a broken Pod later: `kubectl logs` piped into `grep`, over and over.

**Interview question (beginner):** "How would you find all occurrences of the word 'timeout' in a log file, ignoring case?"
**Strong answer:** `grep -i "timeout" app.log` — the `-i` flag makes the search case-insensitive.

---

## Chapter 0.3 — Networking Essentials for DevOps

This is the part of the foundation that pays off the most later. Kubernetes networking, Ingress, Services — almost all of the confusion beginners have with Kubernetes traces back to a shaky grasp of these basics. So we're going to be precise here.

### IP addresses

**Simple explanation:** An IP address is like a postal address for a computer or device on a network — it's how data knows where to go.

**Proper definition:** An **IP address** is a numerical label assigned to a device on a network, used to identify and locate it for communication (IPv4 looks like `192.168.1.10`; IPv6 is longer and hex-based).

**Private vs public IP:**
- A **private IP** (e.g., `10.0.0.5`, `192.168.1.10`) is only reachable within a local network — your home Wi-Fi, or a company's internal network. Not directly reachable from the internet.
- A **public IP** is globally unique and reachable from anywhere on the internet.

**Where you'll see this:** every AWS VPC you build later will have private IP ranges. ShopSphere's database, for instance, should never have a public IP — there's no reason the internet should be able to reach it directly.

**localhost** (`127.0.0.1`) is a special address that always means "this machine, talking to itself." When you run the backend locally and it says "listening on `localhost:8000`," it means: only processes on this same machine can reach it. This distinction becomes very important once we containerize the app, because "localhost" *inside* a container means something different from "localhost" on your laptop — a classic beginner trap we'll walk through directly when we get to Docker networking.

### TCP, UDP, and ports

**TCP (Transmission Control Protocol)** is a reliable, connection-based way for two machines to exchange data — it guarantees delivery and correct order. Most web traffic, database connections, and API calls use TCP.

**UDP (User Datagram Protocol)** is faster but doesn't guarantee delivery or order — used where occasional loss is acceptable and speed matters more (video streaming, DNS lookups, some gaming traffic).

A **port** is a number that identifies a specific "channel" on a machine, so multiple services can share one IP address. PostgreSQL conventionally listens on port `5432`, Redis on `6379`, HTTP on `80`, HTTPS on `443`.

**Practical example:** when the ShopSphere backend connects to the database, it's not enough to know the database's IP — it also needs the port: `postgres://db-host:5432/shopsphere`.

**Interview question (intermediate):** "Why might a service use UDP instead of TCP?" — Because UDP has lower overhead (no handshake, no retransmission), which matters for latency-sensitive traffic that can tolerate some loss — DNS queries, live video, certain metrics-shipping protocols.

### DNS

**Simple explanation:** DNS is the phonebook of the internet — it turns human-readable names like `shopsphere.com` into IP addresses that computers actually use to connect.

**Proper definition:** **DNS (Domain Name System)** is a distributed naming system that resolves domain names to IP addresses.

**Why it exists:** IP addresses change (servers get replaced, IPs get reassigned) and are hard to remember. A stable name that points to a changing address solves both problems.

**Where you'll see it constantly:** Kubernetes has its own internal DNS system (**CoreDNS**) that lets Pods find each other by name instead of IP — because Pod IPs change every time a Pod restarts. This single idea — "give things a stable name because their address changes" — is the same reason Kubernetes Services exist, which we'll cover in depth in Part VI.

### HTTP and HTTPS

**HTTP (HyperText Transfer Protocol)** is the protocol web browsers and APIs use to request and send data — "give me this webpage," "here is the JSON response." **HTTPS** is the same thing, encrypted with TLS, so traffic can't be read or tampered with in transit.

When the ShopSphere frontend calls the backend API, it's making an HTTP request. When a customer's browser loads the storefront, that's HTTPS. Later, when we set up Ingress and TLS certificates, we're solving the specific problem of "how does encrypted traffic from the real internet reach my cluster."

### CIDR notation

You'll see addresses written like `10.0.0.0/16` constantly once we get to AWS VPCs and Kubernetes networking. This is **CIDR (Classless Inter-Domain Routing)** notation — it describes a *range* of IP addresses, not just one.

The `/16` is the important part: it says how many bits of the address are "fixed" (the network portion), leaving the rest free for individual devices. A smaller number after the slash means a *bigger* range:

- `/24` → 256 addresses (a small subnet — enough for one AZ's worth of servers)
- `/16` → 65,536 addresses (a whole VPC, typically)

**Why it matters:** when you design an AWS VPC in the Terraform chapter, you'll be deciding how to carve up address space between public subnets (things with internet access, like load balancers) and private subnets (things that shouldn't be directly reachable, like databases and worker nodes). Getting CIDR ranges wrong is a genuinely common real-world mistake — running out of IP addresses in a subnet because it was sized too small is a real incident category.

### Routing, NAT, and firewalls

**Routing** is how network traffic finds its way from a source to a destination across multiple networks — routers/route tables decide "traffic to this range of addresses goes this direction."

**NAT (Network Address Translation)** lets machines with private IPs communicate with the public internet by translating their private address to a shared public one at the network boundary. This is exactly what an AWS **NAT Gateway** does for private subnets later — it's the mechanism that lets a worker node with no public IP still download a Docker image from the internet.

**Firewalls** control what traffic is allowed in or out, based on rules (source, destination, port, protocol). In AWS this shows up as **Security Groups** (attached to resources, stateful) and **Network ACLs** (attached to subnets). In Kubernetes, this same concept reappears as **NetworkPolicy** — "which Pods are allowed to talk to which other Pods." Same underlying idea — *allow-list traffic based on rules* — implemented at three different layers of the stack. Recognizing that repetition is genuinely useful; interviewers like candidates who see it.

### Load balancing

**Simple explanation:** a load balancer sits in front of multiple copies of your application and spreads incoming traffic across them, so no single server gets overwhelmed and so traffic doesn't go to a server that's down.

**Proper definition:** a **load balancer** distributes network or application traffic across multiple backend targets based on a defined algorithm (round-robin, least-connections, etc.), typically also performing health checks to route traffic away from unhealthy targets.

**Why it exists:** if ShopSphere runs three copies of the backend API for reliability and capacity, something needs to decide which of the three handles each incoming request — and needs to stop sending traffic to a copy that's crashed.

**Where you'll encounter it:** this idea appears at every layer we're going to build: an AWS Application Load Balancer in front of the cluster, a Kubernetes Service load-balancing across Pod replicas, and Ingress routing based on hostnames and paths. Every one of these is a variation on the same core idea.

```text
              Client Requests
                    |
                    v
              Load Balancer
             /       |       \
        Server A  Server B  Server C
        (healthy) (healthy) (down — skipped)
```

**Interview question (intermediate):** "How does a load balancer know not to send traffic to a broken server?" — Health checks: the load balancer periodically probes each backend (an HTTP request to a health endpoint, or a TCP connection attempt) and stops routing to any target that fails. You'll meet this exact concept again, almost unchanged, as **Kubernetes readiness probes** in Part VI.

---

## Chapter 0.4 — Checkpoint

Before we move on to containers, you should be able to answer these without looking back:

**Beginner:**
1. What's the difference between a private and a public IP address?
2. What does a port number do?
3. What's the difference between TCP and UDP?

**Intermediate:**
4. Why does DNS exist, and where do you think it might show up inside a Kubernetes cluster (we haven't covered this yet — take a guess based on what DNS does)?
5. What's the difference between routing and NAT?
6. Why might an application be configured entirely through environment variables instead of a config file baked into the code?

**Scenario:**
7. A teammate says: "The database server has no public IP, so it's completely unreachable from the internet — we're fine." Is that reasoning airtight? *(Hint: think about what a NAT Gateway or a compromised machine inside the private network could do. We'll return to this exact question when we cover NetworkPolicy and defense-in-depth in Part VIII.)*

---

### Hands-On Lab 0.1 — Get comfortable with the tools

**Objective:** Build baseline command-line fluency before we touch Docker.

**Prerequisites:** A Linux, macOS, or WSL2 terminal.

**Steps:**
1. Create a fake log file and practice filtering it:
   ```bash
   printf "INFO started\nERROR db timeout\nINFO handled request\nERROR redis unreachable\n" > app.log
   grep "ERROR" app.log
   ```
2. Check what's listening on port 5432 on your machine (it's fine if nothing is):
   ```bash
   curl -v telnet://localhost:5432 2>&1 | head -5
   ```
3. Set and read an environment variable:
   ```bash
   export APP_ENV=development
   echo "Running in $APP_ENV mode"
   ```

**Expected result:** `grep` prints the two ERROR lines; the `curl` attempt either connects or fails cleanly (both are informative); the echo prints "Running in development mode."

**Verification:** You should be able to explain, in your own words, what each command actually did — not just that it "worked."

**Challenge:** Using `grep`, `awk`, and a pipe, count how many `ERROR` lines are in `app.log` without opening the file. *(Solution: `grep -c "ERROR" app.log`, or `grep "ERROR" app.log | wc -l`.)*

---

*End of Part I. Part II picks up with Container Fundamentals — what a container actually is under the hood (namespaces, cgroups, the overlay filesystem), why virtualization alone wasn't enough for ShopSphere's problems, and the first Dockerfile for the ShopSphere backend.*
