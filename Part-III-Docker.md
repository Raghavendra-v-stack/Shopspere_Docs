# The ShopSphere DevOps Book
## Part III — Docker

---

### Where we left off

You now understand what a container actually is under the hood — namespaces, cgroups, OverlayFS, and the image/runtime/engine distinction. Now it's time to actually use Docker to solve ShopSphere's real problem: package the backend API so it runs identically everywhere.

---

## Chapter 2.1 — Docker's Architecture

**Simple explanation:** when you type a `docker` command, you're not running the container yourself — you're talking to a background service that does the actual work on your behalf.

**Proper definition:** Docker uses a **client-server architecture**. The `docker` CLI (the **client**) sends commands over an API to the **Docker daemon** (`dockerd`, the **server**), which does the heavy lifting — building images, running containers, managing networks and volumes.

```text
   You type:  docker run shopsphere-backend
                        |
                        v
                  Docker CLI (client)
                        |
              REST API over a socket
                        |
                        v
                Docker daemon (dockerd)
                        |
                        v
                   containerd
                        |
                        v
                      runc
                        |
                        v
              Linux namespaces + cgroups
                        |
                        v
                  Running Container
```

**Why it's built this way:** separating the client from the daemon means the daemon can keep running (and keep your containers running) even if you close your terminal. It also means the CLI can talk to a *remote* Docker daemon — for example, in CI systems, or when the CLI runs on your laptop but the daemon runs on a build server.

- **Docker CLI** — the `docker` command you type. Just a thin client; it does no actual container work itself.
- **Docker daemon (dockerd)** — the long-running background process that receives API requests, manages images, builds, networks, and volumes, and orchestrates the lower layers.
- **containerd** — as covered in Part II, a higher-level container runtime that manages the container lifecycle and image storage, called by the daemon.
- **runc** — the low-level OCI runtime that actually creates the namespaces/cgroups and starts the container process.
- **Registry** — a server that stores and distributes container images (Docker Hub, AWS ECR, etc.) — covered fully in Chapter 5.

**How you'd encounter this at a real company:** "the Docker daemon isn't running" is one of the single most common first-day errors (`Cannot connect to the Docker daemon. Is the docker daemon running?`). Understanding that `docker` (the CLI) and `dockerd` (the daemon) are two separate things immediately tells you what to check: is the daemon actually started?

**Interview question (beginner):** "What's the difference between the Docker client and the Docker daemon?" — The client is the CLI tool you interact with; the daemon is the background service that actually builds images and runs containers. The client sends API requests to the daemon.

---

## Chapter 2.2 — The Docker Commands You'll Actually Use

We're not going to memorize these as a cheat sheet — we're going to use each one for a real reason, on a real container, so the *purpose* sticks.

### Inspecting your environment

```bash
docker version     # client and daemon version info — useful for "why doesn't this work" debugging
docker info         # daemon-wide info: number of containers, storage driver, etc.
```

### Pulling and inspecting images

```bash
docker pull postgres:16          # download an image from a registry, without running it
docker images                    # list images stored locally
docker image ls                  # identical to `docker images` — both exist for historical reasons
```

**Real-world usage:** before ShopSphere's backend can talk to a database locally, you'll pull the official `postgres` image rather than installing PostgreSQL directly on your laptop — one of the first tangible "this solves my actual problem" moments with Docker.

**Common mistake:** pulling `postgres:latest` and being surprised later when a teammate's `latest` resolves to a different, newer version than yours. We'll cover why explicit tags matter in the Images chapter.

### Running containers

```bash
docker run postgres:16                          # run in the foreground, attached to your terminal
docker run -d postgres:16                        # run detached (in the background)
docker run -d --name shopsphere-db postgres:16    # give it a memorable name instead of a random one
docker run -d -p 5432:5432 postgres:16            # map host port 5432 -> container port 5432
docker run -d -e POSTGRES_PASSWORD=devpass postgres:16   # set an environment variable
```

Let's break down `-p 5432:5432`, because it trips up almost everyone at first: it's `HOST_PORT:CONTAINER_PORT`. The container has its own isolated network namespace (Part II) — nothing outside it can reach it *unless* you explicitly publish a port, mapping a port on your actual machine to a port inside the container's private network. Forgetting `-p` and then wondering "why can't I connect to my container" is one of the single most common early Docker mistakes.

### Viewing running containers

