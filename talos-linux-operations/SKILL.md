---
name: talos-linux-operations
description: >
  Manage Talos Linux clusters: bootstrapping, machine config authoring and
  patching, Talos OS upgrades, Kubernetes version upgrades, etcd maintenance,
  and disaster recovery using talosctl.
  Use when: bootstrapping a new Talos cluster, applying or patching machine
  configs, upgrading Talos OS version, upgrading Kubernetes version, backing
  up or restoring etcd, troubleshooting Talos node health, rotating cluster
  CAs, managing etcd quorum or disk space, or recovering from a node failure.
  Operating model: Declarative machine configs in Git + imperative CLI ops
  via talosctl. No SSH — all management via mTLS gRPC on port 50000.
  Audience: Platform engineers, SREs.
  Scope: Day-0 (bootstrap) through Day-2 (ongoing operations).
allowed-tools: shell
---

# Talos Linux Operations

> **Profile**: Declarative/IaC (machine configs) + CLI Ops (talosctl)
> **Source of truth**: Git repository containing machine config patches
> **Auth model**: `talosconfig` — mTLS client certificate, gRPC port 50000
> **Blast radius**: Node-scoped (single node) to cluster-wide (etcd/CA ops)
> **No SSH**: Talos is an immutable OS with no shell. All operations use `talosctl`.

Talos Linux is an API-driven, immutable Kubernetes OS. Cluster state is
declared in `controlplane.yaml` / `worker.yaml` machine configs and applied
via `talosctl`. Day-2 changes flow through patch files committed to Git and
applied with `talosctl patch mc` or `talosctl apply-config`.

> ⚠️ **Source-of-truth rule**: Store patch files (deltas) in Git, not full
> generated machine configs. Re-applying a stale full config after
> `talosctl upgrade-k8s` will downgrade control plane component versions.

---

## Prerequisites

### Required tools

| Tool | Verify |
|---|---|
| `talosctl` | `talosctl version --client` |
| `kubectl` | `kubectl version --client` |
| `talosconfig` | `TALOSCONFIG=~/.talos/config talosctl config contexts` |

Install `talosctl` matching your cluster version:

```bash
# Linux/macOS — replace VERSION with your cluster version:
curl -Lo /usr/local/bin/talosctl \
  https://github.com/siderolabs/talos/releases/download/v<VERSION>/talosctl-$(uname -s | tr '[:upper:]' '[:lower:]')-amd64
chmod +x /usr/local/bin/talosctl
```

> Use the `talosctl` version that matches the running cluster. Mismatched
> versions can cause API compatibility issues.

### Version compatibility

| Talos version | Supported Kubernetes versions |
|---|---|
| v1.10 | 1.28 – 1.33 |
| v1.9 | 1.27 – 1.32 |

Full matrix: https://docs.siderolabs.com/talos/v1.10/getting-started/support-matrix

### Context guard — always confirm target before any operation

```bash
talosctl config contexts                   # list all contexts
talosctl config context <context-name>     # switch context
talosctl --talosconfig ~/.talos/config \
  config endpoints                         # show current endpoints
```

> ⚠️ Confirm you are targeting the correct cluster and nodes before every
> mutating operation. Talos has no undo for destructive commands.

---

## Day 0 — Bootstrap a New Cluster

### Step 1 — Generate secrets bundle

Generate once and store securely. This is the root of trust for the cluster.

```bash
talosctl gen secrets -o secrets.yaml
```

> ⚠️ Back up `secrets.yaml` securely (password manager, secrets vault).
> Losing it means you cannot regenerate a matching talosconfig. Never commit
> it to Git.

### Step 2 — Generate machine configs

```bash
talosctl gen config <cluster-name> https://<CONTROL_PLANE_ENDPOINT>:6443 \
  --with-secrets secrets.yaml \
  --install-disk /dev/sda \
  --config-patch @all.yaml \
  --config-patch-control-plane @controlplane.yaml \
  --config-patch-worker @worker.yaml \
  --output ./generated/
```

