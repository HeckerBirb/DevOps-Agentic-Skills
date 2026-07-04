---
name: ansible-k8s-role-development
description: >
  Develop, author, modify, and debug Ansible roles that manage Kubernetes
  resources using the kubernetes.core collection. Use when: writing a new Ansible
  role for Kubernetes, adding tasks to an existing role, deploying a workload to
  Kubernetes via Ansible, debugging a failing Ansible playbook against Kubernetes,
  reviewing an Ansible role for best practices, authoring production-grade platform
  roles (Kafka, Vault/OpenBao, cert-manager, Rook/Ceph, Cilium, monitoring stacks),
  generating k8s module tasks, writing idempotent Kubernetes resource management,
  implementing HA workload deployment with Ansible, handling Kubernetes secrets in
  Ansible Vault, or troubleshooting Kubernetes resource states non-destructively.
  Target environment: on-prem Talos Linux cluster with Cilium CNI and Rook/Ceph CSI.
  Audience: Platform engineers and Operations team members.
allowed-tools: shell
---

# Ansible Role Development for Kubernetes

> **Profile**: Declarative/IaC (primary) + CLI Ops/Diagnostics (secondary)
> **Source of truth**: Git repository containing Ansible roles
> **Auth model**: `~/.kube/config` (cluster-admin) via `delegate_to: localhost` + `run_once: true`
> **Secrets**: Ansible Vault (interactive password prompt or pipeline)
> **Stack**: Talos Linux · Cilium CNI · Rook/Ceph CSI · OpenBao (Vault fork)

This skill guides the authoring, modification, and debugging of Ansible roles that
manage Kubernetes resources declaratively using the `kubernetes.core` collection.
All operations execute on localhost and delegate to the cluster via the Kubernetes
API — no SSH to nodes required and no direct `kubectl apply` outside of Ansible.

---

## Step 0 — Session Setup (ALWAYS run first)

### 0.1 Identify the codebase

At the start of every session, look for an existing Ansible codebase:

```bash
# Look for Ansible project root indicators
ls -la | grep -E "ansible.cfg|site.yml|playbooks|roles|inventory|Justfile"
find . -maxdepth 3 -name "ansible.cfg" -o -name "site.yml" 2>/dev/null | head -10
```

If no codebase is found, ask the user:
> "I couldn't find an Ansible project in the current directory. Please provide the
> path to your Ansible project, or confirm if you want to start a new one."

### 0.2 Analyse the Justfile

If a `Justfile` exists, read it in full before taking any action:

```bash
cat Justfile
```

- Identify recipes for deploying roles, running playbooks, launching debug pods,
  or interacting with the cluster.
- **Use Justfile recipes in preference to raw commands** where they exist and are
  appropriate (e.g., `just deploy <role>` instead of constructing `ansible-playbook`
  commands manually).
- Reference specific recipe names in your explanations so the user knows what to run.

### 0.3 Understand the role being worked on

```bash
# List existing roles
ls roles/ 2>/dev/null || find . -maxdepth 4 -type d -name "tasks" | sed 's|/tasks||'

# Read the target role's structure
find roles/<role-name> -type f | sort

# Read key files
cat roles/<role-name>/tasks/main.yml
cat roles/<role-name>/defaults/main.yml
cat roles/<role-name>/vars/main.yml 2>/dev/null
```

### 0.4 Verify prerequisites

```bash
# Verify kubernetes.core collection is installed
ansible-galaxy collection list | grep kubernetes.core

# Verify Python dependencies
python3 -c "import kubernetes; print(kubernetes.__version__)"
python3 -c "import yaml; print(yaml.__version__)"

# Verify kubeconfig context — ALWAYS confirm before any operation
kubectl config current-context
kubectl config get-contexts
```

> ⚠️ **Confirm the correct kubeconfig context before every session.** Never assume.
> If multiple contexts exist, ask the user which cluster they are targeting.

---

## Step 1 — Ansible Role Structure

Every Ansible role for Kubernetes MUST follow this structure:

