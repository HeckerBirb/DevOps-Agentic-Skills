# devops-platform-skill-designer

A GitHub Copilot CLI meta-skill for designing and creating high-quality,
production-grade `SKILL.md` files for DevOps and Platform Engineering tools,
workflows, and practices.

## Overview

Generic skill creators produce plausible-looking output with DevOps-flavoured
headings but weak operational content — invented commands, missing rollback,
no blast radius assessment, and no source-of-truth alignment.

This skill fixes that by enforcing a structured 6-phase design process with
hard rules, mandatory discovery questions, three differentiated profile
templates, and a scoring rubric that blocks finalization if critical quality
dimensions fail.

**Use this skill to create other skills** — particularly for infrastructure
automation, orchestration, GitOps, observability, secrets management, service
meshes, or platform engineering tools.

## When to Use

Invoke this skill when asking Copilot to:

- Create a new Copilot skill for any DevOps or Platform Engineering tool
- Produce a skill that covers Day-0 through Day-2 operations correctly
- Design a skill that handles rollback, observability, and environment
  variations — not just the happy path
- Audit the quality of an existing skill against operational standards
- Generate skills for tools such as: Kubernetes, Ansible, Terraform, Helm,
  ArgoCD, Flux, Crossplane, Docker, Prometheus, Grafana, Loki, Vault,
  Backstage, GitHub Actions, Tekton, Istio, Cilium, cert-manager, External
  Secrets Operator, and similar

## How It Works

The skill runs through six phases before writing a single line of output:

### Phase 1 — Discovery (9 questions)

Collects: tool scope, audience, **source of truth and operating model** (the
most critical question), day-of-operations scope, target environment, required
permissions, top 3 tasks, observable success criteria, and known pitfalls.

The skill will not proceed to Phase 2 until all 9 questions are answered.

### Phase 2 — Operating Model Classification

Classifies the skill into one or more profiles and orthogonal axes:

| Profile | When | Examples |
|---|---|---|
| **A — CLI Ops** | Imperative, human-driven, stateless | `kubectl`, `helm upgrade`, `ansible-playbook` |
| **B — Declarative/IaC** | State-owned, plan-apply, idempotent | Terraform, Ansible roles, Helm chart authoring |
| **C — GitOps/Controller** | Async reconciler, Git is truth | ArgoCD, Flux, Kubernetes operators |

Orthogonal axes assessed: mutation scope · blast radius · reversibility ·
execution context (human vs pipeline) · state model.

### Phase 3 — Research

Verifies all commands against `--help` output, official documentation, and
real-world repository examples before including them.

Any command that cannot be verified is emitted as
`<!-- NEEDS VERIFICATION: description -->` and flagged to the user.
**No commands are invented.**

### Phase 4 — Generate

Selects and populates the matching profile template. Composite skills (e.g.,
CLI ops on a GitOps-managed cluster) combine profiles with explicit section
headers and ownership boundaries.

Every mutating step in the generated skill includes:
- Pre-check command + expected signal
- Action command
- Expected success signal
- Failure interpretation table

### Phase 5 — Quality Scoring

Scores the generated skill on 7 dimensions before saving:

| Dimension | Score 2 requires |
|---|---|
| Correctness Grounding | All commands verified from docs or `--help` |
| Execution Safety | Blast radius assessed, preflight defined, abort criteria clear |
| Rollback / Recovery | Tool-specific recovery path with runnable commands |
| Observability | Health + convergence signal + audit trail + failure pattern |
| Environment Specificity | Dev/staging/prod with approval gates documented |
| Least Privilege | Minimum role defined with a verification command |
| Composability | Integration boundaries and ownership responsibilities defined |

**Gate**: All dimensions ≥ 1. Correctness Grounding and Execution Safety must
both equal 2. Finalization is blocked if the gate fails.

### Phase 6 — Save and Register

Asks where to save (personal `~/.copilot/skills/` or project `.github/skills/`),
writes the file, and verifies registration with `copilot skill list`.

## What Makes This Better Than Generic Skill Creators

| Generic creator | This designer |
|---|---|
| One template for all tools | Three profiles matched to the operating model |
| No source-of-truth question | Source of truth is the first classification axis |
| Commands assumed or invented | Unverified commands emit `<!-- NEEDS VERIFICATION -->` |
| Checklist at the end | Scoring rubric blocks finalization on failure |
| No blast radius concept | Blast radius + reversibility assessed per mutating step |
| Generic rollback section | Tool-specific recovery with runnable commands |
| Boilerplate sections always included | Sections only included if applicable |
| No Day-2 lifecycle | Upgrade, drift detection, credential rotation, decommission |
| No anti-hallucination guard | Hard rule: MUST NOT invent commands or flags |

## Hard Rules

The skill operates under seven rules that cannot be overridden:

1. MUST complete all Phase 1 discovery questions before generating content
2. MUST NOT invent commands, flags, or API fields
3. MUST classify the operating model before selecting a profile
4. MUST include pre-check, expected signal, and failure interpretation for every mutating step
5. MUST score the skill using the Phase 5 rubric before saving
6. MUST ask the user where to save the skill before writing any file
7. MUST NOT include sections that do not apply — no boilerplate padding

## Installation

```bash
# Personal skill — available across all projects
copilot skill add https://raw.githubusercontent.com/<org>/<repo>/main/skills/devops-platform-skill-designer/SKILL.md

# Project skill — this repo only
copilot skill add --project ./skills/devops-platform-skill-designer/SKILL.md
```

Verify:

```bash
copilot skill list
# Should show: devops-platform-skill-designer
```

## Example Prompts

```
Use the /devops-platform-skill-designer skill to create a new skill for
managing cert-manager certificate issuers and ClusterIssuers with Ansible.
```

```
Use the /devops-platform-skill-designer skill to create a skill for
deploying and operating ArgoCD application sets via GitOps.
```

```
Use the /devops-platform-skill-designer skill to design a skill for
Terraform workflows targeting AWS EKS with remote S3 state.
```

## Compatible With

This skill can be used alongside:

- **`create-skill`** (`abdullahkhawer/devops-skills`) — for rapid scaffolding
  when deep operational correctness is less critical
- **`microsoft-skill-creator`** (`github/awesome-copilot`) — for Microsoft
  technology skills backed by the Learn MCP server

When used together, let `devops-platform-skill-designer` drive the design
process and use the generic creators as reference for structure only.

## License

MIT