Key flags:
```
--with-secrets string         use pre-generated secrets bundle (always use in prod)
--install-disk string         disk for OS install (verify with talosctl get disks)
--kubernetes-version string   pin K8s version (default: latest stable)
--additional-sans strings     extra SANs for kube-apiserver cert (LB IPs, FQDNs)
--config-patch @file.yaml     patch applied to all node types
--config-patch-control-plane  patch applied to controlplane only
--config-patch-worker         patch applied to workers only
```

**Validate before using:**

```bash
talosctl validate -c generated/controlplane.yaml -m metal --strict
talosctl validate -c generated/worker.yaml -m metal --strict
```

Expected: no output (exit code 0). Any output is a validation error.

### Step 3 — Apply configs (maintenance mode, before TLS is established)

```bash
# Control plane nodes (run for each CP node IP):
talosctl apply-config --insecure \
  --nodes <CP_NODE_IP> \
  --file generated/controlplane.yaml

# Worker nodes:
talosctl apply-config --insecure \
  --nodes <WORKER_NODE_IP> \
  --file generated/worker.yaml
```

> `--insecure` connects to the pre-auth maintenance service (port 50000).
> Only valid before a config is applied. Never use `--insecure` on a running
> configured node — the maintenance service is inactive after first apply.

### Step 4 — Bootstrap etcd (run EXACTLY ONCE on ONE control plane node)

```bash
talosctl bootstrap \
  --nodes <FIRST_CP_NODE_IP> \
  --talosconfig ./generated/talosconfig
```

> ⚠️ **Critical**: Run `talosctl bootstrap` exactly once, on one node.
> Running it again or on multiple nodes simultaneously corrupts etcd.

**Wait for etcd to come up:**

```bash
talosctl service -n <CP_NODE_IP> etcd
# Wait until STATE=Running, HEALTH=OK
```

### Step 5 — Retrieve kubeconfig

```bash
talosctl kubeconfig \
  --nodes <CP_NODE_IP> \
  --talosconfig ./generated/talosconfig
# Merges into ~/.kube/config by default
```

**Verify cluster access:**

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

Expected: all nodes `Ready`, all kube-system pods `Running`.

### Step 6 — Full health check

```bash
talosctl health \
  --control-plane-nodes <CP1>,<CP2>,<CP3> \
  --worker-nodes <W1>,<W2>
```

Expected: `ok` for all checks. Do not proceed with workload deployment until
this passes.

---

## Day 1 — Machine Config Management

### How machine configs work in Git

Store **patch files** (not full generated configs) in Git. Apply them with
`talosctl patch mc` or rebuild full configs with `talosctl machineconfig patch`.

```
git-repo/
├── secrets.yaml          # NEVER commit — store in secrets vault
├── patches/
│   ├── all.yaml          # patches applied to all nodes
│   ├── controlplane.yaml # patches applied to CP nodes only
│   └── worker.yaml       # patches applied to workers only
└── generated/            # NEVER commit — regenerate from patches + secrets
    ├── controlplane.yaml
    ├── worker.yaml
    └── talosconfig
```

### Applying a config change to a running node

**Always dry-run first:**

```bash
talosctl apply-config \
  --nodes <NODE_IP> \
  --file generated/controlplane.yaml \
  --dry-run
```

Expected: diff output showing the proposed changes. Review every field.

**Determine the correct apply mode:**

| What is changing | Use mode |
|---|---|
| Network, kubelet args, labels, sysctls, time | `--mode=no-reboot` |
| Install disk, kernel args, CA certs, KubeSpan | `--mode=reboot` |
| Unknown / let Talos decide | `--mode=auto` (default) |
| Stage for next scheduled reboot | `--mode=staged` |
| Test change with auto-revert in 1 min | `--mode=try` |

**Apply:**

```bash
talosctl apply-config \
  --nodes <NODE_IP> \
  --file generated/controlplane.yaml \
  --mode=no-reboot    # adjust as needed
```

