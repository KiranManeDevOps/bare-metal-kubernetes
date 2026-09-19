# 4 — Ingress migration: reverse proxy → Traefik

The cluster's first edge was a single reverse-proxy pod holding a LoadBalancer
address, terminating TLS and forwarding to ClusterIP services. It was quick to
stand up and easy to explain, and it had one fatal property: **it could not
survive the loss of its node.**

This is the migration to Traefik on a dedicated VIP, with certificates issued by
cert-manager — done on a live cluster, with a rollback available at every step.

---

## 4.1 Why move

| | Reverse proxy pod | Traefik |
|---|---|---|
| State | SQLite database + certificate files on a node-local volume | None — routes from the API, certificates in Secrets |
| Replicas | Exactly one (not built for active-active) | 2+, or a DaemonSet across workers |
| Survives a node failure | **No** — the RWO volume is stranded on the dead node | **Yes** — any remaining node serves |
| Configuration | Clicked in a UI | `Ingress` resources in Git |
| Certificates | Requested in the UI | cert-manager, issued and renewed automatically |
| Observability | Basic logs | Prometheus metrics and OpenTelemetry traces per route |

The deciding argument was the second row. A stateful single-replica pod holding a
node-local volume is a single point of failure in front of *every* application,
and it cannot be made properly highly available — its state was never designed to
be shared. Replacing it with a stateless ingress removes the problem rather than
working around it.

## 4.2 The target

```
client ──TCP 80/443──→ kube-vip VIP 10.0.10.60 ──→ Traefik (DaemonSet on workers) ──→ Ingress → service
client ──UDP 443─────→ kube-vip VIP 10.0.10.61 ──→ Traefik QUIC listener (HTTP/3)
```

RKE2 ships Traefik as a bundled addon, so it is **enabled by removing it from the
server config's `disable:` list** and then tuned with a `HelmChartConfig` of the
same name in `kube-system`, which the helm-controller merges over the bundled
chart's values. That keeps the addon under RKE2's management instead of forking
it into a separately installed chart.

> **The one line that must stay untouched.** There is a `ingress-controller:`
> key that appears in K3s documentation and is **invalid in RKE2**. Setting it
> crashes the server process and takes the control plane down with it. The only
> supported change is the `disable:` list. See
> [08 — Lessons learned](08-lessons-learned.md#4-a-config-key-that-is-valid-in-k3s-and-fatal-in-rke2).

## 4.3 Cutover

Sequenced so the old edge keeps serving until the moment the VIP moves.

```bash
# 1. Label the nodes that may announce the ingress VIP (never the control plane —
#    it already holds the API VIP through keepalived).
kubectl label node work-01 work-02 work-03 work-04 kube-vip/iface=eth0

# 2. Shrink the MetalLB pool so it can never hand out the ingress VIPs.
kubectl apply -f examples/network/metallb-pool.yaml

# 3. Release the addresses from the old proxy (its volume is kept, for rollback).
kubectl -n edge scale deploy/proxy --replicas=0
kubectl -n edge delete svc proxy-http proxy-admin

# 4. kube-vip claims the ingress VIPs — class-scoped, so MetalLB ignores them.
kubectl apply -f examples/ingress/kube-vip.yaml
kubectl -n kube-system rollout status ds/kube-vip-ds --timeout=120s

# 5. Tune the bundled Traefik: VIP, worker-only placement, TLS floor, timeouts.
kubectl apply -f examples/ingress/traefik-helmchartconfig.yaml
kubectl -n kube-system get svc rke2-traefik -o wide     # EXTERNAL-IP → 10.0.10.60

# 6. Certificate issuers — staging first.
kubectl apply -f examples/ingress/cluster-issuer.yaml

# 7. Applications, one at a time, staging issuer → verify → production issuer.
kubectl get certificate -A                              # READY must be True
```

Then repoint DNS per host: **TCP 443 → `10.0.10.60`**, **UDP 443 → `10.0.10.61`**.

## 4.4 Things that bite during a cutover

**Let's Encrypt validates from the public internet.** A hostname whose A record
points at a private address fails with "no valid A records" no matter how correct
the cluster is. Every public host needs a real public A record before its
certificate can issue.

**HTTP-01 self-checks need to reach the public address.** cert-manager verifies
its own challenge before telling the CA to validate. If the network cannot route
a LAN client back to its own public IP, that self-check fails even though the
challenge is being served correctly. Enabling hairpin NAT at the firewall fixes it
properly; pinning the hostname inside the cert-manager pod works as a stopgap.

**Self-signed backends.** Services that serve their own TLS (management UIs,
database monitoring tools) fail with `x509: certificate signed by unknown
authority` until the proxy is told to skip verification toward the backend. That
setting is global in Traefik — a per-service transport alone was not honoured.

**Backends that redirect to HTTPS.** Forwarding to a plaintext port on a service
that 302-redirects to HTTPS produces a redirect loop; forwarding HTTPS traffic to
that plaintext port produces `SSL: wrong version number` and a 502. Forward to the
HTTPS port, and enable WebSocket support for UIs with live logs or shells.

**HTTP/3 needs its own service.** The bundled chart adds only TCP 80 and 443 to
the main service, so QUIC gets a separate UDP LoadBalancer on its own VIP with
`http3.advertisedPort=443`. Browsers then see `Alt-Svc: h3=":443"` and send QUIC
to the same hostname on the same port number.

## 4.5 Rollback

| If this breaks | Undo |
|---|---|
| Traefik tuning | `kubectl delete -f traefik-helmchartconfig.yaml` — reverts to bundled defaults |
| Traefik entirely | Re-add it to `disable:` in the server config and restart RKE2 |
| kube-vip | `kubectl delete -f kube-vip.yaml` — the VIPs are released |
| Need the old edge back | Re-expand the MetalLB pool, scale the proxy back up, recreate its services (its volume was never deleted) |
| Anything at all | The API VIP is never touched, so `kubectl` access is never at risk |

That last row is the one that makes the migration safe to attempt on a live
cluster: the control-plane path and the application path are deliberately
separate mechanisms on separate addresses, so an edge mistake can never cost you
access to the cluster you need in order to fix it.

## 4.6 What it bought

- **Node-failure tolerance at the edge** — Traefik runs on every worker; any of
  them can serve, and losing one is not an outage.
- **Certificates as infrastructure** — issued and renewed by cert-manager,
  stored in Secrets, replicated in etcd, with no manual step.
- **Per-route observability** — Prometheus metrics broken down by entrypoint,
  router and service, plus OpenTelemetry traces exported to SigNoz
  ([06 — Observability](06-observability.md)).
- **Routing in Git** — an `Ingress` next to the application it exposes, instead
  of a UI whose state lived in one pod's volume.
