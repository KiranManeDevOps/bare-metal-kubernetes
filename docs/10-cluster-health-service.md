# 10 — The cluster health service, in depth

[06-observability.md](06-observability.md) introduces the health service in a
few paragraphs: ~70 checks, capacity forecasting, per-rule alert routing, a
constraint that it reads only Prometheus. This doc stays with that service —
what the checks actually test, how the forecasting maths works, how a rule
becomes a Slack message or an email, and why "read-only Prometheus" was a hard
line rather than a nice-to-have. It closes with the Prometheus/Grafana/SigNoz
plumbing that feeds it.

Both clusters run the same service. Only its storage checks differ — Ceph on
Cluster A, TopoLVM on Cluster B — because the checks are, deliberately, no
smarter than the metrics already sitting in Prometheus.

---

## 10.1 The shape of the thing

```mermaid
flowchart LR
  subgraph Nodes["Every node — cp-01, work-01…05"]
    NE["node_exporter"]
    PE["process-exporter · DaemonSet"]
  end
  BB["blackbox-exporter<br/>ICMP / TCP / HTTP probes"]
  KSM["kube-state-metrics"]
  CM["cert-manager metrics"]
  STOR["ceph_mgr <i>or</i> topolvm exporter"]
  API["kube-apiserver metrics"]
  TR["Traefik metrics entrypoint"]

  NE --> PROM[("Prometheus")]
  PE --> PROM
  BB --> PROM
  KSM --> PROM
  CM --> PROM
  STOR --> PROM
  API --> PROM
  TR --> PROM

  PROM -->|"PromQL over HTTP — read only"| HS["Cluster health service<br/>~70 checks every 30s"]
  HS --> DB[("SQLite<br/>7-day retention")]
  HS --> WSM["WebSocket manager"]
  WSM --> UI["Live dashboard<br/>17 pages"]
  HS --> AL["Alert engine"]
  AL -->|"per-rule or per-category routing"| SLACK["Slack"]
  AL --> TEAMS["Microsoft Teams"]
  AL --> DISC["Discord"]
  AL --> PD["PagerDuty"]
  AL --> SMTP["Email — SMTP"]
  AL --> HOOK["Generic webhook"]

  classDef svc fill:#1d4ed8,stroke:#1e3a8a,color:#fff
  classDef store fill:#047857,stroke:#065f46,color:#fff
  class HS svc
  class DB,PROM store
```

The service itself is a single FastAPI process: an asyncio loop scrapes
Prometheus on a timer, hands the heavy work to a two-worker thread pool so the
event loop stays responsive, writes results to SQLite, and pushes the same
results out over a WebSocket to anyone watching the dashboard. There is no
controller, no operator, no CRD — one Deployment, one PVC, one Service.

## 10.2 What ~70 checks actually check

Every scrape produces a **fixed set of checks** — the same ~70 entries every
30 seconds, whether or not there's anything to say. A check with nothing wrong
reports `HEALTHY` with a flat summary; a check whose backing exporter isn't
deployed reports `UNKNOWN` rather than disappearing. That's a deliberate
design choice: a dashboard whose row count changes between refreshes reads as
broken even when it isn't. The trade-off is visible in a few "deep" checks —
etcd internals, IPMI sensors, SMART data — that stay structurally present but
sit at `UNKNOWN` until a richer exporter (`smartmon-exporter`, `ipmi-exporter`)
is deployed alongside them. That's the honest cost of reading only
Prometheus: the checks are exactly as deep as what's already exposed there,
and no deeper.