**Verify after apply:**

```bash
talosctl get machineconfig v1alpha1 -n <NODE_IP> -o yaml
talosctl service -n <NODE_IP>
```

### Patching a live node (strategic merge patch)

Preferred for targeted changes without regenerating full configs:

```bash
# Dry-run:
talosctl patch mc \
  -n <NODE_IP> \
  --patch @patches/my-change.yaml \
  --dry-run

# Apply:
talosctl patch mc \
  -n <NODE_IP> \
  --patch @patches/my-change.yaml \
  --mode=no-reboot
```

Example strategic merge patch file:

```yaml
# patches/ntp-servers.yaml
machine:
  time:
    servers:
      - time.cloudflare.com
      - time.google.com
```

> **Strategic merge patches** are recommended over JSON patches. They look
> like partial machine configs and deep-merge into the running config.
> Lists are appended except: `podSubnets`, `serviceSubnets`, and
> `auditPolicy` are replaced on merge.

### Patching a config file offline (without touching a live node)

```bash
talosctl machineconfig patch generated/worker.yaml \
  --patch @patches/my-change.yaml \
  -o generated/worker-patched.yaml
```

---

## Day 2 — Talos OS Upgrade

> **Rule**: Upgrade between adjacent minor versions only.
> v1.7 → v1.8 → v1.9, not v1.7 → v1.9 directly.

### Step 1 — Back up etcd before any upgrade

```bash
talosctl etcd snapshot -n <ANY_HEALTHY_CP_IP> ./etcd-backup-$(date +%Y%m%d).snapshot
```

Expected: `etcd snapshot saved to "etcd-backup-<date>.snapshot" (<N> bytes)`

**Also back up the running machine config:**

```bash
talosctl get machineconfig v1alpha1 -n <CP_IP> -o yaml > machineconfig-backup-$(date +%Y%m%d).yaml
```

### Step 2 — Check cluster health

```bash
talosctl health \
  --control-plane-nodes <CP1>,<CP2>,<CP3> \
  --worker-nodes <W1>,<W2>

talosctl etcd status -n <CP1>,<CP2>,<CP3>
talosctl etcd members -n <CP1>
```

Expected: all health checks pass, etcd shows all members healthy, no `ERRORS`.
**Do not proceed if any check fails.**

### Step 3 — Upgrade control plane nodes (one at a time)

Find installer image at https://factory.talos.dev/ or use:
`ghcr.io/siderolabs/installer:v<VERSION>`

```bash
# Upgrade first CP node and wait:
talosctl upgrade \
  --nodes <CP1_IP> \
  --image ghcr.io/siderolabs/installer:v<NEW_VERSION> \
  --wait

# Verify health before proceeding:
talosctl health --control-plane-nodes <CP1>,<CP2>,<CP3> --worker-nodes <W1>,<W2>
talosctl etcd status -n <CP1>,<CP2>,<CP3>

# Upgrade second CP node (only when first is healthy):
talosctl upgrade \
  --nodes <CP2_IP> \
  --image ghcr.io/siderolabs/installer:v<NEW_VERSION> \
  --wait

# Verify, then upgrade third CP node:
talosctl health ...
talosctl upgrade --nodes <CP3_IP> --image ghcr.io/siderolabs/installer:v<NEW_VERSION> --wait
```

> Talos uses A/B boot slots. If the new image fails to boot, the bootloader
> automatically reverts. Manual rollback: `talosctl rollback -n <NODE_IP>`.
> Talos refuses to upgrade a CP node that would break etcd quorum (unless
> `--force` is used — never use `--force` on multiple CP nodes simultaneously).

### Step 4 — Upgrade worker nodes

```bash
talosctl upgrade \
  --nodes <W1_IP> \
  --image ghcr.io/siderolabs/installer:v<NEW_VERSION> \
  --wait

# Repeat for each worker node. Check workload health between nodes.
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running
```

### Rollback

```bash
# Manual rollback to previous A/B slot:
talosctl rollback -n <NODE_IP>
```