```
roles/<role-name>/
├── tasks/
│   ├── main.yml          # Entry point — imports sub-task files
│   ├── namespace.yml     # Namespace and RBAC setup
│   ├── secrets.yml       # Secret and credential creation
│   ├── deploy.yml        # Core resource deployment
│   └── verify.yml        # Post-deploy verification (read-only)
├── templates/            # Jinja2 templates for K8s manifests (.j2)
├── files/                # Static K8s manifests (no templating needed)
├── defaults/
│   └── main.yml          # All role defaults — override in inventory/group_vars
├── vars/
│   └── main.yml          # High-precedence role vars (not secrets)
├── handlers/
│   └── main.yml          # Handlers (e.g., triggered after config changes)
└── meta/
    └── main.yml          # Dependencies on other roles
```

### Variable and secret file layout

Follow the **vars + vault pattern** so variable names remain grep-visible:

```
inventory/
└── group_vars/
    └── all/
        ├── vars.yml      # All variable names (points to vault_ equivalents for secrets)
        └── vault.yml     # Encrypted with ansible-vault — contains only vault_* values
```

Example `vars.yml`:
```yaml
# Point to vault-encrypted values — never store secret values here
kafka_admin_password: "{{ vault_kafka_admin_password }}"
openbao_root_token: "{{ vault_openbao_root_token }}"
```

Example `vault.yml` (encrypted, never committed in plaintext):
```yaml
vault_kafka_admin_password: "ANSIBLE_VAULT_ENCRYPTED"
vault_openbao_root_token: "ANSIBLE_VAULT_ENCRYPTED"
```

> ⚠️ **Auto-generated credentials MUST be captured.** See Step 5 — Credential Capture.

---

## Step 2 — Task Authoring Rules

These rules apply to EVERY task in every role. No exceptions.

### Mandatory task patterns

```yaml
# Every task MUST have:
# 1. A descriptive name in sentence case
# 2. An explicit FQCN (fully qualified collection name)
# 3. Explicit state: (never rely on defaults)
# 4. run_once: true (all kubernetes.core module tasks)
# 5. delegate_to: localhost (all kubernetes.core module tasks)

- name: Create Kafka namespace
  kubernetes.core.k8s:
    state: present
    run_once: true
    delegate_to: localhost
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ kafka_namespace }}"
        labels:
          app.kubernetes.io/managed-by: ansible
          environment: "{{ environment }}"
```

### Pre-check before every mutating task

Always query the current state before applying changes:

```yaml
- name: Check if Kafka namespace already exists
  kubernetes.core.k8s_info:
    kind: Namespace
    name: "{{ kafka_namespace }}"
  register: kafka_namespace_info
  run_once: true
  delegate_to: localhost

- name: Create Kafka namespace
  kubernetes.core.k8s:
    state: present
    run_once: true
    delegate_to: localhost
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: "{{ kafka_namespace }}"
  when: kafka_namespace_info.resources | length == 0
```

### Check mode (dry-run) MUST be supported

Always verify that the role runs cleanly in check mode before applying:

```yaml
# Tasks that cannot run in check mode must be explicitly skipped
- name: Execute Kafka topic creation script
  kubernetes.core.k8s_exec:
    namespace: "{{ kafka_namespace }}"
    pod: "{{ kafka_pod_name }}"
    command: /opt/kafka/bin/kafka-topics.sh --create ...
  run_once: true
  delegate_to: localhost
  when: not ansible_check_mode   # Skip in check mode — exec has side effects
```

### Wait for resources to reach desired state

```yaml
- name: Deploy Kafka StatefulSet
  kubernetes.core.k8s:
    state: present
    apply: true           # Use apply for idempotent server-side merge
    wait: true            # Wait for the resource to reach desired state
    wait_timeout: 300     # Seconds — increase for slow-starting workloads
    run_once: true
    delegate_to: localhost
    template: kafka-statefulset.j2
```

### CRD installation ordering

CRDs MUST be installed and confirmed ready before any Custom Resources that use them:

