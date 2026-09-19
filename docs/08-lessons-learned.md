# 8 — Lessons learned

Five failures worth writing down. Each one cost hours, each has a one-line fix,
and none of them is in the documentation you would have read first.

---

## 1. A taint on the control plane deadlocked the CNI bootstrap

**Symptom.** A freshly installed cluster never came up. Every worker sat
`NotReady`, and the CNI was never installed.

**Cause.** The control-plane node had been tainted `NoSchedule` so it would only
run a chosen class of workloads — standard practice, and wrong here. RKE2
installs its bundled components (CNI, CoreDNS and friends) through helm-install
**Jobs that are pinned to the control-plane node by node affinity**. The custom
taint left those Jobs `Pending` with `FailedScheduling: untolerated taint`, so
the CNI never installed, so no node ever became Ready.

**Fix.** Leave the sole control-plane node untainted. Keep workloads off it with
`nodeSelector` instead, and protect the control plane with kubelet
`system-reserved` / `kube-reserved` and priority classes
([01 — Architecture](01-architecture.md#11-logical-topology)).

**The general lesson.** On a single-control-plane cluster, a taint there is not a
scheduling preference — it is a bootstrap dependency. Once a second control-plane
node exists, the add-on Jobs have somewhere else to run and the taint becomes
safe again.

---

## 2. Every pod with a PVC was unschedulable

**Symptom.** Storage installed cleanly, volumes provisioned, and then every pod
that mounted one stuck at `Pending`:
`0/N nodes available: Insufficient topolvm.io/capacity`.

**Cause.** Two mutually exclusive scheduling mechanisms, both half-enabled. The
CSI driver's **pod mutating webhook** stamps a `topolvm.io/capacity` extended
resource onto every pod that mounts one of its volumes — a resource that **only
its scheduler extender advertises**. We were deliberately using the modern path
instead: capacity tracking through the standard `CSIStorageCapacity` API with the
stock scheduler, and no extender deployed. So the webhook requested a resource
that nothing in the cluster would ever provide.

**Fix.** With `storageCapacityTracking` enabled, keep the pod mutating webhook
**off**. Pick one mechanism:

| Mechanism | Needs | Webhook |
|---|---|---|
| `CSIStorageCapacity` + stock scheduler | Nothing extra | **Off** |
| Scheduler extender | The extender deployed and wired into the scheduler | On |

**The general lesson.** When a component offers an old and a new way to do the
same job, enabling both is not belt-and-braces — it is a deadlock where each half
waits for something the other half was supposed to provide.

---

## 3. A disk that refused to be wiped

**Symptom.** `wipefs` and `pvcreate` failed on a brand-new storage node with
`Device or resource busy` and `device has a signature`, even though nothing was
mounted from it.

**Cause.** The disks arrived carrying a previous OS layout — an EFI partition, a
boot partition and an LVM volume group — while the running root filesystem lived
on a different drive entirely. LVM had automatically activated that old volume
group at boot, which held the device busy.

**Fix.** Tear the old layout down before building the new one, and confirm every
step:

```bash
findmnt -no SOURCE /      # prove which device the running root is on
pvs                       # identify the volume group on the target disk
vgchange -an <old-vg>     # deactivate — this is what releases the device
vgremove -f <old-vg>
pvremove -ff <device>p3
wipefs -a <device>; sgdisk --zap-all <device>
partprobe <device>        # or reboot if the kernel still holds the old table
```

**The general lesson.** "New disk" means new to you, not empty. On bare metal,
verify which device the root filesystem is on *before* every destructive
command — the safety check costs seconds and the mistake costs the node.

---

## 4. A config key that is valid in K3s and fatal in RKE2

**Symptom.** A single uncommented line in the server config, a restart, and the
control plane was down — the server process crashed on startup and the API was
unreachable.

**Cause.** The key was `ingress-controller:`, which is **K3s configuration and
invalid in RKE2**. The two distributions share heritage, documentation phrasing
and a great deal of tooling, so a key that looks right and appears in plenty of
search results is simply not part of the schema the server accepts.

**Fix.** In RKE2, the bundled ingress is controlled by the `disable:` list, and
tuned with a `HelmChartConfig` of the same name in `kube-system`. Nothing else.

**The general lesson.** K3s answers are not RKE2 answers. Check the distribution
before copying configuration, and change one key at a time on a single-control-plane
cluster — a bad key there is not a degraded feature, it is an outage.

---

## 5. Moving a cluster to a new network

**Symptom.** After physically relocating the cluster and assigning new addresses,
the control plane would not start. etcd refused to come up, reporting a peer URL
mismatch.

**Cause.** etcd records the addresses of its members, and the API server's
certificates carry the old addresses as SANs. Change the addresses underneath and
both sets of assumptions are wrong at once.

**Fix, in order:**

1. **Before the move:** take an etcd snapshot, back up every node's
   configuration, and shut the cluster down gracefully rather than pulling power.
2. **After the move:** set the new addresses everywhere — static configuration,
   `/etc/hosts`, and the server config's `node-ip` and `advertise-address`, which
   is what pins RKE2 to the intended interface.
3. **Delete the stale certificates** so they are regenerated with the new
   addresses as SANs.
4. **Restore etcd from the snapshot** to reset cluster membership to the new peer
   URLs, then start the control plane.
5. **Rejoin the workers**, clearing their stale state first. The node token is
   unchanged, so their credentials still work.

**The general lesson.** A cluster's IP addresses are part of its identity, stored
in etcd membership and baked into certificates. Treat a re-addressing as a
planned restore-from-snapshot with a rehearsed runbook — not as a network change
that the cluster will simply absorb.

---

## What these have in common

Four of the five were **configuration that was correct somewhere else** — a taint
that is best practice on a multi-master cluster, a webhook that is required with
a different scheduler, a key that is valid in a sibling distribution, addresses
that were right in the old rack. Bare metal is where those assumptions stop
holding quietly and start failing loudly.

The habit that catches them: change one thing, verify it before the next, and
write down what the verification was.
