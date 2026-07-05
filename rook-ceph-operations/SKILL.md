---
name: rook-ceph-operations
description: >
  Manage Rook-orchestrated Ceph clusters on Kubernetes: bootstrapping,
  storage class provisioning, OSD failure recovery, upgrades, node drain
  safety, and health diagnostics using the Rook operator and Ceph toolbox.
  Use when: installing Rook/Ceph, creating a CephCluster, adding OSDs or
  storage pools, troubleshooting HEALTH_WARN or HEALTH_ERR, recovering a
  failed OSD, draining a Kubernetes node that hosts Ceph OSDs, upgrading
  the Rook operator or Ceph version, creating RBD or CephFS storage
  classes, or diagnosing PVC binding failures on Ceph-backed storage.
  Operating model: GitOps/Controller — Rook operator reconciles CephCluster
  and related CRDs from Git. Direct ceph tool commands are used for
  diagnostics and emergency operations only.
  Audience: Platform engineers, SREs.
  Scope: Day-0 (bootstrap) through Day-2 (ongoing operations).
allowed-tools: shell
---

# Rook/Ceph Operations

> **Profile**: GitOps/Controller (Rook operator) + CLI Ops (ceph toolbox diagnostics)
> **Reconciler**: Rook operator (watches CephCluster and related CRDs)
> **Source of truth**: Git repository containing Rook CRD manifests
> **Blast radius**: Cluster-wide (data path) + per-node (OSD operations)

Rook is a Kubernetes operator that manages a Ceph storage cluster. Desired
state is declared in CRDs (`CephCluster`, `CephBlockPool`, `CephFilesystem`,
`CephObjectStore`) committed to Git. The Rook operator reconciles live state
toward the declared spec. Day-2 diagnostics use the Ceph toolbox pod to run
`ceph` CLI commands against the live cluster.

> ⚠️ **Controller ownership rule**: Never `kubectl apply` or directly edit
> Ceph daemons or their configs outside of the Rook CRDs. The operator will
> overwrite direct changes on the next reconciliation loop.

---

## Prerequisites

### Required tools

| Tool | Verify |
|---|---|
| `kubectl` | `kubectl version --client` |
| `helm` (for Helm install path) | `helm version` |
| Rook operator running | `kubectl -n rook-ceph get pod -l app=rook-ceph-operator` |

### Kubernetes version compatibility

| Rook version | Kubernetes versions |
|---|---|
| Latest (v1.17+) | v1.31 – v1.36 |

Full compatibility matrix: https://rook.io/docs/rook/latest/Getting-Started/quickstart/

### Node prerequisites — check before install

Nodes that will host OSDs require raw block devices (no partitions, no
existing filesystem). Verify on each storage node:

```bash
# List available disks (looking for unformatted, unmounted block devices):
lsblk -f

# Check for any leftover Ceph LVM metadata on target disks:
ls /dev/mapper/ | grep ceph
pvdisplay | grep ceph
```

Required kernel modules (loaded automatically on most distros, but verify):
```bash
lsmod | grep -E "rbd|ceph"
```

If missing:
```bash
modprobe rbd
modprobe ceph
```

### Context guard — always confirm namespace and cluster

```bash
kubectl config current-context
kubectl -n rook-ceph get cephcluster
# NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE   HEALTH    EXTERNAL
# rook-ceph   /var/lib/rook     3          ...   Ready   ...       HEALTH_OK
```

> ⚠️ Confirm you are in the correct cluster before any mutating operation.

---

## Day 0 — Install Rook Operator

### Option A: Helm (recommended)

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
helm search repo rook-release/rook-ceph   # note the chart version

# Install operator into rook-ceph namespace:
helm install --create-namespace \
  --namespace rook-ceph \
  rook-ceph rook-release/rook-ceph \
  -f operator-values.yaml
```

### Option B: Manifests

```bash
git clone --single-branch --branch master https://github.com/rook/rook.git
cd rook/deploy/examples

kubectl create -f crds.yaml -f common.yaml -f csi-operator.yaml
kubectl create -f operator.yaml
```

**Verify operator is running before proceeding:**

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-operator
# STATUS must be Running
```

Expected: one `rook-ceph-operator` pod in `Running` state.
**Do not proceed to cluster creation until the operator is Running.**

---

## Day 0 — Create CephCluster