> Rollback only works if the node has not been double-upgraded past the
> previous slot.

---

## Day 2 — Kubernetes Version Upgrade

### Step 1 — Dry-run

```bash
talosctl upgrade-k8s \
  --nodes <ANY_CP_IP> \
  --to <TARGET_K8S_VERSION> \
  --dry-run
```

Review the plan output. Confirm component image versions and expected changes.

### Step 2 — Upgrade (run against one CP node — all nodes upgraded automatically)

```bash
talosctl upgrade-k8s \
  --nodes <ANY_CP_IP> \
  --to <TARGET_K8S_VERSION>
```

The command runs 6 phases:
1. Pre-pull images to all nodes
2. Patch control plane component images in machine configs
3. Update kube-proxy DaemonSet
4. Update kubelet on all nodes, wait for healthy
5. Re-apply bootstrap manifests

**Verify after:**

```bash
kubectl get nodes -o wide
# All nodes should show the new Kubernetes version in the VERSION column
talosctl health --control-plane-nodes <CP1>,<CP2>,<CP3> --worker-nodes <W1>,<W2>
```

---

## Day 2 — etcd Maintenance

### Check etcd health

```bash
# Status across all control plane nodes:
talosctl etcd status -n <CP1>,<CP2>,<CP3>
# Check: DB SIZE, IN USE, LEADER, RAFT INDEX, RAFT TERM, ERRORS (must be empty)

# Member list:
talosctl etcd members -n <CP1>
# All members should be present with non-empty client URLs
```

### Defragment etcd (run ONE node at a time — blocks I/O during defrag)

```bash
# Check for NOSPACE alarm first:
talosctl etcd alarm list -n <CP1>

# Defragment:
talosctl etcd defrag -n <CP1>
talosctl etcd status -n <CP1>    # verify DB SIZE reduced

# Repeat for each CP node sequentially (not in parallel):
talosctl etcd defrag -n <CP2>
talosctl etcd defrag -n <CP3>

# Disarm NOSPACE alarm after defrag:
talosctl etcd alarm disarm -n <CP1>
```

> Default etcd quota is 2 GiB. To increase it, add to machine config:
> ```yaml
> cluster:
>   etcd:
>     extraArgs:
>       quota-backend-bytes: "4294967296"  # 4 GiB (max recommended: 8 GiB)
> ```

### Transfer etcd leadership before rebooting a leader node

```bash
# Check who the leader is:
talosctl etcd status -n <CP1>,<CP2>,<CP3>
# LEADER column shows true/false

# Transfer leadership away before rebooting the leader:
talosctl etcd forfeit-leadership -n <LEADER_IP>

# Verify leadership moved:
talosctl etcd status -n <CP1>,<CP2>,<CP3>
```

---

## Disaster Recovery — etcd Restore

### Backup procedure (run regularly, automate if possible)

```bash
talosctl etcd snapshot -n <ANY_HEALTHY_CP_IP> ./etcd-backup.snapshot
```

### Restore from snapshot (quorum lost or corruption)

**Pre-conditions:**

```bash
# 1. Verify node types (no legacy init-type nodes):
talosctl get machinetype -n <CP1>,<CP2>,<CP3>
# All should show: controlplane

# 2. Wipe EPHEMERAL partition on broken CP nodes:
talosctl reset -n <BROKEN_CP_IP> \
  --graceful=false \
  --reboot \
  --system-labels-to-wipe EPHEMERAL

# 3. Wait for etcd to enter Preparing state on wiped nodes:
talosctl service -n <CP_IP> etcd
# STATE should show: Preparing
```

**Restore:**

```bash
# From consistent snapshot:
talosctl bootstrap -n <CP_IP> --recover-from ./etcd-backup.snapshot

# From emergency direct copy (skip hash check):
talosctl bootstrap -n <CP_IP> \
  --recover-from ./etcd-emergency.db \
  --recover-skip-hash-check
```

**Verify recovery:**

