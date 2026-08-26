# The ShopSphere DevOps Book
## Part IV — Docker Networking, Storage, and Compose

---

### Where we left off

You have a solid, production-quality Dockerfile for ShopSphere's backend. But right now it's one container, running alone. A real application isn't one container — it's several, needing to talk to each other and to persist data safely. That's what this part solves.

---

## Chapter 3.1 — Docker Networking

### The problem, stated plainly

ShopSphere needs: a frontend container that can reach the backend container, and a backend container that can reach a PostgreSQL container and a Redis container. Recall from Part II that every container gets its own **network namespace** — its own private, isolated network stack. So by default, containers can't see each other at all. We need something that connects them deliberately.

### Network drivers

**Bridge network.** This is Docker's default. A **bridge network** is a private, isolated virtual network on the host — Docker creates a virtual switch, and containers attached to the same bridge network can reach each other by IP (and, on a *user-defined* bridge, by container name — more on that in a moment). Containers on different bridge networks cannot reach each other unless explicitly connected.

```text
                Host Machine
     +------------------------------------+
     |         docker0 (bridge)            |
     |     /              |            \    |
     | Frontend        Backend        (nothing else
     | 172.17.0.2      172.17.0.3      attached)
     +------------------------------------+
```

**Host network.** A container using `--network host` skips network isolation entirely and shares the host's network namespace directly — the container's `localhost` *is* the host's `localhost`. This removes the network isolation layer of containers, so it's used sparingly — mostly for specific performance-sensitive or low-level networking tools, not typical application workloads.

**None network.** `--network none` gives the container no network access at all — it only has the loopback interface. Used for workloads that intentionally shouldn't reach any network, as an extra isolation guarantee.

**User-defined networks.** This is what you actually want for a multi-container app like ShopSphere, and it behaves meaningfully differently from the default bridge:

```bash
docker network create shopsphere-net
docker run -d --name backend --network shopsphere-net shopsphere-backend:v1
docker run -d --name db --network shopsphere-net postgres:16
```

**Why this matters — container DNS.** On a **user-defined** network (unlike the default bridge network), Docker runs an embedded DNS server that lets containers reach each other **by container name**, automatically. The backend can connect to `postgres://db:5432/shopsphere` — using the literal name `db` — instead of needing to know a container's IP address, which can change every time it restarts.

**Interview question (intermediate):** "Why should you avoid using Docker's default bridge network for a multi-container application, and use a user-defined network instead?" — The default bridge network doesn't provide automatic DNS resolution between containers by name — you'd have to hardcode or discover IP addresses manually, which is fragile since container IPs can change. A user-defined bridge network gives containers reliable name-based service discovery.

This is worth pausing on, because it's the exact same underlying idea — "give things a stable *name* because their IP changes" — that we flagged for real DNS back in Part I, and it's the same idea that reappears as Kubernetes's **CoreDNS** and **Services** in Part VI. You're going to see this pattern at every layer of this stack. Recognize it.

### Container-to-container communication and port mapping, revisited