### Minimal production `CephCluster` template

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v20.2.2   # pin to a specific version for production
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook        # must not be /etc/ceph, /rook, or /var/log/ceph
  mon:
    count: 3                            # always odd; 3 minimum for production quorum
    allowMultiplePerNode: false         # false for production
  mgr:
    count: 2                            # 2 for HA (active + standby)
  dashboard:
    enabled: true
    ssl: true
  storage:
    useAllNodes: false                  # explicitly control which nodes/devices
    useAllDevices: false
    nodes:
      - name: "<NODE_NAME>"
        devices:
          - name: "<DEVICE_NAME>"       # e.g., sdb — raw, unformatted
  # (Optional) placement rules, resource limits, network config, etc.
```

> ⚠️ **`dataDirHostPath`**: Must be set for production clusters (survives pod
> restarts). If left empty, config is ephemeral and lost on pod restart.
> Must not be `/etc/ceph`, `/rook`, or `/var/log/ceph`.

```bash
kubectl create -f cluster.yaml
```

**Monitor cluster formation:**

```bash
# Watch pods come up:
kubectl -n rook-ceph get pod -w

# Expected pods when healthy:
# rook-ceph-mon-a/b/c        Running
# rook-ceph-mgr-a/b          Running (2/2 containers)
# rook-ceph-osd-<N>          Running (one per device)
# rook-ceph-osd-prepare-*    Completed (runs once per node)
# rook-ceph.rbd/cephfs CSI   Running
```

### Verify cluster health via toolbox

Deploy the toolbox pod:

```bash
kubectl create -f https://raw.githubusercontent.com/rook/rook/v<ROOK_VERSION>/deploy/examples/toolbox.yaml
# Replace <ROOK_VERSION> with the deployed Rook version (e.g., v1.17.0)
# Verify the version with: kubectl -n rook-ceph get deploy rook-ceph-operator -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

**Expected `ceph status` output for a healthy new cluster:**

```
cluster:
  id:     <uuid>
  health: HEALTH_OK

services:
  mon: 3 daemons, quorum a,b,c (age Xm)
  mgr: a(active, since Xm), standbys: b
  osd: <N> osds: <N> up (since Xm), <N> in (since Xm)
```

Requirements before proceeding:
- [ ] All mons in quorum
- [ ] mgr active
- [ ] All OSDs `up` and `in`
- [ ] `HEALTH_OK` (not WARN, not ERR)

---

## Day 1 — Provision Storage Classes

### Block storage (RWO) — CephBlockPool + StorageClass

```yaml
# ceph-block-pool.yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host     # tolerate 1 full host failure
  replicated:
    size: 3               # 3 replicas across 3 different hosts
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com   # prefix must match operator namespace
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-publish-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-publish-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
kubectl create -f ceph-block-pool.yaml
```

**Verify:**

```bash
kubectl -n rook-ceph get cephblockpool
kubectl get storageclass rook-ceph-block

# Test: create a PVC and verify it binds
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 1Gi
EOF
kubectl get pvc test-pvc   # Status should become Bound
kubectl delete pvc test-pvc
```

### Shared filesystem (RWX) — CephFilesystem + StorageClass

```yaml
# ceph-filesystem.yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  preserveFilesystemOnDelete: true     # prevents accidental data loss
  metadataServer:
    activeCount: 1
    activeStandby: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
kubectl create -f ceph-filesystem.yaml

# Verify MDS pods are running:
kubectl -n rook-ceph get pod -l app=rook-ceph-mds
```

### Object storage (S3-compatible)

```yaml
# ceph-object-store.yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  preservePoolsOnDelete: true
  gateway:
    port: 80
    instances: 1
```

```bash
kubectl create -f ceph-object-store.yaml

# Verify RGW pod is running:
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
```

---

## Day 2 — Draining a Node with OSDs

> ⚠️ This is the most dangerous routine operation. Always complete all steps
> in order. Skipping the health gate or the `noout` flag risks data loss or
> extended rebalancing.

### Step 1 — Check cluster is HEALTH_OK before touching anything

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# health must be HEALTH_OK — do NOT proceed if HEALTH_WARN or HEALTH_ERR
```

**If not HEALTH_OK:** Investigate with `ceph health detail` before proceeding.

### Step 2 — Set `noout` flag to prevent data rebalancing during drain

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd set noout
```

Expected:
```
noout is set
```

Verify the flag is set:
```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd dump | grep flags
# Should include "noout" in the flags line
```

### Step 3 — Cordon and drain the Kubernetes node

```bash
kubectl cordon <NODE_NAME>
kubectl drain <NODE_NAME> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=300s
```

Monitor OSD pods going down on that node:
```bash
kubectl -n rook-ceph get pods -o wide | grep <NODE_NAME>
```