| Category | Checks | What it actually tests |
|---|---|---|
| Nodes | 3 | `Ready` condition per node, node-level CPU/memory %, kubelet version drift across the fleet |
| Hardware | 8 | CPU load (1/5/15 m), memory, root filesystem %, `node_hwmon` temperature, open file descriptors, zombie process count, kernel/OS info, top-10 processes by CPU (via process-exporter) |
| RKE2 Control Plane | 5 | apiserver p99 latency, `kube-system` pod count, cert-manager certificate expiry, control-plane component presence |
| Etcd Deep | 4 | DB size, leader stability, alarms/compaction, disk/network latency — deliberately conservative, since etcd's own metrics aren't always exposed through a cluster's Prometheus |
| Storage — Ceph | 11 | Cluster health enum, per-OSD up/in state and apply/commit latency, monitor quorum, PG active/degraded counts, per-pool usage and IOPS, CephFS session count, slow-OSD detection, recovery I/O |
| Network | 5 | Calico DaemonSet readiness, CoreDNS replica availability, service endpoint count, ingress controller replica count, network-policy presence |
| Network Hardware | 7 | NIC error/drop rate, carrier-change (link flap) count, RX/TX bandwidth in Mbps, per-node NIC utilisation %, MTU consistency across nodes, bonding status, inter-node ICMP latency via blackbox-exporter |
| Security | 6 | Containers without a CPU limit, PodDisruptionBudget health, RBAC audit surface, resource quotas, privileged-container audit, image-tag audit |
| Workloads | 8 | Pod phase distribution, Deployment/DaemonSet/StatefulSet replica health, OOMKilled count, high-restart pods (>10 restarts), evicted pods |
| K8s Deep | 4 | Admission webhook presence, scheduling failures (pending pod count), resource fragmentation, deep certificate expiry (apiserver client certs) |
| Hardware Deep | 3 | SMART, IPMI, PCIe — flagged `UNKNOWN` unless the matching exporter is deployed |
| Capacity | 5 | Per-node CPU/memory headroom, pod-count-vs-limit, resource waste (requests set far above limits), idle/unused resources (completed pods, zero-replica deployments), growth-trend pointer to the Forecast page |

A few concrete thresholds, read straight out of the scraper: CPU is `WARNING`
above 70% and `CRITICAL` above 90%; memory the same at 75/90%; root filesystem
at 80/90%; hardware temperature at 75/90°C; a Ceph OSD is `WARNING` once
utilisation passes 85%; a certificate is `WARNING` inside 30 days and
`CRITICAL` inside 7; the apiserver p99 read latency check trips `WARNING`
above 500 ms; an OSD is flagged as a "slow OSD" once commit latency exceeds
20 ms or apply latency exceeds 50 ms.

### Storage checks change per cluster, nothing else does

Cluster A (Rook-Ceph) gets the Ceph block above — OSD map, PG status, pool
usage, CephFS sessions, quorum. Cluster B (TopoLVM) swaps that block for
per-node volume-group usage (`topolvm_volumegroup_size_bytes` /
`topolvm_volumegroup_available_bytes`) and CSI pod health, because there is no
Ceph cluster to ask. Every other category — nodes, hardware, network,
workloads, security, capacity — is identical between the two clusters. The
checks track what each cluster's storage plane actually is, not a lowest
common denominator.

## 10.3 Capacity forecasting — the actual maths

The Forecast page isn't a heuristic; it's ordinary least-squares linear
regression run per metric, per node, over the last 7 days of stored samples
(minimum 10 samples, minimum 5 per node, else the node is skipped):

1. Take each stored sample as a point `(x, y)`, where `x` is hours since the
   first sample in the 7-day window and `y` is the metric value.
2. Fit a line through them: slope = `Σ(x-x̄)(y-ȳ) / Σ(x-x̄)²`. This is the
   average rate of change, in units per hour.
3. `daily_change = slope × 24`.
4. Project forward: `value_at(t) = current + slope × t` for `t` = 7 days and
   30 days, clamped to `[0, 999]`.
5. **Days until the limit** (100% for CPU, memory, disk, and Ceph/TopoLVM
   usage): `(limit − current) / slope_per_hour / 24` — only computed when the
   slope is positive and the metric hasn't already hit the ceiling.
6. Status follows directly from that number: `CRITICAL` if the projection
   hits the limit within 7 days, `WARNING` within 30, otherwise `HEALTHY`.
   Forecasts are sorted so the most urgent (soonest to breach) sorts first.

It's linear, not seasonal or ARIMA — deliberately. A slow, steady climb in
volume-group usage is exactly what a straight-line fit is good at catching
early, and it's auditable: anyone can recompute "days until full" from the
same 7-day series by hand. The service runs the identical maths for CPU,
memory, disk and Ceph/TopoLVM usage; nothing metric-specific is hard-coded
beyond which four series get a ceiling.

