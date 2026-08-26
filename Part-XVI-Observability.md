# The ShopSphere DevOps Book
## Part XVI — Observability

---

### Where we left off

ShopSphere is deployed on real, Terraform-managed EKS infrastructure, with CI/CD automatically shipping changes. There's a significant gap remaining: right now, the only way to know something is wrong is a customer complaint. This Part closes that gap.

---

## Chapter 15.1 — Why Observability, and the Three Pillars

**Simple explanation:** observability is the ability to understand what's actually happening inside a running system, from the outside, without having to guess or add new debugging code every time something goes wrong.

**Why this matters more, not less, after everything built in this book.** Recall Chapter 5.1's motivations for Kubernetes in the first place: multiple replicas, automatic rescheduling, rolling deployments. All of that dynamism is exactly what makes observability *harder* than it was in ShopSphere's original one-server setup — you can no longer just SSH into "the server" and read a log file, because there might be six backend Pods right now, on three different nodes, and the specific Pod that handled a given failing request might already have been replaced by the time you go looking.

**Metrics, Logs, and Traces — why all three matter, and what each one is actually good at:**

```text
   METRICS                    LOGS                       TRACES
   "How much? How many?"      "What exactly happened,     "What was the full
                                in detail, on this          path of this ONE
   Numeric time series,        specific event?"            request, across
   aggregated over time                                    every service
                               Detailed, individual,        it touched?"
   Good for: dashboards,       timestamped event records
   alerting, spotting trends                                Good for: understanding
                               Good for: deep-diving         latency and failures
   Weak for: "why did          into one specific failure     in a request that
   THIS ONE request fail"                                    crosses multiple
                               Weak for: aggregate trends     services (frontend
                               across millions of events      -> backend -> db)
```

None of the three replaces the others. A metric dashboard tells you error rate spiked at 2:14pm; logs tell you the specific error message and stack trace from a request in that window; a trace tells you that request spent 4 of its 5 total seconds waiting on a slow database query, not in the backend's own code at all. Real incident response typically moves through all three, in roughly that order — the same "work outward from the most useful signal" instinct as Chapter 10.1's troubleshooting methodology, applied to observability data instead of live debugging commands.

---

## Chapter 15.2 — Metrics: Prometheus and Grafana

### Prometheus

**Simple explanation:** Prometheus is a monitoring system that periodically asks every service "what are your current numbers" (CPU usage, request count, error count, queue depth — whatever that service chooses to expose) and stores the answers as a time series, so you can later ask questions like "how did this number change over the last 6 hours."

**Proper definition:** **Prometheus** is a time-series monitoring system that **scrapes** (pulls, on a defined interval) metrics from configured targets over HTTP, stores them, and provides a query language (**PromQL**) for analysis and alerting.

**Why "pull" rather than "push" is worth knowing as a genuine design decision, not an arbitrary detail.** Prometheus reaches out to each target and pulls its current metrics, rather than every service pushing metrics to a central collector. This has a real, practical consequence directly relevant to Kubernetes: if a target simply isn't reachable to be scraped, that absence *itself* is a meaningful, detectable signal ("this Pod stopped reporting entirely") — distinct from the target actively reporting zero, which a push-based model can't as cleanly distinguish.