```yaml
- name: Install Strimzi CRDs
  kubernetes.core.k8s:
    state: present
    src: "{{ role_path }}/files/strimzi-crds.yaml"
    run_once: true
    delegate_to: localhost

- name: Wait for Kafka CRD to be established
  kubernetes.core.k8s_info:
    api_version: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    name: kafkas.kafka.strimzi.io
  register: kafka_crd_info
  until: >
    kafka_crd_info.resources | length > 0 and
    (kafka_crd_info.resources[0].status.conditions |
     selectattr('type', 'equalto', 'Established') |
     selectattr('status', 'equalto', 'True') | list | length > 0)
  retries: 30
  delay: 10
  run_once: true
  delegate_to: localhost

# Only proceed to Kafka CR after CRD is confirmed Established
- name: Deploy Kafka cluster
  kubernetes.core.k8s:
    state: present
    apply: true
    run_once: true
    delegate_to: localhost
    template: kafka-cluster.j2
```

### Merge strategy for Custom Resources

CRDs do not support strategic-merge patch. Always specify `merge_type` for CRs:

```yaml
- name: Update Kafka cluster configuration
  kubernetes.core.k8s:
    state: present
    apply: true
    merge_type:
      - merge           # Use merge (not strategic-merge) for CRDs
    run_once: true
    delegate_to: localhost
    template: kafka-cluster.j2
```

---

## Step 3 — Template Authoring

Use Jinja2 templates (`.j2` files in `templates/`) for all manifests that need
variable substitution. Use `files/` for static manifests.

### Template conventions

```jinja2
{# templates/kafka-cluster.j2 #}
{# Required variables: kafka_namespace, kafka_replicas, kafka_storage_size #}
{# kafka_replicas MUST be >= 3 for HA #}
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: {{ kafka_cluster_name }}
  namespace: {{ kafka_namespace }}
  labels:
    app.kubernetes.io/managed-by: ansible
    environment: {{ environment }}
spec:
  kafka:
    replicas: {{ kafka_replicas | int }}   {# Enforce int type #}
    storage:
      type: persistent-claim
      size: {{ kafka_storage_size }}
      storageClass: {{ kafka_storage_class }}   {# Must match Rook/Ceph StorageClass name #}
    config:
      # Encryption at rest: handled by Rook/Ceph encryption at the StorageClass level
      # Encryption in transit: TLS listener below
      inter.broker.protocol.version: "{{ kafka_protocol_version }}"
    listeners:
      - name: tls
        port: 9093
        type: internal
        tls: true
        authentication:
          type: tls
      - name: external
        port: 9094
        type: nodeport
        tls: true
    authorization:
      type: simple
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:   {# Hard anti-affinity for HA #}
              - labelSelector:
                  matchLabels:
                    strimzi.io/name: {{ kafka_cluster_name }}-kafka
                topologyKey: kubernetes.io/hostname
```

### Network policies (Cilium)

Always include a NetworkPolicy for every deployed workload. On Cilium, use
`CiliumNetworkPolicy` for L7 rules and standard `NetworkPolicy` for L3/L4:

```jinja2
{# templates/kafka-network-policy.j2 #}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ kafka_cluster_name }}-network-policy
  namespace: {{ kafka_namespace }}
spec:
  podSelector:
    matchLabels:
      strimzi.io/cluster: {{ kafka_cluster_name }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: "{{ allowed_consumer_namespace }}"
      ports:
        - protocol: TCP
          port: 9093   # TLS listener only — never expose 9092 (plaintext)
  egress:
    - to:
        - namespaceSelector: {}  # Allow DNS resolution
      ports:
        - protocol: UDP
          port: 53
```

---

## Step 4 — Preflight and Apply Workflow

### Standard deployment sequence

```
1. ansible-playbook --check   →  dry-run, find structural errors
2. Review check mode output   →  confirm only expected changes
3. ansible-playbook           →  apply (with wait)
4. Verify idempotency         →  re-run, confirm 0 changes
5. Post-deploy health check   →  non-destructive k8s_info / k8s_log queries
```

### Run check mode first — always

```bash
# Via Justfile (preferred — check if recipe exists first)
just check <role-or-playbook>

# Or directly:
ansible-playbook site.yml --tags <role-name> --check --diff
```

Expected output indicators:
- `ok`: Resource already in desired state — no change needed ✅
- `changed` (check mode): Resource would be created or updated — review the diff
- `skipped`: Task condition not met — verify this is expected
- `failed`: Error that would fail the real run — fix before applying

Review every `changed` item in the diff output. If any change is unexpected, stop and investigate.

### Apply

```bash
# Via Justfile (preferred)
just deploy <role-or-playbook>

# Or directly:
ansible-playbook site.yml --tags <role-name>
```

