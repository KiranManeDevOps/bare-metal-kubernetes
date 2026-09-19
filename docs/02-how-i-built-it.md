# 2 — How I built it

The build order, start to finish. Each phase leaves the cluster in a state you
can verify before moving on, which matters on bare metal: a mistake in phase 2
surfaces as an unexplainable failure in phase 6.

```
1. Prepare every node            →  identical OS baseline
2. Prepare disks on storage nodes →  one volume group per node
3. API VIP (HAProxy + keepalived) →  a stable endpoint that outlives any node
4. RKE2 server on the control plane
5. Join the workers               →  all nodes Ready
6. cert-manager                   →  prerequisite for the storage webhook
7. TopoLVM + StorageClasses       →  storage, smoke-tested
8. VolSync                        →  off-node copies
9. MetalLB, then kube-vip + Traefik + issuers  →  the edge
10. Rancher and observability     →  day-2 tooling
```

---

## Phase 1 — Node preparation (every node)

The same baseline everywhere: identical packages, kernel modules, sysctls and
limits. Divergence between nodes is the source of most "works on one node" bugs.

Points that matter more than they look:

- **`lvm2` is mandatory** on storage nodes — TopoLVM drives LVM directly.
- **Time synchronisation is not optional.** etcd is intolerant of clock skew
  between members; `timedatectl` must report the clock as synchronised before
  RKE2 is installed.
- **Hostnames are lowercase RFC1123.** Kubernetes node names must be, and a
  capital letter here surfaces much later as a confusing registration failure.
- **Swap off**, in `fstab` as well as at runtime.
- **Kernel modules and sysctls**: `overlay`, `br_netfilter`, the IPVS modules,
  `nf_conntrack`; IP forwarding, bridge-netfilter, and generous `inotify`,
  `nofile` and `nf_conntrack_max` limits. Raise containerd's systemd limits to
  match, or the daemon caps out long before the kernel does.

## Phase 2 — Disks on storage nodes

Each storage node gets one volume group carved from its NVMe device:

```bash
DEV=/dev/nvme0n1
lsblk -dno NAME,SIZE,MODEL "$DEV"     # confirm the data disk, never the root disk
findmnt -no SOURCE /                  # prove the running root is somewhere else

wipefs -a "$DEV"
sgdisk --zap-all "$DEV"
pvcreate "$DEV"
vgcreate vg-topolvm-work-01 "$DEV"    # named after the node
vgs vg-topolvm-work-01
```

