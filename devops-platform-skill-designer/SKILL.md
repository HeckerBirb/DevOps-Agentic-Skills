---
name: devops-platform-skill-designer
description: >
  Design and create high-quality, production-grade SKILL.md files for DevOps
  and Platform Engineering tools, workflows, and practices. Use when creating
  skills for infrastructure automation, orchestration, configuration management,
  CI/CD, GitOps, observability, secrets management, service meshes, or platform
  engineering (e.g., Kubernetes, Ansible, Terraform, Helm, ArgoCD, Flux,
  Crossplane, Docker, Prometheus, Grafana, Loki, Vault, Backstage, GitHub
  Actions, Tekton, Istio, Cilium, cert-manager, External Secrets Operator).
  Produces skills that are operationally safer, more grounded, and more complete
  than generic skill creators.
---

# DevOps & Platform Engineering Skill Designer

A specialized meta-skill for producing high-quality, production-grade Copilot
skills in the DevOps and Platform Engineering domain. It enforces operational
correctness, source-of-truth alignment, execution safety, and Day-2 readiness
that generic skill creators omit.

---

## Hard Rules

Follow these without exception. They are not guidelines.

1. **MUST complete all Phase 1 discovery questions before generating any skill content.**
2. **MUST NOT invent commands, flags, or API fields.** If a command cannot be verified via `--help`, official docs, or real repo examples, emit `<!-- NEEDS VERIFICATION: <description> -->` and note it to the user.
3. **MUST classify the operating model** (Phase 2) before selecting a profile template.
4. **MUST include a pre-check, expected signal, and failure interpretation** for every mutating step in the generated skill.
5. **MUST score the generated skill** using the Phase 5 rubric before saving. Block finalization if any dimension scores 0, or if Correctness Grounding or Execution Safety scores below 2.
6. **MUST ask the user where to save the skill** before writing any file.
7. **MUST NOT include sections that do not apply** to the specific tool or workflow. No boilerplate padding.

---

## Phase 1: Discovery

Ask the user ALL of the following. Do not proceed to Phase 2 until all are answered.
If a question is unclear, ask a follow-up before moving on.

```
1. What tool, workflow, or practice does this skill cover?
   Be specific — e.g., "Ansible playbook authoring for Linux node configuration"
   not just "Ansible".

2. What is the PRIMARY AUDIENCE for this skill?
   (Platform engineers / App developers / SREs / Security engineers / All engineers)

3. What is the AUTHORITATIVE SOURCE OF TRUTH and OPERATING MODEL?
   This is the most important question. Choose one or more:
   a) Imperative CLI — humans run commands directly, no persistent state
   b) Git-driven GitOps — PRs/pushes trigger reconciliation (ArgoCD, Flux)
   c) IaC state — a state file owns resources (Terraform, Pulumi, Crossplane)
   d) Controller/CRD — a Kubernetes operator manages the lifecycle
   e) SaaS/Cloud console — configuration lives in an external control plane

4. What day-of-operations SCOPE should this skill cover?
   Day 0 (bootstrap/install) / Day 1 (first deploy) / Day 2 (ongoing ops) / All

5. What is the TARGET ENVIRONMENT?
   (e.g., AWS EKS, GKE, bare-metal on-prem, hybrid multi-cloud, cloud-agnostic)

6. What PERMISSIONS, ROLES, or RBAC are required to execute this workflow?
   (e.g., cluster-admin, specific IAM role, Vault policy, namespace-scoped role)

7. What are the 3 MOST COMMON TASKS someone will ask this skill to help with?

8. What does SUCCESS look like? What is the OBSERVABLE END STATE?
   (e.g., "Pod is Running with the correct image tag and no restarts",
   "terraform plan shows 0 changes on re-run")

9. What are KNOWN PITFALLS, anti-patterns, or common mistakes to avoid?
   (Include tool-specific gotchas from your experience or the docs)
```

---

## Phase 2: Classify the Operating Model

Based on Question 3, assign the skill to one of these profiles.
For composite tools, combine profiles with clear section boundaries.