A second analytics pass, separate from forecasting, flags **anomalies**: any
metric more than 2 standard deviations from its own 7-day mean *and* at least
10% off that mean (to filter noise in near-zero series). It's a companion to
the threshold-based alert rules below — thresholds catch "too high"; z-scores
catch "unusual for this node," which is a different and sometimes earlier
signal.

## 10.4 Alert rules and per-rule routing

Alert rules are data, not code — created, edited and deleted from the Alerts
page, evaluated against every check item on every 30-second scrape:

- A rule names a **metric** (must match a key already present in a check
  item's `metrics` dict — e.g. `cpu_percent`, `days_until_expiry`,
  `commit_latency_ms`), a **comparison operator**, and separate **warning**
  and **critical** thresholds.
- An optional **item pattern** (regex) scopes the rule to specific nodes or
  items — e.g. only `work-0[3-5]`.
- A **notify level** — `both`, `critical only`, or `dashboard only` (never
  leaves the UI) — controls whether external channels fire at all.
- A **message template** with `{value}`, `{item}`, `{metric}`, `{threshold}`
  placeholders.

**Routing priority**, evaluated per alert: if the rule lists explicit
notification channels (by ID or name), only those fire. If it doesn't, the
alert falls back to **category-based routing** — each channel has an optional
comma-separated category filter, and an alert only reaches channels whose
filter is blank (all categories) or includes that alert's category. This is
what lets storage alerts go to the storage channel and security alerts go
elsewhere, without every rule having to spell out its own channel list.

**Cooldown and resolution.** A fired rule+item pair won't refire for
`ALERT_COOLDOWN` seconds (default 300) — *unless* the database shows no
active row for it, which means it was previously resolved or cleared; in that
case the cooldown is bypassed so the alert can't get stuck silently above
threshold. Each scrape also diffs the set of currently-tripping rule+item keys
against what's marked active in the database: anything active that didn't
trip this cycle gets marked resolved and a `RESOLVED` notification goes out
through the *same channel mapping the original rule used* — looked up by rule
name, so a resolved notification still routes correctly even if the rule was
edited or deleted in the meantime.

**Channel transport is auto-detected from the URL** — no channel "type" field
to get wrong:

| Signal in the URL | Transport | Notes |
|---|---|---|
| `office.com` / `microsoft` | Microsoft Teams | `MessageCard` payload with severity-coloured theme |
| `hooks.slack.com` | Slack | Coloured attachment, emoji by severity |
| `discord` | Discord | Embed with severity colour |
| `pagerduty.com` | PagerDuty | Events v2 `trigger` payload |
| `smtp://` / `smtps://` | Email | Own code path — see below |
| anything else `https://` | Generic webhook | Plain `{text, alert}` JSON POST |

Email is the one transport that isn't a webhook: the SMTP host, port,
credentials, sender and recipients are packed into the channel URL itself and
parsed out (`smtp://user:pass@host:587?from=...&to=a,b`), then sent via
STARTTLS or implicit TLS depending on scheme and query flags. Every transport
gets a **Test** button that fires a synthetic `info`-severity alert through
the real transport — the fastest way to confirm a webhook or SMTP relay
actually works before waiting for a real incident.

## 10.5 The live dashboard, over WebSockets

A single `/ws` endpoint accepts connections and does nothing but broadcast:
after each scrape, if at least one client is connected, the service pushes a
lightweight `snapshot` message (timestamp, overall status, check count) and
then the full `checks` payload. Clients send `"ping"`, the server answers
`{"type":"pong"}` — a keepalive, not a request/response protocol; all real
data flows server-to-client. Dead sockets are pruned from the broadcast set
on the next failed send, so a lost connection doesn't leak a broadcast target
forever. This is what makes the "17-page" dashboard feel live without polling
17 different endpoints every 30 seconds — one push updates every open tab.

## 10.6 Authentication

Login is **mandatory and cannot be disabled**. There are exactly two roles:

| Role | Access |
|---|---|
| `admin` | Everything — users, alert rules, data sources, notification channels, backup/restore |
| `read` | Every page and every `GET` endpoint, no `POST`/`PUT`/`DELETE` |

