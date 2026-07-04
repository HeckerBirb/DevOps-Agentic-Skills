# ansible-k8s-role-development

A GitHub Copilot CLI skill for developing, authoring, modifying, and debugging
Ansible roles that manage Kubernetes resources using the `kubernetes.core`
collection.

## Overview

This skill encodes production-grade patterns for teams using Ansible as their
primary tool for managing Kubernetes workloads declaratively. It enforces
idempotency, execution safety, secret handling, and Day-2 operational readiness
across the full role development lifecycle.

All tasks in generated roles run against the Kubernetes API from localhost via
`~/.kube/config` — no SSH to cluster nodes is required or used.

**Target stack:**

| Layer | Technology |
|---|---|
| Node OS | Talos Linux (immutable, no SSH, `talosctl`) |
| CNI | Cilium (L3/L4 NetworkPolicy + L7 CiliumNetworkPolicy) |
| CSI | Rook/Ceph (block and filesystem storage) |
| Secrets | OpenBao (Vault fork — manual unseal after pod restart) |
| Ansible auth | `~/.kube/config` (cluster-admin) + Ansible Vault |
| Execution model | `run_once: true` + `delegate_to: localhost` on all K8s module tasks |

## When to Use

Invoke this skill when asking Copilot to:

- Write a new Ansible role that deploys a workload to Kubernetes
- Add or modify tasks in an existing role
- Deploy production-grade platform components (Kafka, OpenBao, cert-manager,
  monitoring stacks, Rook/Ceph operators, Cilium policies, etc.)
- Debug a failing Ansible playbook against the cluster
- Review an existing role for Ansible and Kubernetes best practices
- Handle secrets securely using Ansible Vault
- Troubleshoot Kubernetes resource states non-destructively

## What the Skill Enforces

### Ansible conventions
- Fully Qualified Collection Names (FQCN) on every module — `kubernetes.core.k8s`, not `k8s`
- Explicit `state:` on every task — never rely on module defaults
- Descriptive task names on all tasks, blocks, and plays
- `vars/` + `vault.yml` split pattern — variable names remain grep-visible; only encrypted values go in the vault file

### Kubernetes correctness
- `k8s_info` pre-check before every mutating `k8s` task
- CRD `Established` condition guard (via `until` loop) before deploying Custom Resources
- `merge_type: [merge]` for Custom Resource Definitions — strategic-merge is not supported on CRDs
- `apply: true` for idempotent server-side merges
- `wait: true` with explicit `wait_timeout` for Deployments, StatefulSets, and DaemonSets

### Execution safety
- `--check --diff` (dry-run) before every apply — mandatory, not optional
- Idempotency verification — re-run after apply and expect zero `changed`
- Check mode (`ansible_check_mode`) guard on `k8s_exec` tasks — exec has side effects
- Ceph health gate (`HEALTH_OK`) before any node drain or eviction

### Secret and credential handling
- Ansible Vault for all secrets — never stored as plaintext in any variable file
- Auto-generated credentials (OpenBao root token, admin passwords) are captured,
  vault-encrypted, and committed before any dependent operation proceeds
- Password rotation pattern with post-rotation verification
- `no_log: true` on all tasks that handle unseal keys or credential values

### Stack-specific awareness
- **Talos Linux**: immutable OS, `talosctl apply-config` + reboot required for node changes; `--dry-run` always used first
- **Cilium**: default-deny NetworkPolicy with every workload; `CiliumNetworkPolicy` for L7
- **Rook/Ceph**: StorageClass specified explicitly on every PVC; Ceph health checked before storage-affecting operations
- **OpenBao**: sealed after pod restart by design; manual unseal procedure with Ansible task included

### Justfile integration
- The skill reads and analyses the project's `Justfile` at the start of every
  session and uses its recipes in preference to raw `ansible-playbook` commands

## Module Reference

Verified against `kubernetes.core` collection version **6.4.0**:

| Module | Use |
|---|---|
| `kubernetes.core.k8s` | Create, update, delete K8s objects |
| `kubernetes.core.k8s_info` | Read-only queries and pre-checks |
| `kubernetes.core.k8s_log` | Fetch pod and container logs |
| `kubernetes.core.k8s_exec` | Execute diagnostic commands in pods (read-only only) |
| `kubernetes.core.k8s_rollback` | Roll back Deployments and DaemonSets |
| `kubernetes.core.k8s_scale` | Scale replicas |
| `kubernetes.core.k8s_json_patch` | Patch existing resources with JSON patch |
| `kubernetes.core.helm` | Manage Helm releases |
| `kubernetes.core.helm_info` | Query Helm release status |
| `kubernetes.core.k8s_drain` | Drain, cordon, or uncordon nodes |

## Prerequisites

| Tool | Minimum version | Verify |
|---|---|---|
| `ansible-core` | 2.16.0 | `ansible --version` |
| `kubernetes.core` collection | 6.4.0 | `ansible-galaxy collection list \| grep kubernetes.core` |
| Python `kubernetes` | 24.2.0 | `python3 -c "import kubernetes; print(kubernetes.__version__)"` |
| Python `PyYAML` | 3.11 | `python3 -c "import yaml; print(yaml.__version__)"` |
| Python `jsonpatch` | any | `python3 -c "import jsonpatch; print('ok')"` |
| `kubectl` | — | `kubectl version --client` |
| `helm` | 3.0.0 | `helm version` |

Install collection and Python dependencies:

```bash
ansible-galaxy collection install kubernetes.core
pip install kubernetes>=24.2.0 PyYAML>=3.11 jsonpatch
```

## Installation

```bash
# Personal skill — available across all projects
copilot skill add https://raw.githubusercontent.com/<org>/<repo>/main/skills/ansible-k8s-role-development/SKILL.md

# Project skill — this repo only
copilot skill add --project ./skills/ansible-k8s-role-development/SKILL.md
```

Verify:

```bash
copilot skill list
# Should show: ansible-k8s-role-development
```

## Example Prompts

```
Use the /ansible-k8s-role-development skill to write an Ansible role that
deploys a production-ready Kafka cluster with 3 brokers, TLS encryption,
Rook/Ceph persistent storage, and Cilium network policies.
```

```
Use the /ansible-k8s-role-development skill to help me debug why my
OpenBao deployment role is failing on the unseal step.
```

```
Use the /ansible-k8s-role-development skill to review the kafka role in
roles/kafka/ for idempotency and best practices.
```

## References

- [kubernetes.core collection](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/index.html)
- [Ansible roles guide](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible tips and tricks](https://docs.ansible.com/projects/ansible/latest/tips_tricks/ansible_tips_tricks.html)
- [Talos Linux documentation](https://www.talos.dev/latest/introduction/what-is-talos/)
- [Cilium network policy](https://docs.cilium.io/en/stable/network/kubernetes/policy/)
- [Rook/Ceph documentation](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
- [OpenBao documentation](https://openbao.org/docs/)
- [Strimzi Kafka operator](https://strimzi.io/docs/operators/latest/overview.html)

## License

MIT