| Profile | When to use | Examples |
|---|---|---|
| **A — CLI Ops** | Imperative, human-driven, stateless commands | `kubectl exec/rollout`, `helm upgrade`, `ansible-playbook --check` |
| **B — Declarative/IaC** | State-owned, plan-apply, idempotent configuration | Terraform, Ansible (infra config), Helm chart authoring, Crossplane claims |
| **C — GitOps/Controller** | Async reconciler, Git is truth, controllers converge state | ArgoCD, Flux, Argo Rollouts, Kubernetes operators |

Also identify these orthogonal axes — they affect which template sections are required:

| Axis | Options | Impact on skill |
|---|---|---|
| Mutation scope | Read-only vs mutating | Mutating → requires pre-check + rollback |
| Blast radius | Scoped vs cluster/account-wide | Wide → requires explicit scope guard |
| Reversibility | Fully / Partially / Irreversible | Irreversible → requires forward-fix guidance |
| Execution context | Human-run vs pipeline-run | Pipeline → requires non-interactive flag guidance |
| State model | Stateless vs stateful | Stateful → requires state backend + drift detection |

---

## Phase 3: Research the Tool

Use this sequence to verify all content before including it. Do not skip steps.

**Step A — CLI surface (run directly if available):**
```bash
<tool> --help
<tool> <key-subcommand> --help
<tool> version
```

**Step B — Official documentation:**
Search for and fetch:
- Getting started / quickstart guide
- CLI or API reference
- Security / RBAC / access control guide
- Troubleshooting guide
- Changelog / migration guide (for version-specific behavior)

**Step C — Community patterns:**
Search for real-world usage patterns:
```
"<tool>" best practices site:github.com
"<tool>" production gotchas
"<tool>" common mistakes
filename:SKILL.md "<tool>"
```

**Step D — Failure modes:**
Look for:
- Open issues tagged `bug` or `help-wanted` in the tool's official repo
- Known edge cases in the troubleshooting guide
- Community forum posts about production incidents

> If any command cannot be verified: emit `<!-- NEEDS VERIFICATION: <description> -->` and notify the user before finalizing.

---

## Phase 4: Generate the Skill

Select the matching profile template below. For composite skills (e.g., CLI ops on
a GitOps-managed cluster), combine profiles with an explicit section header marking
the boundary and the ownership model at that boundary.

---

### Profile A — CLI Ops Skill

For imperative, human-driven, stateless command workflows.

```markdown
---
name: <skill-name>
description: >
  <One-sentence primary description.>
  Use when: <5–10 specific trigger phrases, comma-separated>.
  Operating model: Imperative CLI.
  Audience: <audience>.
  Scope: Day-<0/1/2> operations.
allowed-tools: shell
---

# <Skill Title>

> **Profile**: CLI Ops | **Audience**: <audience> | **Environment**: <env>
> **Blast radius**: <scoped/cluster-wide/account-wide>

<One paragraph: what this skill enables and what operational maturity it targets.>

## Prerequisites

### Required tools
| Tool | Min. version | Verify |
|---|---|---|
| `<tool>` | `<version>` | `<tool> version` |

### Required permissions
- [ ] <Role or permission — never a secret value, just the role name>
- [ ] Verify access: `<access-check command>`

### Context / scope guard
```bash
# Always verify you are targeting the correct context before any command
<context-check command>   # e.g., kubectl config current-context
```
> ⚠️ Confirm context before every mutating operation. Never assume.

---

## Core Workflow

### Step 1 — <Step title>

**Pre-check:**
```bash
<pre-check command>
```
Expected: `<expected output that confirms readiness>`

**Action:**
```bash
<command>
```

**Expected signal:**
```
<what success looks like in stdout, status output, or logs>
```

**If this fails:**
| Error | Cause | Fix |
|---|---|---|
| `<error message or pattern>` | <root cause> | <fix command or steps> |

---

### Step 2 — Preflight validation

**MUST run before any apply/mutate step.**

```bash
<dry-run / lint / diff / plan / policy-check command>
```

Review output for:
- [ ] Changes are scoped as expected
- [ ] No unintended resources are affected
- [ ] Blast radius is within acceptable bounds

---

### Step 3 — Apply

```bash
<apply command>
```

**Expected signal:**
```
<success output>
```

### Step 4 — Verify

```bash
<verification command>
```
Expected: `<the observable end state from discovery Q8>`

---

## Rollback / Recovery Strategy

> **Reversibility**: <Fully reversible / Partially / Irreversible>
> If irreversible, state this explicitly and provide forward-fix guidance instead.

```bash
# Rollback
<rollback command>