Enforcement is a single middleware: any request outside a short public-path
allowlist (`/login`, `/api/login`, `/favicon.ico`, `/api/status`,
`/api/health`, and the WebSocket) must carry a valid session cookie, and a
`read`-role session gets a `403` on any mutating verb except login/logout.
Sessions are opaque 32-byte random tokens held in an in-memory dict with a
7-day expiry, set as an `HttpOnly`, `SameSite=Lax` cookie — deliberately not a
signed JWT; there's nothing to verify offline because the token itself carries
no claims. Passwords are salted and hashed (a fresh random salt per user,
`SHA-256(salt‖password)`, compared with a constant-time check) and must meet
a minimum policy: 8+ characters, upper- and lower-case, a digit, a special
character. On first boot, if no users exist yet, an `admin` user is created
either from an `ADMIN_PASSWORD` environment variable (validated against the
same policy) or — if that's unset — from a random password that satisfies the
policy, logged exactly once to the pod's stdout. There is no built-in default
credential to forget to change.

## 10.7 The constraint that shapes everything else: read-only Prometheus

The service's `ServiceAccount` sets `automountServiceAccountToken: false` and
carries no `ClusterRole` or `RoleBinding` at all — there is nothing to bind.
Every fact the dashboard shows, every check it runs, every forecast it
projects comes from one HTTP call: `GET {prometheus_url}/api/v1/query`. No
kubeconfig is mounted, no `kubectl exec`, no elevated RBAC of any kind.

That's a constraint the design keeps to on purpose, for two concrete reasons:

- **A monitoring tool that holds cluster-admin is a second attack surface**
  wearing a high-visibility jacket. It's the one workload with legitimate
  reason to watch everything, which makes it exactly the one workload an
  attacker most wants to compromise. Denying it any Kubernetes API access at
  all removes that surface entirely rather than trying to scope it tightly.
- **It can be deployed, restarted, or upgraded without anyone reviewing an
  RBAC diff.** A `ClusterRole` bump hiding in a routine image update is a real
  failure mode for tools that *do* need API access; a tool that structurally
  cannot request more access cannot have that failure mode.

The cost is real and worth naming: the service can't see anything Prometheus
doesn't already have. If a metric isn't scraped, there's no check for it — no
falling back to `kubectl describe` for the missing detail. In practice this
pushes the effort to the right place: getting node_exporter running on every
node (including tainted control-plane nodes — the DaemonSet needs a
toleration for `NoExecute`) buys more coverage than adding permissions to the
monitor ever would.

## 10.8 Two things that bit in production, and the fixes already in the manifests

- **SQLite is a single writer.** The Deployment strategy is `Recreate`, not
  `RollingUpdate` — two pods writing the same SQLite file is worse than a
  short gap during rollout. The PVC is sized and commented for a **fast local
  NVMe storage class**, not Ceph RBD: the workload writes roughly 370 rows
  every 30 seconds into a database that grows to multiple gigabytes, and
  network-attached storage latency was enough to pin the asyncio event loop
  in disk-wait, which then delayed *everything* the process does, including
  answering health probes.
- **Liveness and readiness probes are TCP, not HTTP.** An HTTP probe would
  route through the same asyncio loop that a heavy SQLite write can briefly
  stall; a TCP-socket probe is answered by the OS below that loop, so a
  momentary stall doesn't read as a crash and trigger a pointless restart.
  `/api/health` (used only as a public status check, not the probe target)
  is written to skip the database entirely for the same reason.

## 10.9 Prometheus, Grafana and the exporters underneath it

The health service is a consumer, not a data source — it only exists because
Prometheus already holds the metrics. What feeds Prometheus:

| Exporter | Deployed as | Why it's there |
|---|---|---|
| `node_exporter` | DaemonSet, toleration for all taints | CPU, memory, disk, temperature, network, file descriptors — on every node, including the control plane, which is usually tainted `NoExecute` and easy to forget |
| `kube-state-metrics` | Deployment | Kubernetes object state — pod phases, deployment/daemonset/statefulset replica counts, PVC phase, PDB status — the layer above raw node metrics |
| `blackbox-exporter` | Deployment, `NET_RAW` capability | ICMP reachability and latency between nodes — the one thing that catches "the network between two hosts is degraded" when every individual host still reports healthy |
| `process-exporter` | DaemonSet, `hostPID` | Per-process CPU/memory by name, feeding the "top processes" check and per-node process-level dashboards — the detail below "the node is busy" |
| `ceph_mgr` exporter (Cluster A) / `topolvm` metrics endpoint (Cluster B) | built into Rook-Ceph / scraped from the CSI pods | Storage-plane health matched to whichever storage model that cluster actually runs |
| cert-manager metrics | scraped from the controller | Certificate readiness and expiry timestamps — the countdown the health service turns into a check |
| Traefik metrics entrypoint | scraped from the ingress DaemonSet | Request rate, status mix and latency per entrypoint/router/service, feeding both the Grafana ingress dashboard and (via labels) the health service's ingress check |

