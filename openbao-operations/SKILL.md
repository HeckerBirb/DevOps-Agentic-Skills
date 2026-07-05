---
name: openbao-operations
description: >
  Deploy and operate OpenBao (open-source Vault fork) on Kubernetes: Helm
  installation, initialization, all unseal methods, secrets engine and auth
  method configuration, policy authoring, Raft snapshots, and upgrades.
  Use when: installing OpenBao on Kubernetes, initializing and unsealing
  OpenBao, configuring Kubernetes auth or AppRole auth, enabling KV/PKI/
  transit secrets engines, writing or updating policies, rotating secrets
  or certificates, backing up or restoring Raft storage, upgrading OpenBao
  via Helm, troubleshooting a sealed or unhealthy OpenBao pod, or setting
  up audit logging.
  Operating model: Declarative/IaC (Helm values + HCL config in Git) +
  imperative CLI ops (bao CLI for config, secrets, and diagnostics).
  CLI binary is "bao" not "vault". Environment variables use BAO_ prefix.
  Audience: Platform engineers, security engineers.
  Scope: Day-0 (bootstrap) through Day-2 (ongoing operations).
allowed-tools: shell
---

# OpenBao Operations

> **Profile**: Declarative/IaC (Helm + HCL config) + CLI Ops (bao CLI)
> **Source of truth**: Git repository containing Helm values and HCL policy files
> **Auth model**: Token-based (scoped service tokens post-init); Kubernetes auth for pods
> **Blast radius**: Cluster-wide (secrets access) — treat with extreme care
> **CLI binary**: `bao` (not `vault`). Env vars: `BAO_ADDR`, `BAO_TOKEN` (not VAULT_*)

OpenBao is an open-source, community-maintained fork of HashiCorp Vault under
the Linux Foundation. It runs as a StatefulSet on Kubernetes and stores all
data in Raft integrated storage. Configuration (Helm values, HCL config,
policy files) lives in Git. Day-to-day operations use the `bao` CLI via
`kubectl exec` or a local binary pointed at the cluster.

> ⚠️ **Unseal keys and the root token are irreplaceable**. If all unseal key
> shares are lost and no KMS is configured, the data vault is permanently
> inaccessible. Treat initialization output as the most sensitive credential
> in your environment.

---

## Prerequisites

### Required tools

| Tool | Verify |
|---|---|
| `kubectl` | `kubectl version --client` |
| `helm` ≥ 3.6 | `helm version` |
| `bao` CLI (optional, for local ops) | `bao version` |

Install the `bao` CLI:
```bash
# Linux — check https://openbao.org/docs/install/ for latest version
curl -Lo /tmp/bao.zip \
  https://github.com/openbao/openbao/releases/download/v<VERSION>/bao_<VERSION>_linux_amd64.zip
unzip /tmp/bao.zip -d /usr/local/bin/
chmod +x /usr/local/bin/bao
bao version
```

### Environment variables

```bash
export BAO_ADDR='https://<OPENBAO_SERVICE_OR_LB>:8200'
export BAO_TOKEN='<token>'          # set after login
export BAO_SKIP_VERIFY='false'      # set true only for testing with self-signed cert
# Note: seal subsystem still uses VAULT_SEAL_TYPE, VAULT_AWSKMS_SEAL_KEY_ID etc.
# for KMS auto-unseal — this is expected, not a bug.
```

### Context guard

```bash
# Verify you are targeting the correct cluster:
kubectl config current-context
kubectl get pods -n <OPENBAO_NAMESPACE> -l app.kubernetes.io/name=openbao

# Check current seal status:
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- bao status
```

---

## Day 0 — Install OpenBao with Helm

### Step 1 — Add the Helm repository

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
helm search repo openbao/openbao   # verify chart version and appVersion
```

### Step 2 — Prepare Helm values

Choose a deploy mode. **HA + Raft is recommended for production.**

#### Standalone (single pod, file backend — dev/test only)

```yaml
# values-standalone.yaml
server:
  standalone:
    enabled: true