### Step 4 — Perform maintenance

Perform your maintenance work (OS update, hardware replacement, etc.).

### Step 5 — Uncordon and wait for OSDs to come back

```bash
kubectl uncordon <NODE_NAME>

# Watch OSD pods come back on the node:
kubectl -n rook-ceph get pods -o wide -w | grep <NODE_NAME>
```

### Step 6 — Verify OSDs are up and in

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd status
# All OSDs on the node should show: up   in
```

### Step 7 — Unset `noout`

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd unset noout
```

Expected: `noout is unset`

### Step 8 — Verify cluster health returns to HEALTH_OK

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# health: HEALTH_OK
```

> ⚠️ Do not drain another node until HEALTH_OK is confirmed. Never drain
> more nodes than the cluster can tolerate losing (`replicated.size - 1`).

---

## Day 2 — OSD Failure Recovery

### Step 1 — Identify the failed OSD

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# Look for: X osds: Y up, Z in — mismatch indicates a failed OSD

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
# Find OSDs with state: down  out

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
# Shows which OSD IDs are failing and why
```

### Step 2 — Check the OSD pod logs

```bash
# Find the OSD pod for the failed OSD:
kubectl -n rook-ceph get pods -l app=rook-ceph-osd | grep osd-<ID>
kubectl -n rook-ceph logs rook-ceph-osd-<ID>-<hash> --previous
```

### Step 3 — Remove the failed OSD (if device is permanently dead)

```bash
# Mark OSD out to trigger data rebalancing to healthy OSDs:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out osd.<ID>

# Wait for PGs to rebalance (watch until active+clean):
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph pg stat

# Pre-check: confirm Ceph considers this OSD safe to remove.
# Do NOT proceed if this returns an error — data may not be fully replicated yet.
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd safe-to-destroy osd.<ID>
# Expected: "OSD(s) <ID> are safe to destroy without reducing data durability."
# If NOT safe: wait for more PGs to recover, investigate with: ceph health detail

# Purge the dead OSD from the cluster (only after safe-to-destroy passes):
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd purge <ID> --yes-i-really-mean-it
```

> Rook will automatically detect the removed OSD and attempt to provision a
> new one on the same or a replacement device if `useAllDevices: true` or
> the device is listed in the storage spec.

### Step 4 — If the device was replaced: wipe and re-add

```bash
# Wipe LVM metadata left by the old OSD (on the affected node):
# WARNING: run on the affected node only, and only on the replaced device
ls /dev/mapper/ | grep ceph
dmsetup remove /dev/mapper/<ceph-device>
sgdisk --zap-all /dev/<DEVICE>
```

Then update the `CephCluster` CR to include the new device, or let Rook
auto-discover it if `useAllDevices: true`.

### Step 5 — Verify recovery

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# New OSD should appear: up  in
# health: HEALTH_OK (may take time while PGs rebalance)
```

---

## Day 2 — Rook and Ceph Upgrades

> **Rule**: Upgrade Rook operator first, then Ceph version. Never skip
> major versions.

### Step 1 — Back up CephCluster state

```bash
# Export current CRD definitions:
kubectl -n rook-ceph get cephcluster -o yaml > cephcluster-backup.yaml
kubectl -n rook-ceph get cephblockpool -o yaml > cephblockpool-backup.yaml
kubectl -n rook-ceph get cephfilesystem -o yaml > cephfilesystem-backup.yaml
```

### Step 2 — Verify cluster is HEALTH_OK before upgrading

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

**Do not upgrade if health is not HEALTH_OK.**

### Step 3 — Upgrade the Rook operator

#### Helm path:

```bash
helm repo update
helm upgrade rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --version <NEW_CHART_VERSION> \
  -f operator-values.yaml
```

#### Manifest path:

```bash
kubectl apply -f crds.yaml -f common.yaml -f csi-operator.yaml
kubectl apply -f operator.yaml
```

**Wait for operator to reconcile:**

```bash
kubectl -n rook-ceph rollout status deploy/rook-ceph-operator
```

### Step 4 — Upgrade Ceph version

Update `spec.cephVersion.image` in the CephCluster CR:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph \
  --type merge \
  -p '{"spec":{"cephVersion":{"image":"quay.io/ceph/ceph:v<NEW_VERSION>"}}}'
```

**Monitor the rolling upgrade:**

```bash
# Watch daemon pods roll:
kubectl -n rook-ceph get pods -w

# Monitor cluster health during upgrade:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- watch ceph status
```