```bash
docker ps               # list running containers
docker ps -a             # list ALL containers, including stopped ones
```

**Common mistake:** running `docker ps`, seeing nothing, and concluding "my container isn't there" — when actually it crashed and stopped seconds after starting. `docker ps -a` would show it, with an exit code, which is your first debugging clue.

### Managing container state

```bash
docker stop shopsphere-db      # send SIGTERM, then SIGKILL after a grace period — graceful stop
docker start shopsphere-db     # start a previously-stopped container again
docker restart shopsphere-db   # stop, then start
docker kill shopsphere-db      # send SIGKILL immediately — no graceful shutdown
docker pause shopsphere-db     # freeze all processes inside (cgroup freezer, from Part II)
docker unpause shopsphere-db   # unfreeze
```

**Why `stop` vs `kill` matters in production:** `docker stop` gives the application a chance to finish in-flight work and shut down cleanly (close database connections, finish processing a request) before being force-killed. `docker kill` gives it none. This exact distinction — graceful termination with a grace period vs. an immediate kill — reappears, almost unchanged, as Kubernetes Pod termination behavior in Part VI. Learn it once here.

### Removing things

```bash
docker rm shopsphere-db          # remove a stopped container
docker rm -f shopsphere-db        # force-remove, even if running
docker rmi postgres:16            # remove an image
```

**Common mistake:** trying to `docker rmi` an image that's still used by an existing (even stopped) container, and getting a "conflict" error. The fix is to remove the container(s) first.

### Looking inside a running container

```bash
docker exec -it shopsphere-db psql -U postgres        # run a new command inside a running container
docker logs shopsphere-db                              # view stdout/stderr from the container's main process
docker logs -f shopsphere-db                            # follow logs in real time (like `tail -f`)
docker inspect shopsphere-db                            # full JSON metadata: IP, mounts, env vars, config
docker stats                                             # live CPU/memory usage across containers
```

`docker exec -it` deserves a proper breakdown: `-i` keeps STDIN open (interactive), `-t` allocates a pseudo-terminal (so it behaves like a normal shell session). Together, `-it` is what you use to essentially "SSH into" a running container — the container equivalent of the SSH access we discussed in Part I.

**How you'd encounter this in a real company:** `docker logs` and `docker exec` are the two commands you'll reach for constantly when debugging "why is this container misbehaving" — before you ever touch Kubernetes's equivalents (`kubectl logs`, `kubectl exec`), which work almost identically.

### A few more utility commands

```bash
docker cp shopsphere-db:/var/lib/postgresql/data ./backup   # copy files out of a container
docker rename old-name new-name                              # rename a container
```

**Interview question (intermediate):** "You run `docker ps` and see nothing, but you know you started a container ten minutes ago. What's your next step?" — Run `docker ps -a` to see stopped containers, then `docker logs <container>` to see why it exited, and check the exit code with `docker inspect` — a non-zero exit code usually means the application crashed on startup.

---

## Chapter 2.3 — Building the First (Bad) Dockerfile

Let's actually containerize ShopSphere's backend — a Python API. Here's what a beginner would typically write first, and it's a genuinely useful exercise to write this badly on purpose, so the fixes actually mean something.

**ShopSphere backend, before containers, on your laptop:**

```text
shopsphere-backend/
├── app.py
├── requirements.txt
```

### Attempt 1 — the naive version

```dockerfile
FROM python
COPY . .
RUN pip install -r requirements.txt
CMD python app.py
```

This will actually work. It will also cause you real problems the moment more than one person uses it. Let's go through the Dockerfile instructions properly, and then come back and fix every one of these mistakes.

### The core Dockerfile instructions

- **`FROM`** — sets the base image everything else builds on top of. Every Dockerfile starts with one.
- **`RUN`** — executes a command *at build time*, and its result becomes a new layer (Part II). Used for installing dependencies, compiling code, etc.
- **`COPY`** — copies files from your local build context into the image.
- **`ADD`** — similar to `COPY`, but with extra (often surprising) behavior: it can automatically extract local `.tar` archives, and can fetch remote URLs.
- **`WORKDIR`** — sets the working directory for subsequent instructions (like `cd`, but persistent for the rest of the file).
- **`ENV`** — sets an environment variable that persists into the running container.
- **`ARG`** — defines a build-time-only variable, available during `docker build` but *not* present in the final running container.
- **`EXPOSE`** — documents which port the container listens on. It does **not** actually publish the port — that's still `-p` at `docker run` time.
- **`USER`** — sets which user subsequent instructions (and the final container process) run as.
- **`CMD`** — the default command run when the container starts, unless overridden.
- **`ENTRYPOINT`** — also defines what runs when the container starts, but is much harder to override at the command line — used when you want the container to always run as a specific executable.