Recall from Part III: `-p 5432:5432` maps a **host** port to a **container** port, so things *outside* Docker (like your laptop's browser, or a debugging tool) can reach the container. But containers on the same user-defined network talking to *each other* don't need `-p` at all — they can reach each other directly, container-to-container, over the internal network, on the container's actual listening port. Port publishing (`-p`) is only about the outside world reaching in.

```text
   Your Laptop (outside Docker)
            |
     needs -p 8000:8000
            |
            v
   +--------------------------------------------+
   |            shopsphere-net (bridge)           |
   |                                               |
   |   Frontend ---> Backend ---> Postgres          |
   |   (talks to "backend"   (talks to "db"          |
   |    by name, no -p        by name, no -p          |
   |    needed — internal)    needed — internal)       |
   +--------------------------------------------+
```

**Common mistake, and the classic beginner trap we flagged back in Part I:** a developer's backend code has `DATABASE_URL=postgres://localhost:5432/shopsphere`, and it works fine when run directly on their laptop. The moment it's containerized, this breaks — because inside the backend container, `localhost` now refers to the backend container *itself*, not the separate database container. The fix is exactly the DNS-by-name pattern above: `postgres://db:5432/shopsphere`, using the container's network name, not `localhost`.

**Interview question (beginner, but genuinely commonly missed):** "A containerized app that worked fine when run directly on a developer's machine can't connect to its database after being containerized, even though the database container is running. What's the most likely cause?" — The application is still configured to connect to `localhost`, which inside a container refers to the container itself, not a separate database container — it needs to use the database container's name (on a shared user-defined network) or a proper service address instead.

### Network isolation as a design tool

Because containers on different networks can't reach each other by default, network segmentation is also a real security tool. For example, you might put the database on a network that only the backend can reach, keeping the frontend container — which faces more direct exposure to user input — unable to talk to the database directly at all, even if compromised. This is the container-level version of the exact same idea we'll formalize with Kubernetes **NetworkPolicy** in Part VIII, and with AWS Security Groups in Part XI. Same underlying principle, three different layers.

---

## Chapter 3.2 — Docker Storage

### Why containers are disposable

This is a mindset shift worth being explicit about. A container's own filesystem — the writable layer from Part II — is meant to be **ephemeral**: when the container is removed, that writable layer is gone with it. This is intentional, not a limitation. It's what makes containers easy to replace: if the backend container misbehaves, you kill it and start a fresh one from the same image, with total confidence it starts in a known-good state, because nothing about its runtime "history" carries over. This exact property — replace, don't repair — is the foundation of how Kubernetes handles failing Pods later: it doesn't try to fix a broken Pod, it just kills it and starts a fresh one from the same spec.

But this creates an obvious problem: ShopSphere's database absolutely cannot lose its data every time the container restarts. We need a way to keep *some* data outside the disposable container filesystem.

### Volumes vs. bind mounts vs. tmpfs

**Volumes.** A **Docker volume** is a storage location managed entirely by Docker, living outside any single container's writable layer, that can be attached to one or more containers. This is the recommended way to persist data in Docker.

```bash
docker volume create shopsphere-db-data
docker run -d --name db \
  --network shopsphere-net \
  -v shopsphere-db-data:/var/lib/postgresql/data \
  postgres:16
```

Now, even if you `docker rm -f db` and start a brand-new database container attached to the same volume, the actual data survives, because it never lived inside the container's disposable writable layer — it lived in the volume, managed independently by Docker.

**Bind mounts.** A **bind mount** maps a specific path *on the host machine's own filesystem* directly into the container.

```bash
docker run -d --name backend \
  -v $(pwd)/src:/app/src \
  shopsphere-backend:v1
```

**Why this matters for local development specifically:** during active development, you want code changes on your laptop to be reflected inside the running container immediately, without rebuilding the image every time. A bind mount gives you exactly that — it's an extremely common local-dev pattern, and one you'll use constantly in the Compose setup below. It's much less common in production, where you generally want the image itself to be the complete, immutable source of truth (we'll expand on this "don't rely on the host filesystem in production" theme heavily once we reach Kubernetes storage in Part VII).

**tmpfs mounts.** A **tmpfs mount** stores data only in the host's memory (RAM), never touching disk, and disappears entirely when the container stops. Used for sensitive, short-lived data you specifically don't want persisted anywhere — a good example is temporary decrypted secrets or session data you don't want lingering on disk.

```bash
docker run -d --tmpfs /app/tmp shopsphere-backend:v1
```