# Verify rollback succeeded
<verification command>
```

---

## Day-2 Operations

### Update / Upgrade
```bash
# Check current version vs available
<version-check command>

# Apply upgrade
<upgrade command>
```
Check for version skew or breaking changes: `<changelog or compatibility command>`

### Troubleshoot
```bash
# Status
<status command>

# Logs
<log command>

# Events / audit trail
<events command>
```

### Drift detection
```bash
<diff/plan/check command>
```

### Credential / cert rotation
```bash
<rotation procedure or reference>
```

### Cleanup / Decommission
```bash
<cleanup command>
```
> ⚠️ Destructive — confirm scope before running.

---

## Security & Credential Handling

- **Minimum required role**: `<role name>`
- **Secret source**: `<preferred backend, e.g., Vault, External Secrets>` — never hardcode values
- **Audit trail**: `<where to find audit logs or events>`
- **Verify least privilege**: `<RBAC/IAM verification command>`

---

## Observability

```bash
# Health / status
<status command>

# Logs
<log command>

# Events / audit trail
<events command>
```

Key signals:
- ✅ Healthy: `<log pattern or metric indicating success>`
- ❌ Failing: `<log pattern or metric indicating a problem>`

---

## Environment Variations

| Environment | Key differences |
|---|---|
| `dev` | <relaxed policies, ephemeral resources, etc.> |
| `staging` | <prod-like config, approval optional> |
| `prod` | <approval gate, change window, explicit blast-radius guard> |

---

## Anti-patterns — Never Do These

- [ ] Do not assume current context/namespace/account — always verify first
- [ ] Do not skip preflight validation before mutating operations
- [ ] Do not use cluster-admin / admin roles when a scoped role suffices
- [ ] Do not ignore diff/plan output — review it before applying
- [ ] <Tool-specific anti-pattern from discovery Q9>

---

## Integration Points

| Adjacent tool | How it connects | Ownership boundary |
|---|---|---|
| `<tool>` | <integration description> | <who owns what> |

---

## References

- [Official docs](<url>)
- [CLI reference](<url>)
- [Security / RBAC guide](<url>)
- [Troubleshooting guide](<url>)
```

---

### Profile B — Declarative/IaC Skill

For state-owned, plan-apply, idempotent configuration workflows.

