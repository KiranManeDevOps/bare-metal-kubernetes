# 7 — Security

On-premises does not mean trusted. The cluster sits on an office LAN, serves
hosts on the public internet, and gives developers access — three different trust
boundaries that need three different controls.

---

## 7.1 Two planes of access, kept separate

| Plane | Who | How |
|---|---|---|
| **Cluster API** (`kubectl`) | Operators and developers | Scoped kubeconfig per environment, through the API VIP |
| **Applications** (HTTPS) | End users | Through the ingress only, with TLS terminated at Traefik |

Nothing bridges the two. A user of an application never receives cluster
credentials, and cluster access never depends on an application being reachable.
They fail independently, which is the point.

## 7.2 Network: default deny

Calico enforces NetworkPolicy, so each application namespace moves from "anything
can reach anything" to "deny by default, allow what is needed" with five policies:

1. **`default-deny-all`** — all ingress and egress denied. The baseline.
2. **`allow-dns-egress`** — egress to CoreDNS on 53/UDP and 53/TCP. Without this,
   nothing resolves and every symptom looks like something else.
3. **`allow-same-namespace`** — pods in a namespace may talk to each other.
4. **`allow-ingress-from-edge`** — ingress from the ingress controller's
   namespace, so published applications are reachable.
5. **`allow-egress-apiserver`** — egress to the API server for in-cluster SDKs.

**Roll it out one namespace at a time**, starting somewhere non-critical, and
watch for broken traffic before proceeding. System namespaces — CNI, storage,
load balancer, ingress, monitoring — need their own allow rules and should not
receive a blanket default-deny until each flow has been tested.

Manifest: [`examples/security/networkpolicies.yaml`](../examples/security/networkpolicies.yaml).

## 7.3 RBAC

Access is granted by role, not by person, and the default is read-only:

- **Developers** get a `ClusterRole` that can read workloads and read logs in
  their own namespaces, and nothing cluster-wide that would let them read
  Secrets across the cluster.
- **Operators** get administrative access scoped per environment, not one
  credential that opens everything.
- **Service accounts** get exactly the verbs their controller needs. kube-vip is
  a good example: services, nodes, endpoints and leases — nothing else.
- **Management UI roles** mirror the same split, so a person's access is the
  same whether they arrive through the UI or through `kubectl`.

The rule that prevents most mistakes: **no permanent cluster-admin for humans.**
Cluster-admin is a deliberate, temporary escalation, not the everyday identity.

## 7.4 Secrets

- **Nothing sensitive is committed.** Manifests reference Secrets by name;
  Secrets are created out of band. That is why this repository can be public.
- **Backup credentials and encryption passwords live outside the cluster too** —
  a backup password stored only in the cluster it protects is not a backup
  password ([05 — Backup and DR](05-backup-and-dr.md)).
- **Certificates are managed, not handled.** cert-manager issues and renews them
  into Secrets; no private key is ever copied to a laptop.

## 7.5 Management interfaces

The cluster management UI holds full cluster-admin, so it is treated as the most
sensitive endpoint on the platform:

- Never published to the internet on its own. It stays on the LAN or VPN, or
  behind an IP allowlist at the ingress.
- Reached only over HTTPS, with the ingress forwarding to the backend's **HTTPS**
  port.
- Its bootstrap password is changed immediately, and its server URL is set
  explicitly rather than inferred from whatever host first reached it.

The same applies to any dashboard that exposes cluster internals — a proxy
dashboard, a metrics UI, a storage console. If it can be reached without
authentication, it is an outage report published in advance.

## 7.6 The platform's own attack surface

Two choices elsewhere in this repository are security decisions as much as
operational ones:

- **The health service reads only Prometheus** — no kubeconfig, no `exec`, no
  elevated permissions ([06 — Observability](06-observability.md)). Monitoring
  that holds cluster-admin is a second way into the cluster.
- **TLS 1.2 is the floor at the edge**, with a modern cipher list, and backend
  verification is relaxed only for in-cluster self-signed services — never for
  anything crossing a trust boundary.