Expected: All tasks should complete with `ok` or `changed`. Zero `failed`.

### Verify idempotency — MUST re-run after every apply

```bash
ansible-playbook site.yml --tags <role-name> --check
```

Expected: ALL tasks report `ok`. Zero `changed`. If any task still shows `changed`
after a successful apply, the role is **not idempotent** — this is a bug. Investigate
and fix before marking the role complete.

---

## Step 5 — Credential Capture Pattern

> **Critical**: Auto-generated admin credentials MUST be captured and stored.
> OpenBao/Vault, Kafka, and similar systems generate root tokens and admin passwords
> on first boot. These are shown once and never again. Losing them requires
> full reinstallation.

### Pattern: Capture and store auto-generated credentials

```yaml
- name: Retrieve OpenBao root token from init output
  kubernetes.core.k8s_exec:
    namespace: "{{ openbao_namespace }}"
    pod: "{{ openbao_pod }}"
    command: cat /tmp/init-output.json
  register: openbao_init_output
  run_once: true
  delegate_to: localhost
  when: not ansible_check_mode

- name: Parse and store root token in Ansible Vault variable file
  ansible.builtin.copy:
    content: |
      vault_openbao_root_token: "{{ (openbao_init_output.stdout | from_json).root_token }}"
      vault_openbao_unseal_keys: {{ (openbao_init_output.stdout | from_json).keys | to_yaml }}
    dest: "{{ inventory_dir }}/group_vars/all/vault.yml"
    mode: '0600'
  run_once: true
  delegate_to: localhost
  when:
    - not ansible_check_mode
    - openbao_init_output is defined
    - openbao_init_output.stdout | length > 0

- name: Encrypt vault file with ansible-vault
  ansible.builtin.command:
    cmd: >
      ansible-vault encrypt
      {{ inventory_dir }}/group_vars/all/vault.yml
  run_once: true
  delegate_to: localhost
  when:
    - not ansible_check_mode
    - openbao_init_output is defined
```

> ⚠️ After capturing credentials: confirm with the user that the vault file was
> encrypted and committed to the repo before proceeding.

### Password rotation pattern

When admin passwords must be rotated (post-initial-deploy):

```yaml
- name: Update Kafka admin password
  kubernetes.core.k8s:
    state: present
    apply: true
    run_once: true
    delegate_to: localhost
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: kafka-admin-credentials
        namespace: "{{ kafka_namespace }}"
      type: Opaque
      stringData:
        password: "{{ kafka_admin_password }}"   # Sourced from vault_kafka_admin_password
  # After this: rotate the password in Ansible Vault, then re-encrypt and re-run

- name: Verify Kafka admin accepts new password
  kubernetes.core.k8s_exec:
    namespace: "{{ kafka_namespace }}"
    pod: "{{ kafka_pod }}"
    command: >
      /opt/kafka/bin/kafka-configs.sh
      --bootstrap-server localhost:9093
      --command-config /opt/kafka/config/admin.properties
      --describe --entity-type users
  register: kafka_auth_check
  run_once: true
  delegate_to: localhost
  when: not ansible_check_mode
  failed_when: kafka_auth_check.rc != 0
```

---

## Step 6 — Non-Destructive Diagnostics

Use these patterns for debugging. They are all read-only and safe at any time.

### Get resource status

```yaml
- name: Get Kafka cluster status
  kubernetes.core.k8s_info:
    api_version: kafka.strimzi.io/v1beta2
    kind: Kafka
    name: "{{ kafka_cluster_name }}"
    namespace: "{{ kafka_namespace }}"
  register: kafka_status
  run_once: true
  delegate_to: localhost

- name: Display Kafka cluster conditions
  ansible.builtin.debug:
    var: kafka_status.resources[0].status.conditions
  when: kafka_status.resources | length > 0
```

Or via Justfile / kubectl for interactive inspection:

```bash
kubectl get kafka -n <namespace> -o wide
kubectl describe kafka <name> -n <namespace>
kubectl get pods -n <namespace> -l strimzi.io/cluster=<name>
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -30
```

### Fetch pod logs