**Verify upgrade completion:**

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph version
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# health: HEALTH_OK
```

---

## Observability — Toolbox Commands

The Ceph toolbox is the primary diagnostic tool. Deploy it if not running:

```bash
# Use the toolbox manifest matching your deployed Rook version:
ROOK_VERSION=$(kubectl -n rook-ceph get deploy rook-ceph-operator \
  -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)
kubectl create -f https://raw.githubusercontent.com/rook/rook/${ROOK_VERSION}/deploy/examples/toolbox.yaml
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

### Health and status

```bash
# Overall cluster health and service summary:
ceph status

# Detailed health warnings and errors:
ceph health detail

# OSD status (up/in/out/down per OSD):
ceph osd status

# OSD tree (shows failure domain layout):
ceph osd tree

# Placement group states (watch for degraded, recovering, backfill):
ceph pg stat

# Disk usage:
ceph df

# Pool-level usage:
ceph df detail
```

### Alerts and their meanings

| Ceph health message | Meaning | Action |
|---|---|---|
| `HEALTH_OK` | All services healthy, all data replicated | No action needed |
| `HEALTH_WARN: <N> osds down` | OSD process stopped | Check OSD pod logs; see OSD recovery section |
| `HEALTH_WARN: nearfull` | OSD usage > `mon_osd_nearfull_ratio` (default 85%) | Add OSDs or delete data |
| `HEALTH_ERR: <N> osds full` | OSD usage > `mon_osd_full_ratio` (default 95%) | **Emergency**: Ceph will stop writes. Delete data or add OSDs immediately |
| `HEALTH_WARN: clock skew` | Node clocks out of sync | Sync NTP on affected nodes |
| `HEALTH_WARN: noout flag(s) set` | Manual noout set — expected after a drain | Unset with `ceph osd unset noout` when node is back |
| `HEALTH_WARN: pool has too few PGs` | Pool PG count too low for OSD count | Increase PGs (requires downtime planning) |
| `HEALTH_ERR: <N> pgs inconsistent` | PG data mismatch detected | Run `ceph pg repair <pgid>` for each |

### Monitoring Rook operator and daemon logs

```bash
# Rook operator logs:
kubectl -n rook-ceph logs deploy/rook-ceph-operator -f

# Individual daemon logs:
kubectl -n rook-ceph logs rook-ceph-mon-a-<hash> -f
kubectl -n rook-ceph logs rook-ceph-osd-0-<hash> -f

# All failing pods:
kubectl -n rook-ceph get pods | grep -v Running | grep -v Completed
```

### Kubernetes-level PVC diagnostics

```bash
# Check PVC status:
kubectl get pvc -A | grep -v Bound

# For a pending PVC, check events:
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>

# Check CSI provisioner logs (resolve pod by label — deployment name varies by Rook version):
kubectl -n rook-ceph get pods -l app=csi-rbdplugin-provisioner
kubectl -n rook-ceph logs -l app=csi-rbdplugin-provisioner \
  -c csi-provisioner --tail=100
```

---

## Day 2 — Expanding Cluster Capacity

### Adding OSDs on existing nodes (new device added to a node)

If `useAllDevices: false`, add the new device to the `CephCluster` storage spec:

```yaml
spec:
  storage:
    nodes:
      - name: "<NODE_NAME>"
        devices:
          - name: "sdb"   # existing
          - name: "sdc"   # new device
```

Apply the change:
```bash
kubectl apply -f cluster.yaml

# Watch for new OSD prepare job and OSD pod:
kubectl -n rook-ceph get pods -w | grep osd-prepare
kubectl -n rook-ceph get pods | grep osd
```

Verify new OSD is up and in:
```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
# OSD count should increase; Ceph starts rebalancing automatically
```

### Adding a new storage node

1. Label the node (if using node affinity in the CephCluster):
   ```bash
   kubectl label node <NEW_NODE> role=storage-node
   ```

2. Add the node to the `CephCluster` storage spec:
   ```yaml
   spec:
     storage:
       nodes:
         - name: "<NEW_NODE>"
           devices:
             - name: "<DEVICE>"
   ```

3. Apply and verify:
   ```bash
   kubectl apply -f cluster.yaml
   kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- watch ceph status
   ```

---

## Troubleshooting Common Failures

### OSDs not created after install (osd-prepare pods fail or complete without OSDs)