global:
  tlsDisable: false   # MUST be false for production
```

#### HA + Raft (3 replicas — production)

```yaml
# values-ha.yaml
global:
  tlsDisable: false             # MUST be false for production

server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      setNodeId: true           # pod name becomes Raft node ID — REQUIRED to prevent split-brain on reschedule
      config: |
        ui = true

        listener "tcp" {
          tls_disable     = false
          address         = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file   = "/openbao/userconfig/tls/tls.crt"
          tls_key_file    = "/openbao/userconfig/tls/tls.key"
          tls_min_version = "tls12"
        }

        storage "raft" {
          path = "/openbao/data"

          retry_join {
            leader_api_addr = "https://openbao-0.openbao-internal:8200"
            leader_ca_cert_file = "/openbao/userconfig/tls/ca.crt"
          }
          retry_join {
            leader_api_addr = "https://openbao-1.openbao-internal:8200"
            leader_ca_cert_file = "/openbao/userconfig/tls/ca.crt"
          }
          retry_join {
            leader_api_addr = "https://openbao-2.openbao-internal:8200"
            leader_ca_cert_file = "/openbao/userconfig/tls/ca.crt"
          }

          performance_multiplier = 1   # set to 1 for production performance
        }

        service_registration "kubernetes" {}

  # Mount TLS secret (create before installing):
  volumes:
    - name: userconfig-tls
      secret:
        secretName: openbao-tls
  volumeMounts:
    - mountPath: /openbao/userconfig/tls
      name: userconfig-tls
      readOnly: true

  # Audit storage (recommended for production):
  auditStorage:
    enabled: true
    size: 10Gi
    mountPath: /openbao/audit

  # Required for Kubernetes auth method:
  authDelegator:
    enabled: true   # creates system:auth-delegator ClusterRoleBinding for OpenBao SA
```

> ⚠️ `global.tlsDisable` defaults to `true` in the chart. Always set it
> to `false` for production. Leaving TLS disabled exposes all tokens and
> secrets in transit.

> ⚠️ `setNodeId: true` is required for HA Raft. Without it, a pod that
> reschedules gets a new Raft node ID, which can cause split-brain.

### Step 3 — Create TLS secret before installing

```bash
# Provide your own TLS certificate (cert-manager, your PKI, etc.):
kubectl create namespace <OPENBAO_NAMESPACE>
kubectl create secret generic openbao-tls \
  --namespace <OPENBAO_NAMESPACE> \
  --from-file=tls.crt=<path/to/tls.crt> \
  --from-file=tls.key=<path/to/tls.key> \
  --from-file=ca.crt=<path/to/ca.crt>
```

### Step 4 — Install

```bash
helm install openbao openbao/openbao \
  --namespace <OPENBAO_NAMESPACE> \
  --create-namespace \
  --version <CHART_VERSION> \
  -f values-ha.yaml
```

**Verify pods are running (but sealed — this is expected):**

```bash
kubectl get pods -n <OPENBAO_NAMESPACE>
# openbao-0   0/1  Running  0  ...  ← Not Ready (sealed) is expected
# openbao-1   0/1  Running  0  ...
# openbao-2   0/1  Running  0  ...
```

---

## Day 0 — Initialize OpenBao

> Run `bao operator init` exactly once, on one pod.
> In HA mode, the other pods learn initialization state via Raft.

### Step 1 — Initialize

#### Option A: Manual unseal (Shamir key splitting)

```bash
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- \
  bao operator init \
  -key-shares=5 \
  -key-threshold=3
```

Output:
```
Unseal Key 1: <key-1>
Unseal Key 2: <key-2>
Unseal Key 3: <key-3>
Unseal Key 4: <key-4>
Unseal Key 5: <key-5>

Initial Root Token: s.<token>
```

> ⚠️ **Save all unseal keys and the root token to secure, offline storage
> immediately.** This is the only time they are shown. Loss of enough keys
> (below threshold) means permanent data loss.

Best practice: use PGP to encrypt each key to a different operator's key:
```bash
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- \
  bao operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys="keybase:alice,keybase:bob,keybase:carol,keybase:dave,keybase:eve" \
  -root-token-pgp-key="keybase:secureop"