```markdown
---
name: <skill-name>
description: >
  <One-sentence primary description.>
  Use when: <5–10 specific trigger phrases>.
  Operating model: Declarative/IaC — always plan before apply.
  Source of truth: <Git repo / Terraform state / Helm values / Crossplane claims>.
allowed-tools: shell
---

# <Skill Title>

> **Profile**: Declarative/IaC | **Source of truth**: <location>
> **State backend**: <backend> | **Blast radius**: <scope>

<One paragraph: what this manages, where authoritative state lives, expected idempotency guarantee.>

## Source of Truth

> All changes MUST flow through `<Git repo / state file / values file>`.
> Direct imperative changes bypass the source of truth and WILL cause drift.
> Drift is reconciled on the next plan-apply cycle and will overwrite manual changes.

---

## Prerequisites

### Required tools
| Tool | Min. version | Verify |
|---|---|---|
| `<tool>` | `<version>` | `<tool> version` |

### Access & state backend
- [ ] State backend accessible: `<backend connection check command>`
- [ ] Required role: `<role>`
- [ ] Secret source: `<backend>` — never store values in variable files

---

## Structure & Conventions

```
<expected file/directory structure>
```

Variable and secret handling:
- Inputs defined in: `<variables file>`
- Environment overrides in: `<env-specific override file>`
- Secrets referenced via: `<secret backend reference pattern — never values>`

---

## Workflow: Plan → Apply → Verify Idempotency

### Step 1 — Initialize / validate

```bash
<init command>
<validate/lint command>
```

Expected: `<clean validation output>`

### Step 2 — Plan / Diff

**MUST review plan output before applying. Do not apply without understanding every change.**

```bash
<plan command>
```

Review for:
- [ ] Only expected resources are changing
- [ ] No unintended `destroy` or replacement operations
- [ ] Blast radius is within acceptable scope for this change

If anything is unexpected: stop, investigate, do not apply.

### Step 3 — Apply

```bash
<apply command>
```

**Expected signal:** `<success output>`

### Step 4 — Verify idempotency

```bash
# Re-run plan immediately after apply — MUST show 0 changes
<plan command>
```
Expected: `<"0 changes" / "no diff" / "already in sync" output>`

---

## Rollback / Recovery Strategy

> **Reversibility**: <level>

**Primary recovery path — always prefer reverting in the source of truth:**
```bash
# Revert the change in Git or the configuration file, then re-apply
git revert <commit>    # if Git-backed
<apply command>
```

**State restore** (if state is corrupted or diverged):
```bash
<state restore / import command>
```

**If the operation is irreversible** (e.g., resource destroy, DB schema migration, secret rotation):
> Do not attempt to undo. Instead: `<forward-fix or restore-from-backup steps>`

---

## Drift Detection

```bash
# Run plan with no intended changes — any output indicates drift
<plan command>
```

If drift is detected:
1. Identify the source of the drift (manual change? controller?)
2. Reconcile via source of truth — never fix drift with direct imperative commands

---

## Day-2 Operations

### Module / provider / role upgrade
```bash
<update-lock / version-bump command>
<plan after upgrade>   # Review for breaking changes before applying
```

### State inspection
```bash
<state list command>
<state show <resource> command>
```

### Credential / cert rotation
```bash
<rotation procedure — rotate secret at source, re-apply to propagate>
```

### Cleanup / Decommission
```bash
<destroy / remove command>   # ⚠️ Destructive — review plan first
```

---

## Security & Credential Handling

- **State encryption**: <yes/no and method>
- **Secret injection**: `<backend reference pattern — never literal values>`
- **Audit trail**: `<state history / git log / audit log location>`
- **Minimum role**: `<role>` — verify with `<check command>`

---

## Environment Variations

| Env | State backend | Override file | Approval required |
|---|---|---|---|
| `dev` | `<>` | `<file>` | No |
| `staging` | `<>` | `<file>` | Recommended |
| `prod` | `<>` | `<file>` | Yes — peer review of plan output |

---

## Anti-patterns — Never Do These

- [ ] Do not apply without reviewing plan output in full
- [ ] Do not make direct imperative changes to resources owned by this tool
- [ ] Do not store secret values in variable files — use `<secret backend>`
- [ ] Do not run destroy in staging/prod without explicit approval and plan review
- [ ] Do not ignore idempotency failures (non-zero plan after apply = something is wrong)
- [ ] <Tool-specific anti-pattern from discovery Q9>

---

## References

- [Official docs](<url>)
- [State backend docs](<url>)
- [Best practices guide](<url>)
- [Security / secret handling](<url>)
```

---

### Profile C — GitOps/Controller-Driven Skill

For async reconciler workflows where Git is the source of truth and controllers converge state.

