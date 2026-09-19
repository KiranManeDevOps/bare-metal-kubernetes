# 6 — Observability

Three layers, each answering a different question:

| Layer | Question it answers | Tool |
|---|---|---|
| Metrics | Is anything outside its normal range? | Prometheus + Grafana |
| Traces | Where did this request actually spend its time? | SigNoz via OpenTelemetry |
| Health | Is the *cluster itself* — nodes, storage, certificates — healthy? | A purpose-built service |

---

## 6.1 Metrics

`kube-prometheus-stack` provides Prometheus, Alertmanager, node-exporter and
kube-state-metrics. On top of the defaults, the things worth scraping on bare
metal are the ones nobody ships dashboards for:

- **Volume group free space per node.** With thick LVM this is *the* storage
  alert. There is no pool metadata to watch — one number per node, and when it
  reaches zero, new volumes stop being provisioned while running workloads carry
  on unaffected.
- **CSI driver health** — controller and per-node components.
- **Replication age** — how long since each backup last succeeded.
- **MetalLB and kube-vip** — address allocation and VIP ownership.
- **Ingress** — request rate, error rate and latency per entrypoint, router and
  service.
- **Blackbox and process probes** — for the handful of things that live outside
  Kubernetes but the cluster depends on.

Dashboards live in Git and are loaded by Grafana's sidecar from ConfigMaps, so a
dashboard change is a commit rather than a click that exists only in one
Grafana's database.

## 6.2 Tracing

Traefik exports OpenTelemetry traces over OTLP/HTTP to a SigNoz collector, with
its own internal routers excluded so the traces describe application traffic
rather than the proxy's bookkeeping. Because the ingress is the first hop for
every external request, this alone answers the most common production question —
*is it slow in the network, in the proxy, or in the application?* — without
instrumenting anything else first.

Prometheus labels are enabled per entrypoint, router and service. That is a
deliberate cardinality choice: it is affordable at this cluster's route count and
would not be at ten times the routes.

## 6.3 A cluster health service

Dashboards answer questions you thought to ask. The gap on bare metal is
everything between the hardware and Kubernetes — a volume group filling up, a
certificate quietly approaching expiry, a node's clock drifting, a backup that
stopped running three days ago.

So the cluster runs a small service of its own:

- **~70 health checks every 30 seconds**, all derived from Prometheus queries.
- **Capacity forecasting** — projecting per-node storage growth forward so the
  warning arrives while there is still time to add a disk.
- **Alert routing per rule** to Slack, Teams, Discord, PagerDuty, email or a
  generic webhook, with automatic resolved notifications.
- **A live dashboard** over WebSockets for the current state at a glance.

The design constraint that mattered most: **it reads only Prometheus.** No
kubeconfig, no `exec` into pods, no elevated cluster permissions. A monitoring
system that holds cluster-admin is a second attack surface wearing a high-visibility
jacket — and one that reads a metrics endpoint can be deployed, restarted and
upgraded without anybody reviewing its RBAC.

Everything it reports is already in Prometheus. The value is in the packaging:
the checks encode what "healthy" means for *this* platform, the thresholds are
tuned to it, and the routing puts each alert in front of whoever can act on it.

## 6.4 Alerting principles

- **Alert on symptoms, not causes.** "Certificates renew in 3 days" is
  actionable; "certificate controller reconcile count is unusual" is noise.
- **Alert on absence.** A backup that never ran produces no failure event.
  Alerting on *age* catches the silent stop; alerting on failure does not.
- **Every alert names its action.** If nobody can say what to do about it, it is
  a dashboard panel, not an alert.
- **Route by ownership.** Storage alerts and application alerts rarely have the
  same audience, and an alert sent to everyone is read by no one.
