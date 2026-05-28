---
name: eks-upgrade-execute
description: Executes an APPROVED EKS upgrade plan written by /eks-upgrade-plan. Reads the plan at ./.plan/eks-upgrade/<cluster>-<target>.md, refuses if not approved, then walks the steps in order — pausing for explicit user confirmation before every mutating step. Invoke when the user runs /eks-upgrade-execute <region> <cluster> [--target <x.y>].
platforms: [linux, macos]
metadata:
  hermes:
    tags: [aws, eks, kubernetes, upgrade, devops]
    requires_tools: [aws, kubectl, helm]
    related_skills: [eks-upgrade-plan]
---

## When to Use

Run this skill when the user wants to apply an upgrade plan that was previously generated and APPROVED by `/eks-upgrade-plan`:

```
/eks-upgrade-execute <region> <cluster> [--target <x.y>]
```

This skill **mutates the cluster**. It calls real `aws eks update-cluster-version`, `aws eks update-nodegroup-version`, `helm upgrade`, etc. It pauses before every mutating step and asks the user to confirm.

If the plan does not exist or is not approved, this skill refuses and tells the user to run `/eks-upgrade-plan` first.

## Quick Reference

- Tool used: Hermes `terminal` (no MCP).
- Input: the plan file at `./.plan/eks-upgrade/<cluster>-<target>.md`.
- Gate: refuses unless the plan's frontmatter has `approved: true`.
- Per-step confirmation: every step with `Type: mutating` requires a `yes` from the user before running.

## Procedure

### Step 1 — Parse arguments

From the user's message after `/eks-upgrade-execute`:
- `region`, `cluster` — required.
- `--target <x.y>` — optional.

If region or cluster is missing, ask and stop.

### Step 2 — Locate and load the plan

Resolve `./.plan/eks-upgrade/<cluster>-<target>.md` (cwd-relative).

- If `--target` is given: read that exact file.
- If not: list `./.plan/eks-upgrade/<cluster>-*.md`; if exactly one, use it; if multiple, ask the user which target; if none, tell the user to run `/eks-upgrade-plan` first and stop.

Read the file. Parse the YAML frontmatter (`cluster`, `region`, `current_version`, `target_version`, `approved`, `generated_at`).

Sanity-check: `cluster` and `region` in the frontmatter must match the args. If they don't, refuse and report the mismatch — the user is pointing at the wrong cluster.

### Step 3 — Refuse if not approved

If frontmatter `approved` is not exactly `true`:
```
Plan at <path> is not approved (approved: <value>).
Run /eks-upgrade-plan <region> <cluster> [--target <x.y>] and reply APPROVE first.
```
Stop. This is the only protective gate — once `approved: true`, the skill proceeds.

Also refuse and stop if step `[0]` is the special `BLOCKED — ...` placeholder (NOT READY plans).

### Step 4 — Re-attach kubeconfig

Idempotent — safe to run even if already attached:
```
aws eks update-kubeconfig --region <region> --name <cluster> --alias <cluster>
kubectl config use-context <cluster>
kubectl get --raw /readyz
```
If any of these fails, abort the run before mutating anything.

### Step 5 — Parse the steps

The plan body has step blocks of the form:
```
[N] <Title>
    Type: mutating | read-only
    Risk: low | medium | high
    Command: <CLI command>
    Rollback: <command, or "not supported by EKS">
```

Parse all blocks into an in-memory list. Preserve order.

### Step 6 — Walk the steps

For each step, in order:

1. **Announce the step**:
   ```
   ─── Step [N]: <Title> ───
   Type: <type>   Risk: <risk>
   Command: <command>
   Rollback: <rollback>
   ```

2. **If `Type: read-only`** — run the command immediately via `terminal`. Print exit code and a short output tail (last ~30 lines). On non-zero exit, **stop** and report.

3. **If `Type: mutating`** — pause and ask the user:
   ```
   Proceed with step [N]? (yes / no / skip)
     yes  — run the command now
     no   — abort the upgrade here
     skip — record skipped and continue to the next step (use only if you know what you're doing)
   ```
   - `yes` → run the command via `terminal`. Stream/print output. After the command returns, capture exit code.
   - `no` → print a summary of completed steps and stop.
   - `skip` → record skipped and continue.

4. **For long-running EKS operations** — the `update-cluster-version` and `update-nodegroup-version` calls return immediately with an update ID. After running them, poll status until terminal:
   ```
   aws eks describe-cluster --region <region> --name <cluster> --query 'cluster.status' --output text
   aws eks describe-nodegroup --region <region> --cluster-name <cluster> --nodegroup-name <ng> --query 'nodegroup.status' --output text
   ```
   Poll every 60 seconds. Control plane upgrades typically take 10–30 minutes; node groups 10–60 minutes per group. Show the user the polling timestamps so they know the skill is alive. If the status reaches `ACTIVE`, mark success; if `DEGRADED` / `CREATE_FAILED` / `UPDATE_FAILED`, mark failure.

5. **On failure (non-zero exit or terminal failure status)**:
   - Print the failure cleanly.
   - Show the step's `Rollback:` line.
   - If rollback is `not supported by EKS` (control plane), explain that EKS does not allow downgrade — the user must work the issue forward (re-run the failed command, contact AWS support, or accept the new version and continue).
   - Otherwise, ask the user: `Attempt rollback? (yes / no)`. If yes, run the rollback command and report its exit code.
   - Either way, stop after handling the failure — do not silently continue to the next step.

### Step 7 — Final verification

After the last step completes (or the user stops early), print a summary:
```
═══════════════════════════════════════════════
Upgrade run summary for <cluster>
  Done : <count>      Skipped : <count>
  Failed : <count>    Remaining : <count>
═══════════════════════════════════════════════
```

Then run a final read-only check (regardless of how the run ended):
```
aws eks describe-cluster --region <region> --name <cluster> --query 'cluster.version' --output text
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
```
Show the result so the user can see the post-run state at a glance.

## Pitfalls

- **EKS control-plane upgrades are one-way.** Always show the user the Rollback field (which will say `not supported by EKS`) before running the control-plane step.
- **Node-group upgrades drain pods.** A blocking PDB will stall the drain forever. If a node group step hangs at `UPDATING` for > 30 minutes, suspect a PDB; offer to investigate.
- **`helm rollback` requires the previous revision's values.** If the user has run `helm history` cleanup, rollback can fail. Verify with `helm history <release> -n <ns>` before relying on it.
- **The plan file is the contract.** Do not invent or alter commands at runtime. If a step's command looks wrong, stop and ask the user to regenerate the plan.
- **`approved: true` is the only gate.** Anyone (including the user) who can write the file can bypass approval. This is acceptable for a PoC; document it.

## Verification

1. **Not approved** — point the skill at a plan with `approved: false`; it must refuse and exit cleanly.
2. **BLOCKED plan** — point it at a NOT READY plan; it must refuse.
3. **Per-step gate** — for an APPROVED plan with multiple steps, the skill must announce the first mutating step and wait for `yes` before running anything.
4. **Failure handling** — induce a step failure (e.g. point a helm step at a wrong release name); the skill should show the rollback line and stop, not continue.
5. **End-to-end** — apply the full plan against a non-production cluster; verify the control-plane and node-group versions match the target afterwards.