```bash
talosctl etcd members -n <CP_IP>
talosctl health --control-plane-nodes <CP1>,<CP2>,<CP3>
kubectl get nodes
```

---

## CA Rotation

```bash
# Dry-run (default — always review output first):
talosctl rotate-ca \
  --control-plane-nodes <CP1>,<CP2>,<CP3> \
  --worker-nodes <W1>,<W2> \
  --output new-talosconfig

# Apply for real:
talosctl rotate-ca \
  --control-plane-nodes <CP1>,<CP2>,<CP3> \
  --worker-nodes <W1>,<W2> \
  --output new-talosconfig \
  --dry-run=false
```

> `--dry-run` defaults to `true`. You must explicitly set `--dry-run=false`
> to apply the rotation.

---

## Observability

### Cluster-level health

```bash
# Full cluster health check:
talosctl health \
  --control-plane-nodes <CP1>,<CP2>,<CP3> \
  --worker-nodes <W1>,<W2>

# Kubernetes node status:
kubectl get nodes -o wide

# All system pods:
kubectl get pods -n kube-system
```

### Node-level service status

```bash
talosctl service -n <NODE_IP>              # all services
talosctl service -n <NODE_IP> etcd         # etcd specifically
talosctl service -n <NODE_IP> kubelet
talosctl service -n <NODE_IP> apid
```

Key services and their roles:

| Service | Purpose |
|---|---|
| `etcd` | Kubernetes state store (CP only) |
| `kubelet` | Node agent |
| `apid` | Talos API daemon |
| `machined` | System state machine |
| `containerd` | Container runtime |
| `networkd` | Network configuration |
| `trustd` | PKI / certificate service (CP only) |

### Logs

```bash
# Kernel messages (boot issues, OOM, driver errors):
talosctl dmesg -n <NODE_IP> --follow

# Service logs:
talosctl logs -n <NODE_IP> kubelet --follow
talosctl logs -n <NODE_IP> etcd --follow
talosctl logs -n <NODE_IP> apid

# API server (Kubernetes-side):
kubectl logs -n kube-system kube-apiserver-<node-name> --follow
```

### Diagnostics bundle

```bash
talosctl support -n <NODE1>,<NODE2> -O support-bundle.tar.gz
```

### Useful `talosctl get` queries

```bash
talosctl get machineconfig v1alpha1 -n <NODE_IP> -o yaml  # running config
talosctl get members -n <NODE_IP>                          # discovered cluster members
talosctl get services -n <NODE_IP>                         # all service states
talosctl get links -n <NODE_IP>                            # network interfaces
talosctl get addresses -n <NODE_IP>                        # assigned IPs
talosctl get routes -n <NODE_IP>                           # routing table
talosctl get disks -n <NODE_IP> --insecure                 # disk devices (pre-config)
```

### Health signals

| Signal | Meaning |
|---|---|
| `talosctl health` — all `ok` | Cluster fully healthy |
| `talosctl etcd status` — `ERRORS` column empty | etcd healthy |
| `talosctl service etcd` — `STATE=Running, HEALTH=OK` | etcd service healthy |
| `talosctl service etcd` — `STATE=Preparing` | etcd starting or waiting for quorum |
| `kubectl get nodes` — all `Ready` | All nodes healthy from Kubernetes perspective |
| `talosctl dmesg` — `OOM` in kernel log | Node under memory pressure |
| `talosctl etcd alarm list` — `NOSPACE` present | etcd quota exceeded — defrag required |

---

## Troubleshooting Common Failures

### Node not becoming Ready after bootstrap or join