### CMD vs ENTRYPOINT, precisely

This distinction is one of the most commonly-asked Docker interview questions, and most explanations online are genuinely unclear about it. Here's the precise version:

- If you only set `CMD`, the whole thing can be trivially overridden: `docker run myimage echo hello` completely replaces the `CMD`.
- If you set `ENTRYPOINT`, that command **always** runs — anything you add on the `docker run` command line is passed to it as *arguments*, not a replacement.
- The common production pattern combines both: `ENTRYPOINT` defines the fixed executable, `CMD` supplies default arguments to it, which *can* be overridden.

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

Running `docker run shopsphere-backend` executes `python app.py --port 8000`. Running `docker run shopsphere-backend --port 9000` executes `python app.py --port 9000` — only the `CMD` portion got replaced; the `ENTRYPOINT` stayed fixed. This pattern is genuinely common for CLI-style tools and for images where you want a guaranteed, un-skippable startup behavior.

**Interview question (intermediate):** "What's the difference between CMD and ENTRYPOINT?" — Both define the container's default startup command, but `CMD` is fully replaceable from the `docker run` command line, while `ENTRYPOINT` is not — arguments passed to `docker run` are appended to `ENTRYPOINT` rather than replacing it. They're commonly combined, with `ENTRYPOINT` as the fixed executable and `CMD` as its default, overridable arguments.

### COPY vs ADD, precisely

**Interview question (beginner):** "When should you use `ADD` instead of `COPY`?" — Almost never, by convention. `COPY` does one thing predictably: copy files in. `ADD`'s extra behaviors (auto-extracting archives, fetching remote URLs) are surprising and rarely what you actually want — remote-URL fetching in particular bypasses Docker's build cache and offers no error handling if the download fails. Docker's own official guidance is to prefer `COPY` unless you specifically need `ADD`'s archive-extraction behavior.

### ARG vs ENV, precisely

`ARG` values exist only during the build — useful for things like choosing a version number to install, without that detail leaking into the final image. `ENV` values persist into the running container and are visible to the application and to anyone who runs `docker inspect` or `docker exec env`.

**A genuinely important security point:** never put a secret (a password, an API key) in `ARG` *or* `ENV`, because both are readable after the fact — `ARG` values are visible in the image's build history (`docker history`), and `ENV` values are visible to anyone with `docker inspect` access or shell access inside the container. We'll cover the correct way to handle secrets properly in the Docker Security chapter (Chapter 5) and again in Kubernetes Secrets (Part VII) — but plant this warning now, because it's a real, common production mistake.

### EXPOSE vs actually publishing a port