```yaml
- name: Fetch Kafka broker logs
  kubernetes.core.k8s_log:
    name: "{{ kafka_pod_name }}"
    namespace: "{{ kafka_namespace }}"
    tail_lines: 100
  register: kafka_logs
  run_once: true
  delegate_to: localhost

- name: Display logs
  ansible.builtin.debug:
    var: kafka_logs.log
```

### Execute diagnostic command in pod (non-destructive only)

```yaml
- name: Check Kafka topic list
  kubernetes.core.k8s_exec:
    namespace: "{{ kafka_namespace }}"
    pod: "{{ kafka_pod_name }}"
    command: >
      /opt/kafka/bin/kafka-topics.sh
      --bootstrap-server localhost:9093
      --list
  register: topic_list
  run_once: true
  delegate_to: localhost
  when: not ansible_check_mode

- name: Display topics
  ansible.builtin.debug:
    var: topic_list.stdout_lines
```

> ⚠️ Only use `k8s_exec` for **read-only diagnostic commands**. Never use it to
> modify application state — make those changes through Ansible tasks and K8s APIs.

---

## Step 7 — Rollback / Recovery Strategy

> **Reversibility**: Partially reversible
> Most K8s resource changes are reversible via `state: absent` + redeploy.
> The following operations are **NOT trivially reversible** and require special handling:
> - OpenBao seal state after drain/evict
> - Talos node configuration changes requiring reboot
> - PVC deletions (data loss)
> - Secret rotations (old credentials invalidated)

### Standard rollback — revert via Ansible

```yaml
# To roll back a deployment, set the previous image tag and re-run
- name: Roll back Kafka to previous version
  kubernetes.core.k8s:
    state: present
    apply: true
    wait: true
    wait_timeout: 300
    run_once: true
    delegate_to: localhost
    definition:
      apiVersion: kafka.strimzi.io/v1beta2
      kind: Kafka
      metadata:
        name: "{{ kafka_cluster_name }}"
        namespace: "{{ kafka_namespace }}"
      spec:
        kafka:
          version: "{{ kafka_previous_version }}"  # Set this in defaults/main.yml
```

Or use the built-in rollback for Deployments:

```yaml
- name: Roll back Deployment to previous revision
  kubernetes.core.k8s_rollback:
    api_version: apps/v1
    kind: Deployment
    name: "{{ deployment_name }}"
    namespace: "{{ deployment_namespace }}"
  run_once: true
  delegate_to: localhost
```

### OpenBao sealed after node eviction

> OpenBao (and HashiCorp Vault) will NOT auto-unseal after pod restart — this is by
> design for security. After any drain/evict/restart operation:

```bash
# Check seal status (non-destructive)
kubectl exec -n <openbao-namespace> <openbao-pod> -- bao status

# Manual unseal (requires unseal keys — must be stored in Ansible Vault)
# Run for each unseal key (minimum threshold required, typically 3 of 5)
kubectl exec -n <openbao-namespace> <openbao-pod> -- bao operator unseal <unseal-key>
```

Ansible task for unseal:

```yaml
- name: Check OpenBao seal status
  kubernetes.core.k8s_exec:
    namespace: "{{ openbao_namespace }}"
    pod: "{{ openbao_pod }}"
    command: bao status -format=json
  register: openbao_status
  run_once: true
  delegate_to: localhost
  failed_when: false   # bao status exits non-zero when sealed — do not fail here

- name: Unseal OpenBao (threshold of unseal keys required)
  kubernetes.core.k8s_exec:
    namespace: "{{ openbao_namespace }}"
    pod: "{{ openbao_pod }}"
    command: "bao operator unseal {{ item }}"
  loop: "{{ vault_openbao_unseal_keys[:openbao_unseal_threshold] }}"
  run_once: true
  delegate_to: localhost
  when:
    - not ansible_check_mode
    - (openbao_status.stdout | from_json).sealed | bool
  no_log: true   # Never log unseal keys
```

### Talos configuration changes

Changes to Talos node configuration (CNI config, kubelet settings, etc.) do not take
effect until the node is fully rebooted. Talos is immutable — there is no in-place
edit.