Disks that arrive with a previous OS layout need that layout torn down first —
LVM holds the device busy, and `wipefs` fails with `Device or resource busy`
until the old volume group is deactivated and removed
([08 — Lessons learned](08-lessons-learned.md#3-a-disk-that-refused-to-be-wiped)).

Nodes without a data disk are skipped entirely and never labelled for storage.

## Phase 3 — API VIP

HAProxy fronts the Kubernetes API on `:6444`, and keepalived owns the VIP
`10.0.10.10` and moves it if the HAProxy health check fails. With one
control-plane node this is a thin shim — its value is that the API endpoint, the
TLS SAN and every kubeconfig are already pointing at an address that does not
belong to a single machine, so control-plane nodes can be added later without
re-issuing anything.

Keep keepalived's VRRP router ID distinct from anything else on the segment, and
keep its VIP out of the MetalLB pool.

Config: [`examples/rke2/haproxy.cfg`](../examples/rke2/haproxy.cfg),
[`examples/rke2/keepalived.conf`](../examples/rke2/keepalived.conf).

## Phase 4 — Control plane

RKE2's server config declares the cluster's identity, disables the bundled
components we replace, and reserves resources for the control plane.

```yaml
tls-san: [k8s-api.example.com, 10.0.10.10, 10.0.10.11]
cluster-name: onprem-cluster-01
cluster-cidr: 10.42.0.0/16
service-cidr: 10.43.0.0/16
cluster-dns: 10.43.0.10
cni: [calico]
disable: [rke2-canal, rke2-ingress-nginx]
```

Two decisions are load-bearing:

- **Calico instead of the bundled Canal**, for NetworkPolicy enforcement
  ([07 — Security](07-security.md)).
- **No taint on the control-plane node.** RKE2 pins its add-on installer jobs to
  the control-plane node; a custom taint there leaves them `Pending` and the CNI
  never installs. Workloads are kept off with `nodeSelector` and held back with
  kubelet reservations instead.

Full file: [`examples/rke2/config-server.yaml`](../examples/rke2/config-server.yaml).

## Phase 5 — Workers

Each agent points at the server's supervisor port with the node token, sets its
own name, IP and labels, and applies its own kubelet reservations. Storage and
application nodes are labelled `role=app`; the batch node takes a
`NoSchedule` taint so only workloads that tolerate it land there.

```bash
kubectl get nodes -o wide                      # every node Ready
kubectl get nodes -L node.example.com/role
```

File: [`examples/rke2/config-agent.yaml`](../examples/rke2/config-agent.yaml).

## Phase 6 — cert-manager

Installed before TopoLVM, because TopoLVM's admission webhook needs a
certificate. It later issues the cluster's Let's Encrypt certificates too
([04 — Ingress migration](04-ingress-migration.md)).

## Phase 7 — Storage

Label the storage nodes, install TopoLVM with one `lvmd` per node, then apply the
per-node StorageClasses:

```bash
for n in work-01 work-02 work-03 work-04; do
  kubectl label node "$n" topolvm.io/storage=true --overwrite
done

helm upgrade --install topolvm topolvm/topolvm -n topolvm-system \
  -f examples/storage/topolvm-values.yaml
kubectl apply -f examples/storage/storageclasses.yaml

kubectl get sc                                  # four classes, none default
kubectl get csistoragecapacities -A -o wide     # per-node free space
```

Two settings decide whether scheduling works at all — capacity tracking **on**,
pod mutating webhook **off**. The reasoning is in
[08 — Lessons learned](08-lessons-learned.md#2-every-pod-with-a-pvc-was-unschedulable).

Smoke-test with a real PVC and confirm the logical volume appears on the
expected node (`lvs vg-topolvm-work-01`) before trusting anything to it.

## Phase 8 — Off-node copies

VolSync provides scheduled PVC-to-PVC or PVC-to-S3 replication. With thick
volumes there are no snapshots, so it runs in `Direct` mode, reading the live
volume. Destinations go on a *different* node's class than the source —
replicating onto the same node protects against nothing.

## Phase 9 — The edge

In order: MetalLB for general LoadBalancer addresses, then kube-vip for the
dedicated ingress VIPs, then Traefik, then the cert-manager `ClusterIssuer`s,
then the application Ingresses. Test with Let's Encrypt **staging** first — the
production endpoint's rate limits are unforgiving of a misconfigured DNS record.

Full sequence, including the cutover from a previous reverse proxy and the
rollback path: [04 — Ingress migration](04-ingress-migration.md).

## Phase 10 — Day-2 tooling

Rancher for cluster management, and the observability stack — Prometheus,
Grafana, SigNoz, plus a purpose-built health service
([06 — Observability](06-observability.md)).

Rancher is full cluster-admin, so it is never simply published: it sits behind an
IP allowlist or stays on the LAN/VPN, and its ingress forwards to the **HTTPS**
backend port, not the plaintext one.

---

## Verification checklist

```bash
# Nodes and scheduling
kubectl get nodes -o wide
kubectl get pods -A | grep -vE 'Running|Completed|^NAMESPACE'   # expect nothing

# Storage
kubectl get sc                                                  # no default class
kubectl get csistoragecapacities -A -o wide
ssh work-01 'sudo vgs vg-topolvm-work-01'                       # free space

# Edge
kubectl -n kube-system get svc rke2-traefik -o wide             # EXTERNAL-IP assigned
kubectl get certificate -A                                      # all True

# Backup
kubectl get replicationsource -A                                # LAST SYNC recent
kubectl -n platform get cronjob                                 # LAST SCHEDULE recent
```