```markdown
---
name: <skill-name>
description: >
  <One-sentence primary description.>
  Use when: <5–10 specific trigger phrases>.
  Operating model: GitOps — Git is source of truth, controller reconciles async.
  Reconciler: <ArgoCD / Flux / Argo Rollouts / <Operator name>>.
allowed-tools: shell
---

# <Skill Title>

> **Profile**: GitOps/Controller | **Reconciler**: <name>
> **Sync model**: <automated / manual gate> | **Blast radius**: <scope>

<One paragraph: what the controller manages, what triggers reconciliation,
and the expected convergence time.>

## Operating Model

> **Source of truth**: `<Git repository>` at `<branch>` path `<path>`
> **Reconciliation trigger**: <push to branch / PR merge / manual sync>
> **Expected convergence time**: <e.g., "30–90 seconds after sync">
>
> ⚠️ NEVER run `kubectl apply` or equivalent directly on resources managed by
> this controller. Direct changes will be overwritten on the next reconciliation
> and will not be reflected in the source of truth.

---

## Prerequisites

| Tool | Min. version | Verify |
|---|---|---|
| `<tool>` | `<version>` | `<tool> version` |

Access:
- [ ] Git write access to `<repo>` on branch `<branch>`
- [ ] Controller CLI access: `<verify command>`
- [ ] Required role: `<role>`

---

## Workflow: Commit → Sync → Verify Convergence

### Step 1 — Author the change in Git

Edit files in the repository at:
```
<repo>/<path>/<resource file>
```

**Validate before committing:**
```bash
<lint/validate command>   # e.g., helm lint, kustomize build, kubeval, kubeconform
```

**Expected signal:** `<clean validation output>`

### Step 2 — Commit and push

```bash
git add <file>
git commit -m "<conventional commit message>"
git push origin <branch>
```

> If using PR-based GitOps: open a PR and await approval before merging to the target branch.

### Step 3 — Check sync status

```bash
<sync status command>   # e.g., argocd app get <app> --refresh, flux get ks <name>
```

Expected: `<Synced / reconciling status>`

If automated sync is not enabled, trigger manually:
```bash
<manual sync command>   # Use sparingly — prefer automated sync
```

### Step 4 — Verify convergence

```bash
<wait/watch command>   # e.g., kubectl rollout status, flux get ks --watch

# Confirm health
<health check command>
```

Expected: `<the observable end state from discovery Q8>`

**Reconciliation signals:**
- ✅ Healthy: `<signal indicating successful convergence>`
- ❌ Degraded: `<signal indicating failed reconciliation>`

**If reconciliation stalls or fails:**
```bash
<describe/events command>   # e.g., kubectl describe <resource>
<controller logs command>   # e.g., kubectl logs -n <ns> <controller-pod>
```

---

## Rollback / Recovery Strategy

> **Reversibility**: <level>

**Primary recovery — always prefer reverting in Git:**
```bash
git revert <commit>
git push origin <branch>
# Controller will reconcile back to previous state
```

**Emergency override** (use ONLY if the controller is unavailable or causing an outage):
```bash
# 1. Suspend reconciliation to prevent the controller overwriting your fix
<suspend command>   # e.g., argocd app terminate-op, flux suspend ks

# 2. Apply manual remediation
<manual remediation steps>

# 3. MANDATORY: Re-enable reconciliation
<resume command>   # e.g., flux resume ks

# 4. Sync source of truth to match the current live state
<update Git to reflect current state>
```

> ⚠️ Always resume reconciliation after manual intervention. Leaving it suspended
> causes indefinite drift accumulation.

---

## Day-2 Operations

### Check sync and health status
```bash
<status command>
```

### View reconciliation history / audit
```bash
<history command>
```

### Detect drift between Git and live state
```bash
<diff command>   # e.g., argocd app diff, flux diff ks
```

### Promote a change between environments
```bash
# Open a PR from <source-env-branch> to <target-env-branch>
# Approval gate applies per environment tier (see Environment Variations)
```

### Upgrade the controller / CRDs
```bash
<version check command>
# Follow the official upgrade guide — CRD upgrades often require specific ordering
<upgrade command>
```

### Suspend / resume reconciliation (maintenance windows)
```bash
<suspend command>
# Perform maintenance
<resume command>
```

---

## Security & Credential Handling

- **Git access**: `<SSH key or deploy token — stored in secret backend, not plaintext>`
- **Controller service account**: `<role / RBAC — minimum required>`
- **Secret management**: `<External Secrets / Vault / SOPS / Sealed Secrets>`
  > Never commit plaintext secrets to Git, even in private repositories.
- **Audit trail**: `<controller audit log / Git history location>`

---

## Observability

```bash
# Controller health
<controller pod status command>