```bash
# 1. Check Talos services:
talosctl service -n <NODE_IP>
# Look for services in FAILED or STOPPED state

# 2. Check kubelet logs:
talosctl logs -n <NODE_IP> kubelet --follow
# Common causes: wrong container runtime, CNI not installed, API endpoint unreachable

# 3. Check machined/networkd:
talosctl logs -n <NODE_IP> machined --follow
talosctl logs -n <NODE_IP> networkd --follow

# 4. Verify the node can reach the control plane endpoint:
talosctl get addresses -n <NODE_IP>   # check assigned IPs
talosctl get routes -n <NODE_IP>      # verify routing

# 5. On the K8s side:
kubectl describe node <NODE_NAME>     # look at Conditions and Events
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

**Common causes:**
- CNI not installed: node shows `NetworkPluginNotReady`. Install your CNI (Cilium, Calico, etc.) after bootstrap.
- Wrong `cluster.controlPlane.endpoint` in machine config: node cannot reach API server.
- Missing `--additional-sans` for the LB IP: kubelet certificate rejected by API server.

### etcd service won't start (STATE=Preparing indefinitely)

```bash
talosctl service -n <CP_IP> etcd
talosctl logs -n <CP_IP> etcd --follow
talosctl etcd members -n <ANY_OTHER_HEALTHY_CP_IP>

# Common cause: node was reset but not removed from etcd membership
# Check if the node appears as a dead member:
talosctl etcd members -n <HEALTHY_CP_IP>
# Remove dead member (if node is unreachable):
talosctl etcd remove-member -n <HEALTHY_CP_IP> <MEMBER_ID>
```

### Upgrade stuck — node not uncordoning after upgrade

```bash
# Check Talos node state:
talosctl service -n <NODE_IP> kubelet
talosctl logs -n <NODE_IP> kubelet --follow

# Check if K8s sees the node:
kubectl get node <NODE_NAME>
kubectl describe node <NODE_NAME>

# Check boot state:
talosctl dmesg -n <NODE_IP> --follow
# Look for: panic, BUG, filesystem errors

# If node booted the new image but kubelet failed:
talosctl service -n <NODE_IP> containerd
talosctl service -n <NODE_IP> kubelet

# Manual rollback if image is broken:
talosctl rollback -n <NODE_IP>
```

### Config apply fails with "permission denied" or "connection refused"

```bash
# Verify talosconfig is pointing at the correct endpoint:
talosctl config contexts
talosctl --talosconfig ~/.talos/config config endpoints

# Verify node is reachable on port 50000:
talosctl version --nodes <NODE_IP>   # "connection refused" = node not running talos API

# If using --insecure on a node that already has a config applied:
# --insecure only works in maintenance mode (before first config apply)
# Remove --insecure and ensure talosconfig has valid certs
```

### etcd NOSPACE alarm — cluster refusing writes

```bash
talosctl etcd alarm list -n <CP_IP>
# If NOSPACE alarm present:

# 1. Defrag ONE node at a time:
talosctl etcd defrag -n <CP1>
talosctl etcd status -n <CP1>   # verify DB SIZE reduced

talosctl etcd defrag -n <CP2>
talosctl etcd defrag -n <CP3>

# 2. Clear the alarm:
talosctl etcd alarm disarm -n <CP1>
talosctl etcd alarm list -n <CP1>   # should be empty

# 3. Prevent recurrence — increase quota (apply to controlplane nodes):
# Add to machine config patch:
# cluster:
#   etcd:
#     extraArgs:
#       quota-backend-bytes: "4294967296"   # 4 GiB
```

---



> ⚠️ `talosctl reset` is destructive and partially irreversible. Always
> take an etcd snapshot before resetting any control plane node.

```bash
# Take etcd snapshot first:
talosctl etcd snapshot -n <ANY_HEALTHY_CP_IP> ./etcd-pre-reset.snapshot

# Graceful reset (default — cordons, drains, leaves etcd):
talosctl reset -n <NODE_IP>

# Non-graceful (node unreachable or single-node recovery):
talosctl reset -n <NODE_IP> --graceful=false --reboot

# Cloud VM: wipe only STATE and EPHEMERAL (preserves bootloader):
talosctl reset -n <NODE_IP> \
  --system-labels-to-wipe STATE \
  --system-labels-to-wipe EPHEMERAL