```bash
# Check prepare job logs:
kubectl -n rook-ceph logs -l app=rook-ceph-osd-prepare --previous

# Common causes:
# - Device already has a filesystem or partition table:
lsblk -f <DEVICE>   # check on the node — must be empty
# - Leftover Ceph LVM from a previous cluster:
ls /dev/mapper/ | grep ceph
pvdisplay | grep ceph   # must be clean

# Fix: wipe the device on the node:
sgdisk --zap-all /dev/<DEVICE>
wipefs -a /dev/<DEVICE>
dd if=/dev/zero of=/dev/<DEVICE> bs=1M count=10   # wipe LVM header
```

### PVC stuck in Pending

```bash
# Check PVC events:
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
# Look for: "waiting for first consumer", provisioner errors, pool issues

# Check the StorageClass exists:
kubectl get storageclass

# Check CSI provisioner pod:
kubectl -n rook-ceph get pods -l app=csi-rbdplugin-provisioner
kubectl -n rook-ceph logs -l app=csi-rbdplugin-provisioner \
  -c csi-provisioner --tail=100

# Check CephBlockPool health:
kubectl -n rook-ceph get cephblockpool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls
```

### PVC stuck in Terminating

```bash
# PVCs with Ceph RBD stuck in Terminating usually have a finalizer
kubectl get pvc <PVC_NAME> -n <NAMESPACE> -o yaml | grep finalizer

# Remove the finalizer (only after confirming no active mounts):
kubectl patch pvc <PVC_NAME> -n <NAMESPACE> \
  -p '{"metadata":{"finalizers":null}}'

# Also check for a corresponding PV stuck in Released:
kubectl get pv | grep <PVC_NAME>
kubectl delete pv <PV_NAME>

# The underlying RBD image may need manual cleanup:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls replicapool
```

### Mons failing to form quorum

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
kubectl -n rook-ceph logs rook-ceph-mon-a-<hash> --previous

# Common causes:
# - Clock skew between nodes (must be < 0.05 seconds for Ceph)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
# HEALTH_WARN: clock skew → sync NTP on all nodes

# - Mon count mismatch (fewer than count running)
kubectl -n rook-ceph get cephcluster -o yaml | grep monCount
# If a mon pod is crash-looping, check its data dir on the host:
# dataDirHostPath/namespace/mon-N/ must be writable
```

### MDS pods crash-looping (CephFS issues)

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds
kubectl -n rook-ceph logs rook-ceph-mds-myfs-a-<hash> --previous

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs status
# Look for: MDS rank not active, journal issues

# Reset a stuck MDS (last resort):
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mds fail <MDS_NAME>
```

---

## Rollback / Recovery

> **Reversibility**: Partially reversible. CRD changes can be reverted in Git.
> OSD and pool data deletions are **irreversible**.

### Revert CRD changes (the Git path — preferred)

```bash
git revert <commit>
kubectl apply -f cephcluster.yaml   # reapply previous spec
# Operator will reconcile toward the reverted spec
```

### Emergency: operator crash-looping

```bash
# Check operator status:
kubectl -n rook-ceph describe pod -l app=rook-ceph-operator

# If a bad CRD update caused the loop, revert the CRD change in Git and apply:
kubectl apply -f crds.yaml

# Force pod restart:
kubectl -n rook-ceph rollout restart deploy/rook-ceph-operator
```

### Disaster recovery notes

- Ceph stores all state in OSDs and mons — as long as the data disks are intact
  and `dataDirHostPath` is preserved, the cluster can be recovered