**How Prometheus discovers targets in Kubernetes specifically.** Recall the label-selector pattern from Chapter 6.3 and 6.5 — Prometheus uses **Kubernetes service discovery**, automatically finding Pods and Services to scrape based on labels/annotations, rather than needing a hand-maintained static list of targets — genuinely important in a system where Pods are constantly being created and destroyed (Chapter 5.1's core premise, once again).

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
    prometheus.io/path: "/metrics"
```

For a real application to be scraped meaningfully, it needs to expose a `/metrics` endpoint in Prometheus's text format — most languages have a well-maintained client library for this (`prometheus_client` for Python, for example) that ShopSphere's backend would use to expose request counts, latencies, and error counts directly.

### Grafana

**Simple explanation:** Grafana is the dashboard layer on top of Prometheus (and other data sources) — it doesn't collect or store metrics itself; it queries and visualizes data that already lives somewhere else.

**Proper definition:** **Grafana** is a visualization and dashboarding tool that queries data sources like Prometheus and renders it as graphs, tables, and alerting rules.

```text
   ShopSphere Backend Pods
        |  (expose /metrics)
        v
   Prometheus (scrapes, stores as time series)
        |
        v
   Grafana (queries Prometheus, renders dashboards)
```

### The Four Golden Signals

A widely-used, genuinely practical framework (popularized by Google's SRE practice) for what to actually put on a dashboard, worth knowing by name for interviews specifically:

- **Latency** — how long requests take.
- **Traffic** — how much demand the system is under (requests/second).
- **Errors** — the rate of failed requests.
- **Saturation** — how "full" the system is (CPU, memory, connection pool usage — directly connecting back to Chapter 8.1's resource requests/limits).

**Interview question (intermediate):** "What are the Four Golden Signals, and why are they a good starting point for a new dashboard rather than trying to graph everything available?" — Latency, traffic, errors, and saturation — together they give a fast, high-level read on whether a service is healthy and how much headroom it has, without drowning in low-value metrics; most real incidents show up clearly in at least one of these four before you need to dig into anything more granular.

### Alerting

Prometheus (via **Alertmanager**, its companion alerting component) can fire an alert when a PromQL expression crosses a threshold — for example, error rate above 5% for more than 2 minutes. **Why the "for more than 2 minutes" duration matters, connecting directly back to Chapter 8.2's probe-flapping caution:** an alert that fires on a single noisy data point creates exactly the same kind of self-inflicted, false-alarm churn as an overly aggressive liveness probe — a brief, genuinely transient blip shouldn't page anyone at 3 a.m. Good alerting design deliberately requires *sustained* abnormal behavior before notifying a human, the same underlying discipline as a well-tuned probe.

---

## Chapter 15.3 — Logs

### Application logs, container logs, and Kubernetes logs — the layers, precisely

**Application logs** are what your code itself writes (`log.info("order placed")`). **Container logs** are simply whatever the container's main process writes to **stdout/stderr** — recall Chapter 3's `docker logs` command, which reads exactly this; Kubernetes's `kubectl logs` (Chapter 6.2) works the same way, reading stdout/stderr from the container. **This is a deliberate, important design convention worth naming explicitly:** containerized applications are expected to log to stdout/stderr, *not* to a file inside the container's own disposable filesystem (recall Chapter 3.2 — a container's writable layer disappears when it's removed, so logs written only to a local file would vanish along with it).

### The problem centralized logging solves

`kubectl logs` reads logs from *one specific Pod, right now*. But recall the exact same dynamism problem from Chapter 15.1: by the time you're investigating an issue, the specific Pod that handled the failing request may already be gone, its logs gone with it (`kubectl logs --previous`, from Chapter 10.2, only gets you *one* prior restart, not a durable history). Production systems need logs collected and stored centrally, outside the lifecycle of any individual Pod, exactly the same underlying motivation as Chapter 7.3's PersistentVolume discussion, applied to log data instead of application data.

### The standard pattern: a log-shipping agent

```text
   Node
   +----------------------------------------------+
   |   backend Pod    frontend Pod    worker Pod     |
   |       |               |               |          |
   |       +---------------+---------------+           |
   |                       |  (stdout/stderr,            |
   |                       |   written to a known         |
   |                       |   location on the node          |
   |                       |   by the container runtime)      |
   |                       v                                    |
   |              Log agent (e.g. Fluent Bit)                    |
   |              runs as a DaemonSet — one per node               |
   +----------------------------------------------+
                          |
                          v
                Centralized log storage
             (e.g., Loki, or CloudWatch Logs)
```

**DaemonSet**, worth introducing by name here since this is its most common real-world use case: a **DaemonSet** ensures exactly one copy of a Pod runs on every node in the cluster — a natural fit for a log-shipping agent, which needs to run everywhere logs are being generated, rather than a fixed replica count the way a Deployment (Chapter 6.4) has.

### Loki

**Loki** is a log aggregation system designed to integrate tightly with Grafana and Prometheus's own operational model — notably, it indexes logs by a small set of labels (similar in spirit to Kubernetes's own label system) rather than indexing the full text content of every log line, which keeps it comparatively lightweight and cost-efficient compared to full-text-indexing alternatives, at the cost of somewhat less powerful arbitrary full-text search.

### CloudWatch Logs, as the AWS-native alternative

On EKS specifically, **CloudWatch Logs** is a genuinely common choice, integrating directly with the rest of AWS with comparatively little additional setup (via the CloudWatch Container Insights add-on, or a Fluent Bit configuration shipping to CloudWatch instead of Loki). The honest tradeoff worth naming: CloudWatch is operationally simpler on AWS specifically, but Prometheus/Grafana/Loki is portable across any Kubernetes environment and is often preferred by teams wanting a consistent observability stack regardless of cloud provider — a real, legitimate architectural choice, not a strictly correct-vs-incorrect one.

**Cost Warning:** CloudWatch Logs charges for ingestion and storage — genuinely easy to underestimate for a chatty application logging verbosely at `debug` level in production; setting an appropriate log retention period and log level (recall the `LOG_LEVEL` ConfigMap value from Chapter 6.7) is a real, practical cost control, not just a cleanliness concern.

---

## Chapter 15.4 — Traces

### The problem traces solve, precisely

Recall ShopSphere's architecture: a request might flow frontend → backend → database, and separately trigger an asynchronous worker job. If that request is slow, *which* of those hops is actually responsible? Metrics can show you each service's aggregate latency separately, but not this *one specific request's* actual path and per-hop timing. Logs, similarly, give you separate, disconnected log lines from each service, with no inherent link tying them together as "all part of the same request."

**Simple explanation:** distributed tracing tags a single request with a shared ID as it enters the system, and carries that ID through every service it touches, so you can later reconstruct the request's *entire* journey — and see exactly how much time was spent in each part of it — as one connected picture.

**Proper definition:** a **trace** represents one request's full path through a distributed system, composed of multiple **spans** — each span representing time spent in one specific service or operation, with parent/child relationships showing which spans triggered which others.

```text
   Trace: "checkout request abc123"

   [=== Frontend: 5000ms total ===============================]
      [== Backend API call: 4800ms ============================]
         [= DB query: 4200ms ========================]  <- the actual bottleneck
         [ Redis cache check: 50ms ]
         [ Worker enqueue: 30ms ]
```

This is a genuinely different, and often far more actionable, view than a metrics dashboard alone — it immediately shows the database query as the dominant cost in this specific slow request, rather than requiring you to separately correlate a backend latency spike with a database latency spike and *infer* the connection.

### OpenTelemetry

**What it is.** **OpenTelemetry (OTel)** is a vendor-neutral standard (and set of libraries) for generating and exporting traces (and also metrics and logs), letting an application be instrumented once and send data to whichever backend you choose (Jaeger, Grafana Tempo, a commercial APM vendor) without re-instrumenting the application if you later switch backends — directly analogous to the OCI standard from Part II, which let a container image work across different runtimes without needing to be rebuilt per-runtime; same underlying "agree on one standard interface, keep implementations swappable" motivation, one more time.

**Interview question (advanced):** "You have Prometheus dashboards showing an elevated error rate, but no clear indication of which specific part of the request path is failing. What would tracing add that metrics alone can't provide?" — Tracing reconstructs the full path of individual requests across every service they touch, showing exactly where time is spent and where failures occur within a *specific* request — metrics show aggregate trends across many requests, but can't by themselves show you the internal structure of any one particular request's journey through a multi-service system.

---

## Chapter 15.5 — Personal Lab vs. Production for Observability

| | Production | Personal Lab |
|---|---|---|
| Metrics | Prometheus + Grafana (self-hosted or managed), or CloudWatch | Prometheus + Grafana, self-hosted in-cluster — genuinely free to run locally |
| Logs | Centralized — Loki or CloudWatch Logs, with a retention policy | `kubectl logs`, or a lightweight local Loki instance for practicing the DaemonSet pattern |
| Tracing | OpenTelemetry + a tracing backend (Jaeger/Tempo) | OpenTelemetry + Jaeger locally — free, genuinely worth practicing the instrumentation itself |
| Alerting | Alertmanager routing to on-call tooling (PagerDuty, Slack) | Alertmanager configured, routed to a personal Slack webhook or just observed manually |

**Cost note specific to this Part:** unlike EKS or NAT Gateways, most of this Part's core tools (Prometheus, Grafana, Loki, Jaeger) are genuinely free and open-source, and run perfectly well inside your existing kind cluster or personal-lab EKS cluster with no separate licensing cost — the primary cost consideration here is really just the underlying compute/storage they consume, and (if used) CloudWatch's own ingestion/storage pricing as the one genuinely metered piece in this chapter.

---

## Chapter 15.6 — Checkpoint

**Beginner:**
1. What are the three pillars of observability, and what specific question does each one answer best?
2. Why should a containerized application log to stdout/stderr instead of a file?

**Intermediate:**
3. Why does Prometheus's "pull" model matter specifically in a Kubernetes environment where Pods come and go constantly?
4. What problem does a DaemonSet solve for log collection that a Deployment wouldn't?

**Advanced:**
5. Explain what a "span" is, and how multiple spans combine into a trace.
6. Why does an alert threshold typically require a sustained duration ("for 2 minutes") rather than firing on a single data point, and how does this connect to a concept from Part IX?

**Scenario:**
7. Grafana shows the backend's overall latency is normal, but customer complaints about slow checkout are increasing. Using everything in this chapter, explain why aggregate metrics alone might miss this, and what you'd add to catch it.

---

### Hands-On Lab 15.1 — Stand Up Full Observability for ShopSphere

**Objective:** Get Prometheus, Grafana, and basic tracing running against the ShopSphere backend, end to end.

**Prerequisites:** A kind cluster with ShopSphere's backend deployed (Part XI); Helm.

**Steps:**

1. Install the kube-prometheus-stack (a widely-used community Helm chart bundling Prometheus, Grafana, and Alertmanager together):
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm install monitoring prometheus-community/kube-prometheus-stack \
     --namespace monitoring --create-namespace
   ```

2. Add a `/metrics` endpoint to ShopSphere's backend (using your language's Prometheus client library) exposing at minimum a request counter and a latency histogram, and add the scrape annotations from Chapter 15.2 to its Deployment.