Custom Grafana dashboards live as JSON in Git and are loaded by Grafana's
**dashboard sidecar**, which watches for ConfigMaps labelled
`grafana_dashboard=1` and reloads within about 60 seconds of a change — a
dashboard edit is a commit and a `kubectl apply`, not a click that only exists
in one Grafana's own database.

| Dashboard | Question it answers |
|---|---|
| Nodes Overview | Are all nodes up, and how loaded — CPU, memory, load, temperature, network throughput — both in-cluster and any external hosts the same stack watches? |
| Network | Is inter-node connectivity clean — ICMP latency and reachability, receive/transmit throughput, drops and errors, conntrack pressure? |
| Security & Cluster Health | Is the control plane itself sound — apiserver error rate and p99 latency, NotReady nodes, node pressure conditions, evicted/failed pods, PDB health, NetworkPolicy coverage, containers missing a CPU limit? |
| Workloads & Pods | Which pods are unstable — restart counts, phase distribution over time, top CPU/memory consumers, deployment availability? |
| TopoLVM Storage (Cluster B) | Is any node's local volume group about to fill, and is the CSI driver healthy? |
| Traefik Ingress | Request rate, latency percentiles, status-code mix, TLS version/cipher share and certificate expiry, per entrypoint/router/service — the metrics-side view of the edge |

## 10.10 Tracing — OpenTelemetry through SigNoz

```mermaid
flowchart LR
  C["Client request"] --> EP["Traefik EntryPoint span<br/>ingress"]
  EP --> RP["Traefik ReverseProxy span<br/>call to backend pod"]
  EP -->|"OTLP/HTTP"| OTEL["SigNoz OTel collector"]
  RP -.->|"span timing"| OTEL
  OTEL --> STORE[("SigNoz trace store")]
  STORE --> DASH["Traefik Ingress (Traces) dashboard"]

  subgraph DASH_DETAIL["Panels"]
    RED["RED: rate, errors, duration"]
    LAT["Latency p50/p90/p99<br/>incl. WebSocket-excluded view"]
    BACK["Backend request rate & p99<br/>by upstream address"]
    SLOW["Slowest requests<br/>click-through to full trace"]
  end
  DASH --> DASH_DETAIL

  classDef svc fill:#1d4ed8,stroke:#1e3a8a,color:#fff
  class OTEL svc
```

Traefik exports two kinds of span for every request: an **EntryPoint**
(server) span — one per request arriving at the edge — and a **ReverseProxy**
(client) span for the call it makes onward to the backend pod. Traefik's own
internal routers are excluded, so the trace describes application traffic,
not the proxy's bookkeeping. This is why tracing answers a different question
than the metrics side: Prometheus tells you *whether* something is slow;
individual traces tell you *where in the hop chain* the time went, without
having instrumented the application itself.

One detail worth keeping: the latency panels compute two versions of the same
percentiles — one over all requests, one excluding `101 Switching Protocols`
upgrades. A long-lived WebSocket connection reports its entire lifetime as
"duration" to a naive p99, which would make every regular HTTP request look
fast by comparison; excluding upgrades keeps the percentile honest for actual
request/response traffic.

## 10.11 Why this design, taken together

None of the individual pieces here is exotic — Prometheus queries, a linear
fit, webhook POSTs, a WebSocket broadcast. What makes the service worth
having isn't novel technology; it's that someone already decided what
"healthy" means for *this* platform, tuned the thresholds to it, wired the
routing so the right alert reaches the person who can act on it, and drew a
hard line at *read-only Prometheus* so the tool that watches everything can
never itself become the thing that needs watching most closely. Everything it
reports was already sitting in Prometheus; the value is entirely in that
packaging.
