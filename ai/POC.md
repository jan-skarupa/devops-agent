# Proof of Concept

## Goal
Validate that a Hermes bundle of two markdown skills can drive a real EKS minor-version upgrade end-to-end: produce a useful plan from live cluster state, gate it on APPROVE, then execute it — first in dry-run during development, then once for real against an existing throwaway cluster.

## Scope

**In:**
- Hermes bundle layout for this repo (config + the two skills, per agentskills.io)
- `/eks-upgrade-plan <region> <cluster>` skill — shells out to aws / kubectl / helm / eksctl, produces a markdown plan at `./.plan/eks-upgrade/<cluster>-<target>.md`
- APPROVE workflow — user types `APPROVE` in chat; skill marks the plan file as approved
- `/eks-upgrade-execute <region> <cluster>` skill with a dry-run mode that prints commands instead of running them
- One real run against the existing throwaway EKS cluster after dry-run looks correct
- Single target upgrade path: current → current+1 minor version

**Out (explicitly):**
- Helper Python scripts for preflight (deferred — LLM reasons over raw CLI output for v1)
- An EKS MCP server (shell-out only)
- Multi-cluster / multi-account support (single cluster, inherited AWS creds)
- Rollback automation beyond what the plan documents (PoC validates the plan + execute path, not failure recovery)
- Tests, CI, error-handling polish, packaging for redistribution
- Skills beyond the two EKS upgrade skills

## Implementation Notes
- Hardcoding is fine — this is not production code
- Minimal config, skip error handling, skip tests
- Speed over quality — make it work, nothing more
- Dry-run vs real execute: a single env var or skill argument is sufficient (no need for fancy mode plumbing)
- Plan file format: follow the example in `ai/user/PROJECT.md` (Preflight summary + numbered steps with type/risk/rollback)

## Steps
1. Install Hermes Agent (`uv` based) and confirm `hermes` CLI runs locally
2. Create the agent bundle skeleton in this repo (Hermes bundle config + `skills/` directory per agentskills.io)
3. Write `/eks-upgrade-plan` skill: prompt instructs LLM to gather state via `aws eks describe-cluster`, `kubectl version`, `kubectl get nodes`, `helm list -A`, etc., then emit a plan markdown file at `./.plan/eks-upgrade/<cluster>-<target>.md`
4. Wire the APPROVE flow: skill watches for `APPROVE` in chat and updates a status line at the top of the plan file
5. Write `/eks-upgrade-execute` skill: reads the approved plan, walks each step, runs the command — but in dry-run mode just prints what it would run
6. Run `hermes run devops-agent` against the throwaway cluster; iterate on `/eks-upgrade-plan` until the plan reads correctly
7. Approve the plan; run `/eks-upgrade-execute` in dry-run; verify the printed command sequence matches the plan
8. Flip dry-run off and run `/eks-upgrade-execute` for real once; observe the cluster reach the target minor version

## Success Criteria
- `hermes run devops-agent` loads the bundle and exposes both skills as slash commands
- `/eks-upgrade-plan` produces a plan file matching the format in `ai/user/PROJECT.md`, with at least the control-plane upgrade step and any node-group / add-on steps the live cluster needs
- The plan file is only marked APPROVED after the user types `APPROVE`; `/eks-upgrade-execute` refuses to proceed otherwise
- Dry-run execute prints the exact command sequence the plan describes, in order
- The final real run upgrades the throwaway cluster's control plane (and node group, if in the plan) to the target version with no manual intervention