**Interview question (intermediate, and a common trap):** "If a Dockerfile has `EXPOSE 8000`, can other machines now reach the container on port 8000?" — No. `EXPOSE` is documentation for humans and tooling (and it's used by tools like `docker run -P` to auto-publish), but it doesn't itself open any network path. You still need `-p 8000:8000` at `docker run` time (or the Kubernetes equivalent, a Service, which we'll get to) to actually make the port reachable.

---

## Chapter 2.4 — Fixing the Dockerfile, One Problem at a Time

Now let's go back to Attempt 1 and fix every real issue in it, explaining why each one matters.

```dockerfile
FROM python
COPY . .
RUN pip install -r requirements.txt
CMD python app.py
```

**Problem 1 — `FROM python` with no tag.** This pulls whatever `python:latest` currently resolves to — which changes over time, is a large, full-OS-based image (hundreds of MB, with tons of packages your app doesn't need), and isn't reproducible: your teammate building the same Dockerfile next month might get a different Python version than you did today.

**Fix:** pin an explicit, minimal base image.

```dockerfile
FROM python:3.12-slim
```

`slim` variants strip out documentation, compilers, and other build-only tooling not needed at runtime — dramatically smaller than the full image.

**Problem 2 — `COPY . .` before `RUN pip install`.** This is a build-caching mistake, not a functional bug, but it matters a lot in practice. Remember from Part II: each Dockerfile instruction creates a cacheable layer, and Docker reuses a cached layer only if nothing in it (or before it) changed. Because `COPY . .` copies your *entire* source tree — including files that change on every single commit — this layer, and every layer after it (including the expensive `pip install`), gets invalidated on every single code change, even a one-line change to a comment.

**Fix:** copy only the dependency manifest first, install dependencies, *then* copy the rest of the source.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

Now, editing application code invalidates only the final `COPY . .` layer — the (often slow) dependency-install layer stays cached, because `requirements.txt` didn't change. This alone can turn a multi-minute rebuild into a few seconds.

**Problem 3 — running as root.** By default, without a `USER` instruction, the process inside the container runs as root. Combined with the earlier security warning about kernel-sharing (Part II), this means that if an attacker finds a way to exploit the application, they get root inside the container — a meaningfully worse starting position for them to try to escalate further, compared to an unprivileged user.

**Fix:** create and switch to a non-root user.

```dockerfile
RUN useradd --create-home appuser
USER appuser
```

**Problem 4 — using the shell form of `CMD`.** `CMD python app.py` runs through a shell (`/bin/sh -c "python app.py"`), which means your actual application isn't PID 1 — the shell is, with your app as a child process. This matters because Linux only delivers termination signals (like the `SIGTERM` from `docker stop`) directly to PID 1; a shell wrapping your process may not forward that signal properly, meaning your app never gets a chance to shut down gracefully and just gets hard-killed after the grace period expires.

**Fix:** use the exec form (a JSON array), which runs your process directly as PID 1, with no shell in between.

```dockerfile
CMD ["python", "app.py"]
```

**Problem 5 — no `.dockerignore`.** `COPY . .` will happily copy your local `.git` folder, `__pycache__`, `.env` files with secrets in them, virtual environments, and anything else sitting in your project directory — bloating the image, slowing the build, and in the `.env` case, potentially baking secrets directly into the image.

**Fix:** add a `.dockerignore` file, which works exactly like `.gitignore`:

```text
.git
__pycache__
*.pyc
.env
.venv
node_modules
```

**Problem 6 — no health signal, no clear entrypoint boundary.** We'll actually address this properly with Kubernetes probes in Part VIII, but even at the plain-Docker level, a production image should make it easy to check "is this thing actually healthy," not just "is the process still running."

### Attempt 2 — the production-quality version

```dockerfile
# ---- Base image: explicit version, minimal footprint ----
FROM python:3.12-slim

# ---- Metadata and working directory ----
LABEL maintainer="devops@shopsphere.example"
WORKDIR /app

# ---- Install dependencies first, so this layer is cached
#      independently of application code changes ----
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ---- Now copy application code ----
COPY . .

# ---- Create and switch to a non-root user ----
RUN useradd --create-home --shell /bin/bash appuser \
    && chown -R appuser:appuser /app
USER appuser

# ---- Document the port (does not publish it) ----
EXPOSE 8000

# ---- Exec form: app runs as PID 1, receives signals directly ----
CMD ["python", "app.py"]
```

This is a genuinely reasonable production Dockerfile for a small Python service. It's explicit about its base image, builds efficiently by respecting the layer cache, runs as a non-root user, and terminates cleanly.

---

## Chapter 2.5 — Multi-Stage Builds and Real Image Optimization

There's one more level of Dockerfile quality worth understanding now, because it comes up constantly in interviews and in real production images: **multi-stage builds**.

**Simple explanation:** sometimes you need heavy tools (compilers, build dependencies) to *build* your application, but you don't want any of that bulk sitting in the final image that actually runs in production. A multi-stage build lets you use one temporary image to do the building, and copy only the finished result into a clean, minimal final image.

**Why it exists:** imagine ShopSphere's backend needed to compile a native extension. You'd need `gcc` and build headers during the build — but including a full C compiler toolchain in your production image is pure waste: it makes the image bigger, slower to pull, and gives an attacker more tools to work with if they ever get a shell inside your container.

```dockerfile
# ---- Stage 1: build ----
FROM python:3.12 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/install -r requirements.txt

# ---- Stage 2: final, minimal runtime image ----
FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local/lib/python3.12/site-packages
COPY . .
RUN useradd --create-home appuser
USER appuser
EXPOSE 8000
CMD ["python", "app.py"]
```

Only the `--from=builder` copied artifacts make it into the final image — the full build toolchain from Stage 1 never does.

**Distroless images**, as an even more extreme version of this idea, go further: instead of a "slim" OS with a shell and package manager, a **distroless** base image contains *only* your application's runtime dependencies — no shell, no package manager, nothing an attacker could use even if they got code execution. Google's `gcr.io/distroless` images are the most common example. The tradeoff: debugging is harder (you can't `docker exec` into a shell that doesn't exist), so many teams use distroless for the final production image while keeping a `-debug` variant with a shell available for troubleshooting.

**Interview question (advanced):** "Why would a team use a multi-stage Docker build?" — To keep build-time-only tools and dependencies (compilers, dev headers, test frameworks) out of the final production image entirely — reducing image size, reducing the attack surface, and keeping the shipped artifact limited to exactly what's needed at runtime.

---

## Chapter 2.6 — Checkpoint

**Beginner:**
1. What does `docker ps -a` show that `docker ps` doesn't?
2. What does `-p 8000:8000` actually do?

**Intermediate:**
3. Explain, precisely, the difference between `CMD` and `ENTRYPOINT`.
4. Why does the order of `COPY` and `RUN` instructions in a Dockerfile affect build speed?
5. Why is running a container as root a security concern, given what you learned about namespaces in Part II?

**Advanced:**
6. What problem do multi-stage builds solve, and what's the tradeoff of taking that idea to its extreme with a distroless image?
7. Explain why using the shell form of `CMD` can cause a container to ignore `docker stop` and require a hard kill instead.

**Scenario:**
8. A teammate's Dockerfile has `ENV DB_PASSWORD=supersecret123`. They argue it's fine because "it's inside the container, not exposed to the internet." What's wrong with that reasoning?

---

### Hands-On Lab 2.1 — Build both versions and compare

**Objective:** Feel the difference between a naive and a production-quality Dockerfile directly, not just read about it.

**Prerequisites:** Docker installed; a simple Python app (a two-line Flask "hello world" is enough).

**Steps:**

1. Create a minimal app:
   ```bash
   mkdir shopsphere-backend && cd shopsphere-backend
   echo "flask" > requirements.txt
   cat > app.py << 'EOF'
   from flask import Flask
   app = Flask(__name__)

   @app.route("/")
   def health():
       return "ShopSphere backend OK"

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=8000)
   EOF
   ```

2. Build Attempt 1 (naive) and note the image size:
   ```bash
   cat > Dockerfile.naive << 'EOF'
   FROM python
   COPY . .
   RUN pip install -r requirements.txt
   CMD python app.py
   EOF
   docker build -f Dockerfile.naive -t shopsphere-backend:naive .
   docker images shopsphere-backend
   ```

3. Build Attempt 2 (production-quality, from Chapter 2.4) as `Dockerfile`, and compare:
   ```bash
   docker build -t shopsphere-backend:v1 .
   docker images | grep shopsphere-backend
   ```

4. Test the caching improvement — change a comment in `app.py`, rebuild both, and time them:
   ```bash
   time docker build -f Dockerfile.naive -t shopsphere-backend:naive .
   time docker build -t shopsphere-backend:v1 .
   ```

5. Confirm the container runs as a non-root user in the production version:
   ```bash
   docker run --rm shopsphere-backend:v1 whoami
   ```

**Expected result:** the production image is noticeably smaller; the rebuild after a code-only change is noticeably faster for the layer-cache-friendly version; `whoami` prints `appuser`, not `root`.

**Verification:** `docker images` size comparison, build time comparison, and the `whoami` output all confirm the fixes worked — not just that the Dockerfile "looks" better.

**Troubleshooting:** if `whoami` fails or still prints `root`, double check the `USER appuser` line comes *after* the `useradd` line and *before* the final `CMD`.

**Challenge:** convert this Dockerfile to a multi-stage build that installs dependencies in a `builder` stage and copies only the installed packages into a `python:3.12-slim` final stage, and confirm the final image is still smaller than Attempt 2. *(We covered the pattern in Chapter 2.5 — apply it yourself before checking back against that example.)*

---

*End of Part III (Chapters 2.1–2.6). Next, we continue Docker with networking and storage — how containers actually talk to each other and to the outside world, why containers are meant to be disposable, and how Docker Compose lets us run ShopSphere's frontend, backend, database, and Redis together as one coordinated local environment.*