```bash
# Apply Talos config change (via talosctl — not Ansible)
talosctl apply-config --nodes <node-ip> --file <config.yaml>

# Reboot the node (controlled, one at a time — verify cluster health first)
# Check Rook/Ceph health before draining any node
kubectl get cephcluster -A
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status

# Only drain after confirming Ceph is HEALTH_OK
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Reboot (via talosctl)
talosctl reboot --nodes <node-ip>

# Wait for node to rejoin
kubectl wait node/<node-name> --for=condition=Ready --timeout=300s

# Uncordon
kubectl uncordon <node-name>
```

> ⚠️ Never drain a node when Rook/Ceph is in HEALTH_WARN or HEALTH_ERR state.
> Always drain one node at a time. Verify node is Ready and Ceph is healthy before
> draining the next.

---

## Step 8 — Stack-Specific Guidance

### Rook/Ceph (CSI)

```yaml
# Always verify Ceph health before storage-affecting operations
- name: Verify Rook/Ceph cluster health
  kubernetes.core.k8s_info:
    api_version: ceph.rook.io/v1
    kind: CephCluster
    namespace: rook-ceph
  register: ceph_cluster_info
  run_once: true
  delegate_to: localhost
  failed_when: >
    ceph_cluster_info.resources | length == 0 or
    ceph_cluster_info.resources[0].status.phase != 'Ready'
```

For PVCs, always specify the Rook/Ceph StorageClass explicitly:

```yaml
storageClassName: rook-ceph-block   # Or rook-cephfs for RWX
accessModes:
  - ReadWriteOnce
resources:
  requests:
    storage: "{{ kafka_storage_size }}"
```

### Cilium (CNI)

Default-deny network policy — always deploy a NetworkPolicy with every workload:

```yaml
# Deny all ingress/egress by default, then add explicit allow rules
- name: Apply default-deny NetworkPolicy
  kubernetes.core.k8s:
    state: present
    run_once: true
    delegate_to: localhost
    definition:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      metadata:
        name: default-deny-all
        namespace: "{{ target_namespace }}"
      spec:
        podSelector: {}
        policyTypes:
          - Ingress
          - Egress
```

For L7 visibility or Kafka-protocol-aware policies, use `CiliumNetworkPolicy` (CRD):

```yaml
- name: Apply CiliumNetworkPolicy for Kafka
  kubernetes.core.k8s:
    state: present
    apply: true
    merge_type:
      - merge   # CiliumNetworkPolicy is a CRD — use merge, not strategic-merge
    run_once: true
    delegate_to: localhost
    template: cilium-kafka-policy.j2
```

### Talos Linux specifics

- Talos has **no SSH access**. All node-level operations are via `talosctl`.
- Talos is immutable. Configuration changes require `talosctl apply-config` + reboot.
- Kernel parameters, CNI config, and kubelet settings are in `machineconfig` — not editable in-place.
- `talosctl` is not an Ansible module. Use `ansible.builtin.command` with `delegate_to: localhost` and always check the return code.

```yaml
- name: Apply Talos machine configuration
  ansible.builtin.command:
    cmd: >
      talosctl apply-config
      --nodes {{ item }}
      --file {{ role_path }}/files/talos-machineconfig.yaml
      --dry-run
  loop: "{{ talos_node_ips }}"
  run_once: false   # Must run per node
  delegate_to: localhost
  register: talos_apply_result
  changed_when: "'Applied configuration' in talos_apply_result.stdout"
```

> ⚠️ Always `--dry-run` first for `talosctl apply-config`. Reboot is required for changes to take effect and cannot be undone without re-applying the previous config + reboot.

---

## Step 9 — Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `No module named 'kubernetes'` | Python `kubernetes` package missing | `pip install kubernetes>=24.2.0` |
| `No resources found` (k8s_info on CRD) | CRD not yet installed or not Established | Wait for CRD Established condition before deploying CRs |
| `strategic merge patch format is not supported` | CRD doesn't support strategic-merge | Add `merge_type: [merge]` to the k8s task |
| `Task reported changed on re-run` (idempotency failure) | Missing `apply: true`, or resource definition contains mutable auto-populated fields | Use `apply: true`; strip server-generated fields from definition |
| `Error: UPGRADE FAILED: another operation is in progress` (Helm) | Previous Helm operation timed out and left a lock | `helm rollback <release> -n <ns>` or delete the lock secret |
| `OpenBao: error checking seal status: ...connection refused` | OpenBao pod is sealed or not yet started after eviction | Check pod status; manually unseal using stored unseal keys |
| `failed: [localhost] FAILED! => {"msg": "No Ansible inventory..."}` | Playbook run from wrong directory | Run from the Ansible project root where `ansible.cfg` is located |
| Pod stuck in `Pending` | No nodes match affinity/taint rules, or Rook/Ceph StorageClass unavailable | Check `kubectl describe pod <name>`, verify Ceph health and StorageClass |
| `CephCluster not in Ready phase` | Ceph OSD or monitor problem | Check `kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status` |
| `ImagePullBackOff` | Image tag wrong, registry unreachable, or missing imagePullSecret | Check `kubectl describe pod`, verify registry access and secret |