# Reconciliation events
<events command>

# Sync lag / convergence time
<metric or watch command>
```

Key signals:
- ✅ `<signal indicating healthy reconciliation>`
- ❌ `<signal indicating sync or health failure>`
- ⏳ `<signal indicating reconciliation is still in progress>`

---

## Environment Variations

| Env | Sync policy | Approval gate | Target cluster/namespace |
|---|---|---|---|
| `dev` | Automated | None | `<>` |
| `staging` | Automated | PR review | `<>` |
| `prod` | Manual gate | Peer approval + diff review | `<>` |

---

## Anti-patterns — Never Do These

- [ ] Do not `kubectl apply` / directly mutate resources managed by this controller
- [ ] Do not leave reconciliation suspended indefinitely
- [ ] Do not commit plaintext secrets to Git — use `<secret backend>`
- [ ] Do not promote changes to prod without a path through staging
- [ ] Do not bypass the PR approval gate for prod changes
- [ ] <Tool-specific anti-pattern from discovery Q9>

---

## References

- [Controller official docs](<url>)
- [GitOps best practices](<url>)
- [Secret management guide](<url>)
- [Upgrade guide](<url>)
```

---

## Phase 5: Quality Scoring Rubric

Score every dimension before saving. Do not finalize if any dimension = 0.
**Correctness Grounding and Execution Safety MUST both score 2.**

| Dimension | Score 0 | Score 1 | Score 2 |
|---|---|---|---|
| **Correctness Grounding** | Commands invented without verification | Some commands verified, some assumed | All commands verified via `--help`, official docs, or real-world repo examples |
| **Execution Safety** | No preflight / dry-run step | Basic dry-run present | Blast radius assessed, pre-checks defined, abort criteria clear |
| **Rollback / Recovery** | Missing entirely | Generic "undo" mentioned with no commands | Tool-specific recovery path with runnable commands and reversibility caveat |
| **Observability** | Missing | Status command only | Health signal + convergence/reconciliation signal + audit trail + failure pattern |
| **Environment Specificity** | Single env assumed throughout | Dev/prod mentioned | Dev/staging/prod variations with approval gates documented |
| **Least Privilege** | Admin role assumed | A role is mentioned | Minimum RBAC/IAM role defined with a verification command |
| **Composability** | Isolated, no ecosystem context | Adjacent tools listed | Integration boundaries and ownership responsibilities defined |

**Scoring gate**: All ≥ 1. Correctness Grounding = 2. Execution Safety = 2.
If the gate fails: identify which dimensions scored below threshold, fix the generated content, and re-score.

---

## Phase 6: Save & Register

Ask the user:
```
Where should this skill be saved?
  1. Personal skill (available across all projects):
     ~/.copilot/skills/<skill-name>/SKILL.md
  2. Project skill (this repo only):
     .github/skills/<skill-name>/SKILL.md
```

Create the directory and write the file. Then verify registration:
```bash
copilot skill list
```

Confirm the new skill name appears in the output.

---

## Global Anti-patterns — This Skill MUST NEVER Produce

The generated skill must not:

- Recommend bypassing the source of truth (e.g., direct `kubectl apply` on GitOps-managed resources, direct infra edits on Terraform-owned resources)
- Embed or display literal secret values, tokens, passwords, or private keys
- Default to admin / cluster-admin roles when a scoped role would suffice
- Present `<!-- NEEDS VERIFICATION -->` placeholders as if they are runnable commands
- Assume the user's current context, namespace, cluster, or cloud account — always verify explicitly
- Skip preflight validation (dry-run / lint / plan / diff) before any mutating operation
- Omit rollback or recovery guidance for destructive or partially irreversible operations
- Mix Day-0 bootstrapping with Day-2 incident response in the same workflow without clear section separation
- Pad sections with generic content that does not apply to the specific tool or workflow
- Present rollback as trivial when the operation is partially or fully irreversible