| | Volume | Bind Mount | tmpfs |
|---|---|---|---|
| Managed by | Docker | You (host filesystem) | Docker (in-memory) |
| Survives container removal | Yes | Yes (it's just a host path) | No |
| Typical use | Production persistent data | Local dev, live code editing | Sensitive short-lived data |

**Interview question (intermediate):** "Why is a named Docker volume generally preferred over a bind mount for a production database?" — A volume is managed by Docker independently of any specific host directory structure, is more portable across environments, and doesn't depend on a particular file layout existing on the host machine — a bind mount ties your container directly to the host's own filesystem paths and permissions, which is fragile and less portable across different servers.

---

## Chapter 3.3 — Docker Compose

### The problem, again

Running ShopSphere locally right now means manually running four separate `docker run` commands, in the right order, with the right network, the right volumes, and the right environment variables, remembered correctly every single time. That's fragile and doesn't scale past one person's memory. **Docker Compose** solves exactly this: it lets you define your entire multi-container application — services, networks, volumes, environment variables — in a single declarative file, and bring the whole thing up or down with one command.

**Simple explanation:** Compose is a way to describe "here's my whole app: these are the containers, here's how they're networked, here's what data they need to keep" in one file, so anyone on the team can run the entire thing with one command.

**Proper definition:** **Docker Compose** is a tool for defining and running multi-container Docker applications using a YAML configuration file (`docker-compose.yml` or `compose.yaml`), which describes services, networks, and volumes declaratively.

### Building ShopSphere's Compose file

Let's build this up piece by piece, the way you'd actually develop it.

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://shopsphere:devpass@db:5432/shopsphere
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  worker:
    build: ./worker
    environment:
      DATABASE_URL: postgres://shopsphere:devpass@db:5432/shopsphere
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: shopsphere
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: shopsphere
    volumes:
      - shopsphere-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shopsphere"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    volumes:
      - shopsphere-cache-data:/data

volumes:
  shopsphere-db-data:
  shopsphere-cache-data:
```

Let's walk through what each concept is actually doing.

**Services.** Each top-level entry under `services:` — `frontend`, `backend`, `worker`, `db`, `cache` — becomes one container. This directly mirrors our target architecture from Part I: frontend, backend, worker, database, cache.

**Networks (implicit).** Notice we never manually created a network here — Compose automatically creates a single default network for the whole project and attaches every service to it. This is exactly the user-defined bridge network behavior from Chapter 3.1: every service can reach every other service **by its service name** — `backend` resolves `db`, `cache`, automatically, with zero manual DNS or IP configuration. This is arguably Compose's single biggest convenience over raw `docker run`.

**Volumes.** `shopsphere-db-data` and `shopsphere-cache-data` are named volumes, declared once at the bottom and referenced by the services that need them — same underlying concept as Chapter 3.2, just declared in one readable place instead of repeated `-v` flags across multiple commands.

**Environment variables.** Set directly under each service — this is how the backend and worker learn how to reach the database and cache, using the service names as hostnames (`db`, `cache`) exactly as DNS-by-name resolves them.

**depends_on.** This controls *startup order* — Compose starts `db` and `cache` before `backend`. But there's a crucial nuance here worth getting right, because it's a very common source of real bugs.

### depends_on is not the same as "is ready"

**Common mistake:** a plain `depends_on: [db]` only waits for the database *container to start* — not for PostgreSQL *inside* it to actually finish initializing and be ready to accept connections. A container can be "started" while the application inside it is still booting. If the backend tries to connect during that window, it fails — and depending on timing, this bug might not even show up consistently, making it maddening to debug.

**The fix** is exactly what we did above: a **healthcheck**, combined with `condition: service_healthy`.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U shopsphere"]
  interval: 5s
  timeout: 3s
  retries: 5
```

This tells Compose to actually run `pg_isready` inside the `db` container repeatedly, and only consider it truly "healthy" once that check succeeds — and `backend`'s `depends_on: db: condition: service_healthy` means Compose won't start the backend until that real health signal passes, not just until the container process exists. Hold onto this concept tightly — it is *exactly* the same problem, solved with the exact same idea, as Kubernetes **readiness probes** in Part VIII. You are learning that concept right now, a few chapters early, in a smaller, easier context.

**Interview question (intermediate):** "Why isn't `depends_on` alone enough to guarantee a backend service can successfully connect to its database on startup?" — `depends_on` only controls container start *order*, not application readiness inside the container — the database container can be running while PostgreSQL itself is still initializing. A healthcheck combined with `condition: service_healthy` is needed to actually wait for the dependency to be ready to accept connections, not just started.

### Compose commands you'll actually use

```bash
docker compose up              # build (if needed) and start everything, foreground
docker compose up -d           # same, but detached
docker compose down            # stop and remove containers, network (volumes kept by default)
docker compose down -v         # also remove named volumes — be careful, this deletes your data
docker compose logs -f backend # follow logs for one specific service
docker compose ps              # list this project's running services
docker compose build           # rebuild images without starting containers
docker compose exec backend sh # shell into a running service, same idea as `docker exec`
```

### Scaling with Compose

```bash
docker compose up -d --scale worker=3
```

This starts three copies of the `worker` service. It's a genuinely useful local-dev tool for testing how your app behaves with multiple concurrent workers — but it's important to be honest about its limits: Compose scaling is a single-machine convenience, not a production orchestration system. It doesn't handle a machine dying, doesn't redistribute load across multiple physical hosts, and doesn't automatically react to real load. That gap — "I need this to survive a machine failing, and scale based on actual demand, across many machines" — is precisely the problem Kubernetes exists to solve, and it's where we're headed next in Part VI.

---

## Chapter 3.4 — Local vs. Production, Named Explicitly

This is a good moment to apply the "local vs. production" distinction the book promised up front, because Compose is a perfect first example of it.

| | Local Development (Compose) | Production |
|---|---|---|
| Database | `postgres:16` container, disposable | Managed service (e.g., Amazon RDS) — we'll cover why in Part XI |
| Scaling | `--scale worker=3`, single machine | Kubernetes, across multiple nodes, reacting to real load |
| Networking | Compose's automatic single-machine bridge network | Multi-node cluster networking, load balancers, Ingress |
| Startup ordering | `depends_on` + healthchecks | Kubernetes readiness probes + retry logic in the app itself |
| Config/secrets | Plaintext `environment:` in a file on your laptop | Kubernetes Secrets, AWS Secrets Manager — never plaintext in source |

We are **not** going to tell you to run Compose in production. It's a genuinely excellent tool for exactly what it's for: a fast, reproducible local development environment, and this is the correct, realistic way ShopSphere's engineers would actually work day-to-day before code ever reaches a cluster.

---

## Chapter 3.5 — Checkpoint

**Beginner:**
1. Why can't two containers reach each other over the network by default?
2. What's the difference between a Docker volume and a bind mount?

**Intermediate:**
3. Why does the default bridge network not support container name resolution, while a user-defined bridge network does?
4. Why is `depends_on` alone insufficient to guarantee a dependent service is actually ready?

**Advanced:**
5. Explain why containers are described as "disposable," and how that design principle connects to why persistent data needs volumes in the first place.
6. What are the real limits of `docker compose --scale`, and what specific gap does that leave that Kubernetes fills?

**Scenario:**
7. A backend container can't connect to a database container even though both are running and both are on the same Compose project. What are the first three things you'd check, in order?

---

### Hands-On Lab 3.1 — Bring up ShopSphere locally with Compose

**Objective:** Get the full ShopSphere stack (frontend, backend, worker, db, cache) running locally with one command, and prove data survives a container restart.

**Prerequisites:** Docker and Docker Compose installed; the `backend` Dockerfile from Part III (a minimal `frontend` and `worker` can be simple placeholder Flask/Node apps for this lab — the specific app code isn't the point).

**Steps:**

1. Create the project structure and the `docker-compose.yml` from Chapter 3.3 at the root.

2. Bring everything up:
   ```bash
   docker compose up -d
   docker compose ps
   ```

3. Confirm service-name DNS resolution works from inside the backend:
   ```bash
   docker compose exec backend sh -c "getent hosts db"
   ```

4. Prove data persistence: create a table, then destroy and recreate the `db` container.
   ```bash
   docker compose exec db psql -U shopsphere -c "CREATE TABLE test_survival (id int);"
   docker compose rm -f -s db
   docker compose up -d db
   docker compose exec db psql -U shopsphere -c "\dt"
   ```

5. Clean up (keeping data):
   ```bash
   docker compose down
   ```

**Expected result:** all services show as running/healthy in step 2; step 3 resolves an IP address for `db` by name; step 4's final `\dt` still shows `test_survival`, proving the volume outlived the container.

**Verification:** the table surviving a full container removal and recreation is the actual proof — not just that `docker compose up` didn't error.

**Troubleshooting:** if the backend can't reach `db`, check that `DATABASE_URL` uses the service name `db`, not `localhost` (Chapter 3.1's classic trap) — and check `docker compose logs db` to confirm PostgreSQL actually finished initializing.

**Challenge:** modify the Compose file so the `worker` service only starts once `backend` reports healthy (add a healthcheck to `backend` first — think about what a meaningful health check for an API looks like, then wire up `condition: service_healthy` the same way we did for `db`).

---

*End of Part IV. Part V covers Docker Security (root vs. non-root, the Docker socket, privileged containers, image scanning) and Container Registries (Docker Hub, private registries, and AWS ECR) — the last stop before Kubernetes enters the story.*