# Wipe etcd data dir only (etcd recovery tool):
talosctl reset -n <NODE_IP> \
  --graceful=false \
  --reboot \
  --system-labels-to-wipe EPHEMERAL
```

> ⚠️ A full `talosctl reset` on a cloud VM that boots from disk (not PXE/ISO)
> will wipe the bootloader and leave the VM unbootable. Use
> `--system-labels-to-wipe STATE --system-labels-to-wipe EPHEMERAL` instead.

---

## Security & Credential Handling

- **talosconfig** contains client private key — treat as a credential, store in secrets vault
- **secrets.yaml** is the cluster root of trust — never commit to Git, never share
- **kubeconfig**: regenerate on demand with `talosctl kubeconfig` rather than storing long-lived copies
- **Certificate TTL**: default Talos client cert TTL is 1 year (`talosctl config new --crt-ttl 8760h`)
- **CA rotation**: use `talosctl rotate-ca` with `--dry-run` first — always review before applying
- **Minimum access**: create scoped client configs with `talosctl config new --roles os:reader` for read-only access

```bash
# Verify current context is correct before any mutating op:
talosctl config contexts
```

---

## Environment Variations

| Aspect | Single-node / lab | Multi-node production |
|---|---|---|
| etcd quorum | Single member — no quorum concern | 3+ CP nodes — never reboot >1 simultaneously |
| Upgrade safety | Can upgrade all at once | One node at a time, health check between each |
| etcd backup | Optional but recommended | Mandatory before every upgrade or reset |
| `talosctl reset --graceful` | `--graceful=false` acceptable | Always use graceful unless node is unreachable |
| Bootstrap | Single `bootstrap` call on the one node | On exactly one of the CP nodes |

---

## Anti-patterns — Never Do These

- [ ] Do not run `talosctl bootstrap` more than once — it corrupts etcd
- [ ] Do not upgrade more than one control plane node at a time without checking health between each
- [ ] Do not skip `--dry-run` before patching configs or applying changes
- [ ] Do not use `--force` on `talosctl upgrade` across multiple CP nodes simultaneously
- [ ] Do not store `secrets.yaml` or full generated configs in Git
- [ ] Do not re-apply a full stale `controlplane.yaml` after `upgrade-k8s` — it will downgrade component versions
- [ ] Do not use `--insecure` on a running configured node — maintenance port is inactive after first config apply
- [ ] Do not skip `--system-labels-to-wipe STATE,EPHEMERAL` when resetting a cloud VM that boots from disk
- [ ] Do not skip etcd backup before any reset or recovery operation
- [ ] Do not skip adjacent-minor-version upgrade path — no skipping minor releases

---

## References

- [Talos documentation](https://docs.siderolabs.com/talos/)
- [CLI reference](https://docs.siderolabs.com/talos/v1.10/reference/cli)
- [Machine config reference](https://docs.siderolabs.com/talos/v1.10/reference/configuration/v1alpha1/config)
- [Config patching guide](https://docs.siderolabs.com/talos/v1.10/configure-your-talos-cluster/system-configuration/patching)
- [Apply modes](https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/system-configuration/editing-machine-configuration.md)
- [Talos OS upgrade guide](https://docs.siderolabs.com/talos/v1.10/configure-your-talos-cluster/lifecycle-management/upgrading-talos)
- [Kubernetes upgrade guide](https://docs.siderolabs.com/kubernetes-guides/advanced-guides/upgrading-kubernetes.md)
- [etcd backup and disaster recovery](https://docs.siderolabs.com/talos/v1.13/build-and-extend-talos/cluster-operations-and-maintenance/disaster-recovery.md)
- [etcd maintenance](https://docs.siderolabs.com/talos/v1.13/build-and-extend-talos/cluster-operations-and-maintenance/etcd-maintenance.md)
- [Support matrix (Talos × Kubernetes versions)](https://docs.siderolabs.com/talos/v1.10/getting-started/support-matrix)
- [Image Factory](https://factory.talos.dev/)
- [GitHub: siderolabs/talos](https://github.com/siderolabs/talos)