3. Port-forward into Grafana and log in:
   ```bash
   kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
   ```

4. Build a dashboard panel graphing request rate and error rate for the backend, using a PromQL query against the metric you exposed in step 2.

5. Generate some load and error conditions (a quick loop of `curl` requests, including some against a deliberately-broken endpoint) and watch the dashboard update in near-real-time.

6. Install Jaeger for a basic tracing example:
   ```bash
   kubectl create namespace tracing
   kubectl apply -n tracing -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml
   ```
   Instrument one endpoint of the backend with OpenTelemetry, and confirm a trace appears in the Jaeger UI after calling it.

**Expected result:** the Grafana dashboard shows live request/error rates responding to your generated load; a trace for the instrumented endpoint appears in Jaeger, showing at least one span.

**Verification:** deliberately spike the error rate (hit the broken endpoint repeatedly) and confirm the dashboard panel visibly reflects it within Prometheus's scrape interval — proof the full pipeline (app → scrape → storage → dashboard) is genuinely working, not just configured.

**Troubleshooting:** if Grafana shows no data at all, check `kubectl get servicemonitor -n monitoring` (the kube-prometheus-stack chart typically uses a `ServiceMonitor` custom resource to configure scraping, rather than the raw annotations shown in Chapter 15.2 — confirm one exists and correctly targets your backend's Service).

**Cleanup:**
```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring tracing
```

**Challenge:** create a Prometheus alerting rule that fires when the backend's error rate exceeds 5% for 2 minutes, and verify it actually transitions to a firing state (visible in Alertmanager's UI) when you sustain the broken-endpoint load from step 5 long enough.

---

*End of Part XVI. Part XVII covers Production Operations — high availability, backups and disaster recovery, cluster upgrades, capacity planning, incident response, and SLO/SLA/SLI concepts — the operational discipline that ties every technical capability built so far into an actual, sustainable way of running ShopSphere in production.*
