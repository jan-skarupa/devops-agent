# devops-agent

A two-skill [Hermes Agent](https://hermes-agent.nousresearch.com) bundle that acts as a DevOps engineer for a single AWS account. The first iteration targets EKS Kubernetes minor-version upgrades with a `plan → approve → execute` flow.

| Skill | Purpose |
|---|---|
| `/eks-upgrade-plan` | Inspects the cluster, runs PoC readiness checks, writes an approvable plan to disk. **Read-only**. |
| `/eks-upgrade-execute` | Reads the approved plan and walks the steps — pauses for confirmation before every mutating step. **Mutating**. |

This is a Proof of Concept. It shells out to standard CLIs (`aws`, `kubectl`, `helm`), keeps no state outside the plan file, and assumes one human + one cluster per session.

---

## Prerequisites

- **Hermes Agent** installed locally. See [Hermes quickstart](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart).
- The following CLIs on your `PATH`:
  - `aws` (AWS CLI v2)
  - `kubectl`
  - `helm`
- AWS credentials available to the shell that starts Hermes — via env vars (`AWS_PROFILE`, `AWS_DEFAULT_REGION`, or `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`), `~/.aws/credentials`, or an attached instance/role.
- The IAM principal must be able to:
  - read EKS (`eks:DescribeCluster`, `eks:List*`, `eks:DescribeAddon*`, `eks:DescribeNodegroup`)
  - for execute: also `eks:Update*` plus EC2/IAM permissions sufficient to drive node-group rolls
- Network reach to the EKS API server (your cluster's endpoint must be reachable from where Hermes runs).

---

## Install Hermes

```bash
pip install hermes-agent
hermes postinstall
```

(Or use the curl installer from the Hermes docs, which also installs `uv`, Python 3.11, Node, ripgrep, ffmpeg.)

Verify:
```bash
hermes --version
```

---

## Install these skills

There are two ways to make Hermes see the skills in this repo. Pick one.

### Option A — point Hermes at this repo (recommended for development)

Edit `~/.hermes/config.yaml` and add (or extend) an `external_dirs` entry pointing at the `skills/` directory of this repo:

```yaml
skills:
  external_dirs:
    - /absolute/path/to/devops-agent/skills
```

Then reload:
```bash
hermes skills list
```

You should see `eks-upgrade-plan` and `eks-upgrade-execute` in the list.

### Option B — copy the skills into `~/.hermes/skills/`

```bash
mkdir -p ~/.hermes/skills/eks
cp -r skills/eks/eks-upgrade-plan    ~/.hermes/skills/eks/
cp -r skills/eks/eks-upgrade-execute ~/.hermes/skills/eks/
hermes skills list
```

Either way, no additional config is needed — the skills declare what they use via Hermes' built-in `terminal` tool.

---

## Usage

Start a Hermes session in the directory where you want the plan file to land (e.g. a working directory for this cluster):

```bash
cd ~/work/my-eks
hermes
```

Then, inside the session:

```
/eks-upgrade-plan us-east-1 my-cluster
```

Optional flags: `--target 1.34` to pin the target version (default = current minor + 1).

The skill:
1. Connects `kubectl` to the cluster via `aws eks update-kubeconfig`.
2. Runs readiness checks (version path, deprecated APIs, node/workload health, PDBs, helm releases, add-ons).
3. Writes the plan to `./.plan/eks-upgrade/<cluster>-<target>.md`.
4. Prints a summary and waits for your reply.

Reply with one of:
- **`APPROVE`** — marks the plan executable. Refused if the plan is `NOT READY`.
- A free-text change request (e.g. "skip the coredns step", "add a step to upgrade aws-load-balancer-controller first") — the skill regenerates the plan.

Once approved:

```
/eks-upgrade-execute us-east-1 my-cluster
```

The execute skill reads the approved plan and walks the steps. Before every mutating step it asks `Proceed? (yes/no/skip)`. On failure it shows the rollback line from the plan and asks whether to roll back.

---

## Plan file

Location: `./.plan/eks-upgrade/<cluster>-<target>.md`, relative to the Hermes session's working directory.

Shape:
```markdown
---
cluster: my-cluster
region: us-east-1
current_version: "1.33"
target_version: "1.34"
approved: false
generated_at: 2026-05-28T10:15:00Z
---

# Preflight summary
Status: READY WITH WARNINGS
Blocking issues:
  - None
Warnings:
  - aws-load-balancer-controller should be upgraded before the control plane.

# Proposed execution plan
[1] Upgrade aws-load-balancer-controller Helm release
    Type: mutating
    Risk: medium
    Command: helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --version 1.8.1 --reuse-values
    Rollback: helm rollback aws-load-balancer-controller <prev-rev> -n kube-system

[2] Upgrade EKS control plane 1.33 -> 1.34
    Type: mutating
    Risk: high
    Command: aws eks update-cluster-version --region us-east-1 --name my-cluster --kubernetes-version 1.34
    Rollback: not supported by EKS

[3] ...
```

`approved: true` is the only gate `/eks-upgrade-execute` checks. The plan file is intended to be human-readable and human-editable — you can hand-edit it before approving.

`.plan/` is gitignored.

---

## A note on `hermes run devops-agent`

Earlier design drafts in `ai/PROJECT.md` and `ai/user/PROJECT.md` describe the invocation as `hermes run devops-agent`. Hermes' public CLI doesn't expose a `run <bundle>` command — bundles are aliases for sets of already-installed skills, and skills are invoked inside a `hermes` (or `hermes --tui`) session as slash commands. The flow above (`hermes`, then `/eks-upgrade-plan ...`) is what actually works.

---

## Limitations (PoC)

- Single AWS account / single cluster per invocation.
- EKS control-plane upgrades cannot be rolled back — this is an AWS constraint, not a skill limitation.
- Node-group rolls can stall behind blocking PodDisruptionBudgets; the skill surfaces this but doesn't auto-resolve it.
- Trimmed readiness check set (7 categories: VER, API, NOD, WRK, PDB, HLM, ADD). The fuller catalogue (WEB, CRD, STR, PSP, IAM) lives in `cc-skill-migration/requirements.md` and can be ported in later.
- The approval gate is filesystem-based — anyone who can write the plan file can bypass it. Acceptable for a PoC; production use would want a stronger signal.

---

## Layout

```
devops-agent/
├── README.md                                 (this file)
├── ai/                                       (design docs — PROJECT.md, TECH.md, user/PROJECT.md)
├── cc-skill-migration/                       (original Claude Code skill, kept as reference)
└── skills/
    └── eks/
        ├── eks-upgrade-plan/SKILL.md
        └── eks-upgrade-execute/SKILL.md
```