```

#### Option B: Auto-unseal via cloud KMS

Add the seal stanza to your Helm values config before installing:

**AWS KMS:**
```hcl
seal "awskms" {
  region     = "<AWS_REGION>"
  kms_key_id = "<KMS_KEY_ARN_OR_ID>"
  # Credentials via IRSA (preferred) or env vars
}
```

**GCP CKMS:**
```hcl
seal "gcpckms" {
  project    = "<GCP_PROJECT>"
  region     = "global"
  key_ring   = "<KEYRING_NAME>"
  crypto_key = "<KEY_NAME>"
  # Credentials via Workload Identity (preferred) or GOOGLE_APPLICATION_CREDENTIALS
}
```

**Azure Key Vault:**
```hcl
seal "azurekeyvault" {
  tenant_id  = "<TENANT_ID>"
  client_id  = "<CLIENT_ID>"
  vault_name = "<KEYVAULT_NAME>"
  key_name   = "<KEY_NAME>"
  # Credentials via Managed Service Identity (preferred) or client_secret
}
```

With KMS configured, initialize with recovery keys (not unseal keys):
```bash
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- \
  bao operator init \
  -recovery-shares=5 \
  -recovery-threshold=3
```

> ⚠️ If the KMS key is deleted or disabled, OpenBao **cannot unseal** even
> from snapshots. Protect the KMS key with IAM policies that prevent deletion.
> Treat the KMS key as the most critical resource in the environment.

### Step 2 — Unseal all pods

#### Manual unseal (Shamir — run for each pod separately)

```bash
# Each pod needs key-threshold key shares entered separately:
for POD in openbao-0 openbao-1 openbao-2; do
  echo "Unsealing $POD (enter ${THRESHOLD} key shares when prompted)..."
  kubectl exec -n <OPENBAO_NAMESPACE> -ti $POD -- bao operator unseal   # key 1
  kubectl exec -n <OPENBAO_NAMESPACE> -ti $POD -- bao operator unseal   # key 2
  kubectl exec -n <OPENBAO_NAMESPACE> -ti $POD -- bao operator unseal   # key 3
done
```

#### Auto-unseal (KMS)

With KMS configured, pods unseal automatically on start. Verify:
```bash
for POD in openbao-0 openbao-1 openbao-2; do
  echo "=== $POD ===" && kubectl exec -n <OPENBAO_NAMESPACE> $POD -- bao status
done
```

### Step 3 — Verify initialization

```bash
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- bao status
```

Expected output:
```
Seal Type          shamir (or awskms/gcpckms/azurekeyvault)
Initialized        true
Sealed             false
HA Enabled         true
HA Mode            active      ← one pod shows active, others show standby
Raft Committed Index   <N>
Raft Applied Index     <N>
```

---

## Day 0 — Initial Configuration (as root token — then revoke it)

```bash
export BAO_ADDR='https://<OPENBAO_ADDR>:8200'
export BAO_TOKEN='s.<ROOT_TOKEN>'

# Step 1: Enable audit logging FIRST (before any secrets are written)
bao audit enable file file_path=stdout       # container stdout → pod logs
bao audit enable -path=audit-file file \
  file_path=/openbao/audit/audit.log         # persistent audit (if PVC mounted)
bao audit list
```

> ⚠️ Enable audit logging before writing any secrets. There is no retroactive
> audit trail. If ALL audit devices fail, OpenBao stops serving requests —
> configure 2+ devices.

```bash
# Step 2: Enable auth methods
bao auth enable kubernetes
bao auth enable approle   # optional

# Step 3: Configure Kubernetes auth (running inside the cluster)
bao write auth/kubernetes/config \
  kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
# token_reviewer_jwt and kubernetes_ca_cert auto-loaded from pod service account
# when running inside the cluster