- If all mons are lost: follow the Rook
  [mon disaster recovery guide](https://rook.io/docs/rook/latest/Troubleshooting/disaster-recovery/#restoring-mon-quorum)
- Never delete the `rook-ceph` namespace without first deleting all CRDs and
  wiping OSD disks — leftover LVM metadata will block fresh installs

---

## Cleanup / Decommission

> ⚠️ Destructive and irreversible. All data in Ceph will be permanently lost.

```bash
# 1. Delete all application PVCs first, to avoid stuck finalizers
# 2. Delete storage resources in reverse order:
kubectl delete -n rook-ceph cephobjectstore --all
kubectl delete -n rook-ceph cephfilesystem --all
kubectl delete -n rook-ceph cephblockpool --all
kubectl delete -n rook-ceph cephcluster rook-ceph

# 3. Wait for cluster to be deleted, then delete operator:
helm uninstall rook-ceph -n rook-ceph   # or: kubectl delete -f operator.yaml

# 4. Delete CRDs:
kubectl delete -f crds.yaml

# 5. On EACH storage node, wipe the disks Ceph used:
# (run on the node directly or via a DaemonSet job)
DISK="/dev/sdX"
sgdisk --zap-all $DISK
dd if=/dev/zero of=$DISK bs=1M count=100 oflag=direct,dsync
ls /dev/mapper/ | grep ceph | xargs -I{} dmsetup remove {}
rm -rf /var/lib/rook/*   # if using default dataDirHostPath

# 6. Delete namespace:
kubectl delete namespace rook-ceph
```

Reference: https://rook.io/docs/rook/latest/Getting-Started/ceph-teardown/

---

## Security & Access

- **Rook operator RBAC**: managed by the Helm chart / `common.yaml` — do not
  modify manually; changes will be overwritten
- **Ceph admin credentials**: stored as Kubernetes Secrets in `rook-ceph`
  namespace (e.g., `rook-ceph-admin-keyring`) — restrict access to these secrets
- **Dashboard access**: default credentials stored in Secret `rook-ceph-dashboard-password`
  ```bash
  kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
    -o jsonpath="{['data']['password']}" | base64 --decode
  ```
- **Minimum operator permissions**: `cluster-admin` is typically required during
  initial CRD and RBAC setup; post-install the operator runs with a scoped SA
- **Network policy**: consider restricting Ceph port access (6789/mon, 6800-7300/OSD)
  to storage nodes and the Kubernetes API only

---

## Environment Variations

| Aspect | Lab / dev | Production |
|---|---|---|
| Mon count | 1 (test only) | 3 minimum (odd number for quorum) |
| OSD replicated size | 1 | 3 (across different hosts) |
| `allowMultiplePerNode` | `true` acceptable | `false` — one OSD daemon per physical node |
| `dataDirHostPath` | ephemeral (empty) | Always set — e.g., `/var/lib/rook` |
| Node drain | Skip noout is acceptable | Always set noout + health gate |
| Upgrades | Skip health checks if needed | Always HEALTH_OK before and after each step |
| PodDisruptionBudgets | Not required | Review for OSD + mon PDBs before draining |

---

## Anti-patterns — Never Do These

- [ ] Do not drain a node without first setting `ceph osd set noout`
- [ ] Do not drain more nodes than the cluster can tolerate simultaneously
- [ ] Do not proceed with upgrades when health is HEALTH_WARN or HEALTH_ERR
- [ ] Do not delete a `CephCluster` without first wiping OSD disks — leftover LVM metadata blocks reinstall
- [ ] Do not delete the Rook CRDs while the CephCluster still has data
- [ ] Do not use `useAllDevices: true` in production without auditing all node disks — it will consume any available unformatted block device
- [ ] Do not set `allowUnsupported: true` in production unless you know the risk
- [ ] Do not skip waiting for HEALTH_OK between upgrade steps
- [ ] Do not `kubectl apply` directly to Ceph daemon configs — the operator owns them

---

## References

- [Rook documentation](https://rook.io/docs/rook/latest/)
- [Quickstart / install guide](https://rook.io/docs/rook/latest/Getting-Started/quickstart/)
- [Node prerequisites](https://rook.io/docs/rook/latest/Getting-Started/Prerequisites/prerequisites/)
- [CephCluster CRD reference](https://rook.io/docs/rook/latest/CRDs/Cluster/ceph-cluster-crd/)
- [CephBlockPool CRD reference](https://rook.io/docs/rook/latest/CRDs/Block-Storage/ceph-block-pool-crd/)
- [CephFilesystem CRD reference](https://rook.io/docs/rook/latest/CRDs/Shared-Filesystem/ceph-filesystem-crd/)
- [Block storage (RBD)](https://rook.io/docs/rook/latest/Storage-Configuration/Block-Storage-RBD/block-storage/)
- [Shared filesystem (CephFS)](https://rook.io/docs/rook/latest/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/)
- [Rook upgrade guide](https://rook.io/docs/rook/latest/Upgrade/rook-upgrade/)
- [Toolbox](https://rook.io/docs/rook/latest/Troubleshooting/ceph-toolbox/)
- [Ceph common issues](https://rook.io/docs/rook/latest/Troubleshooting/ceph-common-issues/)
- [Disaster recovery](https://rook.io/docs/rook/latest/Troubleshooting/disaster-recovery/)
- [Cluster teardown](https://rook.io/docs/rook/latest/Getting-Started/ceph-teardown/)
- [Helm operator chart](https://rook.io/docs/rook/latest/Helm-Charts/operator-chart/)
- [GitHub: rook/rook](https://github.com/rook/rook)