---

## Step 10 — Observability and Health Verification

After every deployment, run these checks:

```bash
# Pod health
kubectl get pods -n <namespace> -o wide

# Resource conditions
kubectl get <kind> <name> -n <namespace> -o jsonpath='{.status.conditions}' | jq .

# Recent events (catches scheduling, image pull, and readiness failures)
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# Logs from all pods in a StatefulSet
kubectl logs -n <namespace> -l <label-selector> --tail=50 --prefix

# Rook/Ceph health (always after storage operations)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd status
```

Key health signals:
- ✅ All pods: `Running` or `Completed`, 0 restarts
- ✅ CephCluster: `HEALTH_OK`
- ✅ Ansible re-run: all tasks `ok`, zero `changed`
- ❌ Pod: `CrashLoopBackOff` → check logs, check resource limits
- ❌ Pod: `Pending` → check node affinity, taints, PVC binding, resource requests
- ❌ CephCluster: `HEALTH_WARN` → do NOT drain nodes until resolved

---

## Anti-patterns — Never Do These

- [ ] Do not omit `run_once: true` and `delegate_to: localhost` from kubernetes.core tasks
- [ ] Do not use short module names — always use FQCN (`kubernetes.core.k8s`, not `k8s`)
- [ ] Do not write tasks without explicit `state:` — rely on defaults
- [ ] Do not skip check mode before applying — always run `--check --diff` first
- [ ] Do not store secret values in `vars/main.yml` or `defaults/main.yml` — use `vault.yml`
- [ ] Do not drain a Talos node when Ceph is not `HEALTH_OK`
- [ ] Do not assume OpenBao will auto-unseal after pod restart — it will not
- [ ] Do not deploy CRD-dependent resources before the CRD is in `Established` condition
- [ ] Do not use `k8s_exec` for state-modifying operations — it bypasses idempotency
- [ ] Do not apply without verifying idempotency (re-run after apply, expect 0 changed)
- [ ] Do not hardcode namespace, image tag, or cluster name — use variables from `defaults/main.yml`
- [ ] Do not use `strategic-merge` for CRDs — use `merge_type: [merge]`
- [ ] Do not use `force: true` on k8s module unless absolutely necessary — it replaces the entire resource
- [ ] Do not ignore the Justfile — check it first and use it where applicable
- [ ] Do not deploy a workload without an accompanying NetworkPolicy

---

## References

- [kubernetes.core collection index](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/index.html)
- [kubernetes.core.k8s module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/k8s_module.html)
- [kubernetes.core.k8s_info module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/k8s_info_module.html)
- [kubernetes.core.k8s_log module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/k8s_log_module.html)
- [kubernetes.core.k8s_exec module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/k8s_exec_module.html)
- [kubernetes.core.k8s_rollback module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/k8s_rollback_module.html)
- [kubernetes.core.helm module](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/helm_module.html)
- [Ansible roles guide](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible tips and tricks](https://docs.ansible.com/projects/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Ansible Vault guide](https://docs.ansible.com/projects/ansible/latest/vault_guide/vault_using_encrypted_content.html)
- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [Strimzi Kafka operator](https://strimzi.io/docs/operators/latest/overview.html)
- [Rook/Ceph documentation](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
- [Cilium network policy](https://docs.cilium.io/en/stable/network/kubernetes/policy/)
- [Talos Linux documentation](https://www.talos.dev/latest/introduction/what-is-talos/)
- [OpenBao documentation](https://openbao.org/docs/)