# Step 4: Enable secrets engines
bao secrets enable -version=2 kv        # KV v2 at kv/
bao secrets enable pki
bao secrets tune -max-lease-ttl=87600h pki   # 10-year max for root CA
bao secrets enable transit

# Step 5: Write a non-root admin policy and token
cat <<EOF | bao policy write admin-policy -
path "sys/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
path "auth/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
bao token create -policy=admin-policy -ttl=8h

# Step 6: Switch to the scoped admin token
export BAO_TOKEN='<new-admin-token>'

# Step 7: REVOKE the root token immediately
bao token revoke s.<ROOT_TOKEN>

# Verify revocation (must fail with permission denied):
BAO_TOKEN='s.<ROOT_TOKEN>' bao token lookup \
  && echo "ERROR: root token still valid" \
  || echo "OK: root token revoked"
```

> Root token can be regenerated if needed via `bao operator generate-root`
> using a quorum of unseal key holders. This is the only legitimate path
> to regain root access.

---

## Day 1 — Kubernetes Auth Method Configuration

```bash
# Create a role per service (one role per service account — do not share roles):
bao write auth/kubernetes/role/<SERVICE_NAME> \
  bound_service_account_names=<SERVICE_ACCOUNT_NAME> \
  bound_service_account_namespaces=<NAMESPACE> \
  policies=<POLICY_NAME> \
  ttl=1h

# Test login from a pod (or verify manually):
bao write auth/kubernetes/login \
  role=<SERVICE_NAME> \
  jwt=$(kubectl exec -n <NAMESPACE> <POD> -- \
    cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

**Verify the resulting token has the right capabilities:**
```bash
bao token capabilities kv/data/<path>
```

---

## Day 1 — Writing Policies

Policies use HCL, are path-based, and deny by default. Store policy files
in Git and apply with `bao policy write`.

```hcl
# policies/myapp.hcl — store in Git
path "kv/data/myapp/*" {
  capabilities = ["create", "read", "update", "patch", "delete", "list"]
}

path "kv/metadata/myapp/*" {
  capabilities = ["list", "read"]
}

# KV v2 path note:
# Write/read operations hit kv/data/<path>
# Metadata lives at kv/metadata/<path>
# Delete specific version: kv/delete/<path>
```

```bash
bao policy write myapp policies/myapp.hcl
bao policy read myapp     # verify content
bao policy list           # list all policies
```

---

## Day 2 — Reading, Writing, and Managing Secrets

### KV v2 — core operations

```bash
# Write a secret:
bao kv put -mount=kv myapp/db-creds username=admin password=changeme

# Read the latest version:
bao kv get -mount=kv myapp/db-creds

# Read a specific version:
bao kv get -mount=kv -version=2 myapp/db-creds

# Patch (update specific keys without replacing all):
bao kv patch -mount=kv myapp/db-creds password=newpassword

# List secrets at a path:
bao kv list -mount=kv myapp/

# Soft-delete the latest version (recoverable):
bao kv delete -mount=kv myapp/db-creds

# Restore a deleted version:
bao kv undelete -mount=kv -versions=2 myapp/db-creds

# Permanently destroy a specific version:
bao kv destroy -mount=kv -versions=1 myapp/db-creds

# View version history and metadata:
bao kv metadata get -mount=kv myapp/db-creds
```

### Lease management (Day 2 hygiene)

Dynamic secrets (database creds, PKI certs) create leases. Expired or orphaned
leases consume memory; renewing them keeps credentials alive.

```bash
# List active leases under a path:
bao list sys/leases/lookup/kv/data/myapp/

# Renew a dynamic secret lease:
bao lease renew <lease-id>

# Revoke a specific lease (rotates the credential):
bao lease revoke <lease-id>

# Revoke all leases under a prefix (e.g., rotate all DB creds for a service):
bao lease revoke -prefix kv/data/myapp/db-creds

# Check lease TTL:
bao write sys/leases/lookup lease_id=<lease-id>
```

### Token management

```bash
# Inspect the current token:
bao token lookup

# Create a scoped token for an automation task:
bao token create -policy=myapp-policy -ttl=1h -use-limit=10

# Renew current token:
bao token renew
bao token renew -increment=2h

# Revoke a specific token:
bao token revoke <token>

# Revoke all tokens with a given accessor (without exposing the token):
bao token revoke -accessor <accessor>
```

---

## Troubleshooting Common Failures

### Pod sealed after restart (manual Shamir)

```bash
# Check which pods are sealed:
kubectl get pods -n <OPENBAO_NAMESPACE>
# Not-Ready pods are sealed

# Unseal each sealed pod (requires key-threshold key shares):
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-1 -- bao status
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-1 -- bao operator unseal

# Verify:
kubectl exec -n <OPENBAO_NAMESPACE> openbao-1 -- bao status
# Sealed: false
```

### Auto-unseal (KMS) pod not unsealing on restart

```bash
# Check pod logs for KMS connectivity errors:
kubectl logs -n <OPENBAO_NAMESPACE> openbao-0

# Common causes:
# - KMS key not accessible (IAM/Workload Identity issue):
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- bao status
# Sealed: true, Seal Type: awskms → KMS reachable but key access denied

# - Network policy blocking outbound KMS traffic:
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  curl -s https://kms.<REGION>.amazonaws.com/   # for AWS

# - Wrong KMS key ID in config:
kubectl get secret -n <OPENBAO_NAMESPACE> -o yaml | grep -i kms
```

### No active leader (all pods in standby)

```bash
# All pods show HA Mode: standby — Raft election failed
kubectl logs -n <OPENBAO_NAMESPACE> openbao-0 | tail -50
# Look for: "failed to contact" between peers

# Verify Raft peer connectivity (requires a partially unsealed pod):
bao operator raft list-peers
# Check that all peer addresses resolve:
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  nslookup openbao-1.openbao-internal

# Common cause: pod was replaced and old Raft peer addr is stale
# Fix: remove stale peer from an unsealed active pod:
bao operator raft remove-peer openbao-1   # only if pod is permanently gone

# After resolving connectivity, leader election should happen automatically
```

### Audit device failure — OpenBao refusing requests

```bash
# All requests fail with: "1 error occurred: * failed to write audit log"
# This means ALL audit devices are unavailable

# Check audit devices:
bao audit list   # this will also fail if all devices are blocked

# Emergency: disable the failing audit device from inside the pod
# (requires a token that was cached before the failure)
bao audit disable file   # or the path of the failing device

# Add a working audit device immediately after:
bao audit enable file file_path=stdout

# Root cause: PVC full, or log file permissions issue
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  df -h /openbao/audit/
```

### Permission denied when accessing a secret path

```bash
# Check the token's policies:
bao token lookup
# Look at: policies field

# Check what capabilities the token has on the path:
bao token capabilities kv/data/myapp/db-creds

# Read the policy to see what it allows:
bao policy read <POLICY_NAME>

# Common KV v2 mistake: policy uses kv/myapp/* instead of kv/data/myapp/*
# KV v2 always requires the /data/ segment for read/write operations
```

---



### Backup (run regularly — automate with a CronJob)

```bash
# From inside the cluster:
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  bao operator raft snapshot save /tmp/raft.snap

# Copy out of the pod:
kubectl cp <OPENBAO_NAMESPACE>/openbao-0:/tmp/raft.snap \
  ./openbao-backup-$(date +%Y%m%d-%H%M%S).snap

# Verify the snapshot is not zero-size:
ls -lh openbao-backup-*.snap
```

> ⚠️ Snapshots contain all encrypted data. Store them with the same security
> as the OpenBao cluster itself. An attacker with a snapshot AND the unseal
> keys (or KMS access) has full access to all secrets.

### Restore from snapshot

> ⚠️ **Restore is destructive** — all data written after the snapshot was taken
> will be lost. The sequence depends on whether you are restoring in-place
> (overwriting a broken cluster) or bootstrapping a fresh cluster.

#### In-place restore (broken cluster, data intact enough to unseal)

```bash
# 1. Scale down all applications that write to OpenBao to quiesce writes:
kubectl scale deploy/<APP_DEPLOY> --replicas=0 -n <APP_NAMESPACE>

# 2. Verify all OpenBao peers are healthy enough to proceed:
bao operator raft list-peers
# Abort if quorum is lost — use fresh-cluster restore path instead

# 3. On the active pod, restore (this overwrites live Raft state):
kubectl cp ./openbao-backup.snap <OPENBAO_NAMESPACE>/openbao-0:/tmp/restore.snap
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  bao operator raft snapshot restore /tmp/restore.snap

# 4. Unseal all pods if using manual Shamir (KMS unseals automatically):
for POD in openbao-0 openbao-1 openbao-2; do
  kubectl exec -n <OPENBAO_NAMESPACE> -ti $POD -- bao operator unseal
done

# 5. Verify state is consistent:
bao operator raft list-peers
bao status
bao kv get -mount=<ENGINE> <TEST_PATH>    # spot-check a known secret
```

#### Fresh cluster restore (full data loss, rebuilding from snapshot)

```bash
# 1. Deploy a new OpenBao cluster (Helm install, not yet initialized)
# 2. Initialize with recovery restore (not bao operator init):
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- \
  bao operator raft snapshot restore /tmp/restore.snap
# This initializes AND restores state atomically

# 3. Unseal (KMS or manual Shamir with original unseal keys from the source cluster)
# 4. Verify with bao status + spot-check secrets
```

**Abort criteria:** Do not proceed with restore if:
- Raft quorum is healthy and data is accessible — investigate the issue instead
- You do not have the original unseal keys or KMS access — a restore cannot be decrypted without them
- Any writers are still active and writing — quiesce first to avoid inconsistency

---

## Day 2 — Upgrade OpenBao on Kubernetes

> **Critical**: OpenBao StatefulSet uses `OnDelete` update strategy.
> Pods are NOT automatically restarted by Helm upgrade. You must delete
> standby pods manually, in order: standbys first, active last.

### Step 1 — Snapshot before upgrading

```bash
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- \
  bao operator raft snapshot save /tmp/pre-upgrade.snap
kubectl cp <OPENBAO_NAMESPACE>/openbao-0:/tmp/pre-upgrade.snap ./pre-upgrade.snap
```

### Step 2 — Dry-run the Helm upgrade

```bash
helm upgrade openbao openbao/openbao \
  --namespace <OPENBAO_NAMESPACE> \
  --version <NEW_CHART_VERSION> \
  -f values-ha.yaml \
  --dry-run
```

### Step 3 — Apply the Helm upgrade (updates StatefulSet template, no pods restart yet)

```bash
helm upgrade openbao openbao/openbao \
  --namespace <OPENBAO_NAMESPACE> \
  --version <NEW_CHART_VERSION> \
  -f values-ha.yaml
```

### Step 4 — Upgrade standby pods first

```bash
# Identify which pod is active:
kubectl get pods -n <OPENBAO_NAMESPACE> -l openbao-active=true
kubectl get pods -n <OPENBAO_NAMESPACE> -l openbao-active=false

# Delete a standby pod (it restarts with the new image):
kubectl delete pod -n <OPENBAO_NAMESPACE> openbao-1

# If using manual unseal: unseal the restarted pod
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-1 -- bao operator unseal
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-1 -- bao status
# Sealed: false — confirm unsealed before proceeding

# Repeat for next standby:
kubectl delete pod -n <OPENBAO_NAMESPACE> openbao-2
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-2 -- bao operator unseal
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-2 -- bao status
```

### Step 5 — Upgrade the active pod last

```bash
# Confirm active pod (Raft will elect a new leader after this pod restarts):
kubectl get pods -n <OPENBAO_NAMESPACE> -l openbao-active=true

kubectl delete pod -n <OPENBAO_NAMESPACE> openbao-0
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- bao operator unseal   # if manual
kubectl exec -n <OPENBAO_NAMESPACE> -ti openbao-0 -- bao status
```

### Step 6 — Verify all pods healthy

```bash
bao operator raft list-peers
# All three nodes should appear as followers/leader, all voters
bao operator raft autopilot state
# Healthy: true, Failure Tolerance: 1
```

> ⚠️ **Downgrades are not supported** — the Raft storage may have been
> migrated forward. Always have a snapshot from before the upgrade.

---

## Observability

### Pod and seal status

```bash
# Check all pods:
kubectl get pods -n <OPENBAO_NAMESPACE>
# Pods NOT ready = sealed or not initialized (expected before unseal)

# Seal status per pod:
for POD in $(kubectl get pods -n <OPENBAO_NAMESPACE> -o name | grep openbao); do
  echo "=== $POD ===" && kubectl exec -n <OPENBAO_NAMESPACE> $POD -- bao status 2>&1
done

# HA: which pod is active:
kubectl get pods -n <OPENBAO_NAMESPACE> -l openbao-active=true
```

### Raft cluster health

```bash
# List peers (requires authenticated bao CLI):
bao operator raft list-peers
# Expected: 3 nodes — one leader, two followers, all voters

# Autopilot state (shows health and failure tolerance):
bao operator raft autopilot state
```

### Health signals

| Signal | Meaning |
|---|---|
| `Sealed: false`, `HA Mode: active/standby` | Healthy |
| `Sealed: true` | Pod needs unsealing — check if KMS is reachable or manually unseal |
| `Initialized: false` | Not yet initialized — run `bao operator init` |
| Pod `0/1 Running` (readiness failing) | Sealed or uninitialized |
| `HA Mode: standby` on all pods | No active node — leadership election issue |
| Audit log stops writing | All audit devices failed — OpenBao stops serving requests |

### Audit logs

```bash
# Pod logs (if audit device is stdout):
kubectl logs -n <OPENBAO_NAMESPACE> openbao-0 -f | grep '"type":"request"'

# Persistent audit file (if auditStorage PVC mounted):
kubectl exec -n <OPENBAO_NAMESPACE> openbao-0 -- tail -f /openbao/audit/audit.log

# List active audit devices:
bao audit list
```

Audit log entries are JSON, one per line. Sensitive values are HMAC-SHA256
hashed. Use `bao write sys/audit-hash/<device-path> input=<value>` to check
if a specific value matches a hash in the logs.

### Kubernetes readiness probe behavior

The chart's default readiness probe calls `/v1/sys/health`:
- `200` — initialized, unsealed, active → **Ready**
- `429` — unsealed standby → **Not Ready** (expected for standbys with default probe)
- `503` — sealed → **Not Ready**
- `501` — not initialized → **Not Ready**

To make standby pods ready (e.g., for read-heavy workloads):
```yaml
server:
  readinessProbe:
    path: '/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204'
```

---

## Security Hardening Checklist

Run after initial deployment to verify production posture:

```bash
# 1. Root token is revoked:
bao token lookup s.<ROOT_TOKEN> 2>&1 | grep -q "permission denied" \
  && echo "OK: root token revoked" \
  || echo "FAIL: root token still active"

# 2. Audit logging is enabled:
bao audit list | grep -v "^$" \
  && echo "OK: audit enabled" \
  || echo "FAIL: no audit devices"

# 3. TLS is enabled (check from outside the pod):
curl -sk https://<OPENBAO_ADDR>:8200/v1/sys/health | python3 -m json.tool

# 4. Raft is healthy:
bao operator raft list-peers

# 5. Policies follow least privilege (no wildcards on sensitive paths):
bao policy list
```

- **TLS**: `global.tlsDisable: false` in Helm values — non-negotiable for production
- **Root token**: revoke immediately after init; regenerate only via `bao operator generate-root`
- **One role per service**: never share Kubernetes auth roles across multiple service accounts
- **Unseal keys**: split across ≥5 holders; PGP-encrypt; store offline/out-of-band
- **KMS key protection**: use IAM policies, SCPs, or Key Vault access policies to prevent deletion
- **Audit redundancy**: configure 2+ audit devices; test that logs are flowing

---

## Environment Variations

| Aspect | Dev / lab | Production |
|---|---|---|
| Deploy mode | standalone (single pod) | HA Raft (3+ pods) |
| `global.tlsDisable` | `true` acceptable | `false` — always |
| Unseal method | Manual Shamir | KMS auto-unseal or well-distributed Shamir |
| Root token | Can leave active briefly | Revoke immediately after init |
| Audit logging | Optional | Mandatory on Day 0 |
| `setNodeId` | Not critical | `true` — required to prevent split-brain |
| Raft snapshots | Manual / ad-hoc | Automated CronJob; tested restores |
| `performance_multiplier` | Default (5) | `1` for production performance |

---

## Anti-patterns — Never Do These

- [ ] Do not lose all unseal key shares or disable the KMS key — this is unrecoverable
- [ ] Do not operate day-to-day with the root token — revoke it after init
- [ ] Do not skip enabling audit logging before writing any secrets
- [ ] Do not set `global.tlsDisable: true` in production
- [ ] Do not share Kubernetes auth roles across multiple services — one role per service account
- [ ] Do not store unseal keys in Kubernetes Secrets or Git — they protect the vault itself
- [ ] Do not store secret values in policy files — policies reference paths, not values
- [ ] Do not set `setNodeId: false` in HA Raft — pod reschedules will cause split-brain
- [ ] Do not upgrade the active pod before all standbys — upgrade standbys first
- [ ] Do not skip a Raft snapshot before upgrades — downgrades are unsupported
- [ ] Do not delete or disable the KMS key used for auto-unseal — even backups become inaccessible
- [ ] Do not configure only one audit device — if it fails, OpenBao stops serving all requests

---

## References

- [OpenBao documentation](https://openbao.org/docs/)
- [Helm chart overview](https://openbao.org/docs/platform/k8s/helm/)
- [Helm chart — run / operations](https://openbao.org/docs/platform/k8s/helm/run/)
- [Helm values reference](https://openbao.org/docs/platform/k8s/helm/configuration/)
- [Helm chart GitHub](https://github.com/openbao/openbao-helm)
- [Seal concepts](https://openbao.org/docs/concepts/seal/)
- [AWS KMS seal](https://openbao.org/docs/configuration/seal/awskms/)
- [GCP CKMS seal](https://openbao.org/docs/configuration/seal/gcpckms/)
- [Azure Key Vault seal](https://openbao.org/docs/configuration/seal/azurekeyvault/)
- [Raft storage configuration](https://openbao.org/docs/configuration/storage/raft/)
- [TCP listener / TLS configuration](https://openbao.org/docs/configuration/listener/tcp/)
- [Kubernetes auth method](https://openbao.org/docs/auth/kubernetes/)
- [AppRole auth method](https://openbao.org/docs/auth/approle/)
- [KV v2 secrets engine](https://openbao.org/docs/secrets/kv/kv-v2/)
- [PKI secrets engine](https://openbao.org/docs/secrets/pki/)
- [Transit secrets engine](https://openbao.org/docs/secrets/transit/)
- [Policies](https://openbao.org/docs/concepts/policies/)
- [Tokens](https://openbao.org/docs/concepts/tokens/)
- [Audit devices](https://openbao.org/docs/audit/)
- [File audit device](https://openbao.org/docs/audit/file/)
- [`bao operator init`](https://openbao.org/docs/commands/operator/init/)
- [`bao operator raft`](https://openbao.org/docs/commands/operator/raft/)
- [`bao operator generate-root`](https://openbao.org/docs/commands/operator/generate-root/)
- [Upgrading OpenBao](https://openbao.org/docs/upgrading/)
- [GitHub: openbao/openbao](https://github.com/openbao/openbao)
- [GitHub: openbao/openbao-helm](https://github.com/openbao/openbao-helm)
