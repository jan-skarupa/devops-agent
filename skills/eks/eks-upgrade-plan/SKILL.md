---
name: eks-upgrade-plan
description: Inspects an EKS cluster, runs a PoC readiness check, and writes an approvable Kubernetes minor-version upgrade plan to ./.plan/eks-upgrade/<cluster>-<target>.md. Invoke when the user runs /eks-upgrade-plan <region> <cluster> [--target <x.y>], or when they reply APPROVE / requests changes after a plan has been generated in this session.
platforms: [linux, macos]
metadata:
  hermes:
    tags: [aws, eks, kubernetes, upgrade, devops]
    requires_tools: [aws, kubectl, helm]
    related_skills: [eks-upgrade-execute]
---

## When to Use

Run this skill when the user wants to plan an EKS minor-version upgrade — typically as:

```
/eks-upgrade-plan <region> <cluster> [--target <x.y>]
```

Also run this skill (continuing the same session) when, immediately after the plan has been written, the user replies:
- `APPROVE` (case-insensitive) — mark the plan as approved.
- Anything else (e.g. "drop step 3", "add a coredns upgrade") — regenerate the plan with their feedback.

This skill is **read-only against the cluster**. It never mutates anything. The companion skill `/eks-upgrade-execute` performs the actual upgrade.

## Quick Reference

- Tools used: Hermes `terminal` to shell out to `aws`, `kubectl`, `helm`.
- Output: a markdown plan file at `./.plan/eks-upgrade/<cluster>-<target>.md` plus an inline summary.
- Approval gate: the plan's frontmatter has `approved: false` until the user types APPROVE.

## Procedure

### Step 1 — Parse arguments

The user message after `/eks-upgrade-plan` is free text. Extract:
- `region` — first AWS region token (e.g. `us-east-1`, `eu-west-2`).
- `cluster` — the cluster name (the next non-flag token).
- `--target <x.y>` — optional. If absent, default to `current_minor + 1` after Step 3.

If `region` or `cluster` is missing, ask the user for them and stop.

### Step 2 — Connect kubeconfig

Run:
```
aws eks update-kubeconfig --region <region> --name <cluster> --alias <cluster>
kubectl config use-context <cluster>
```

If `aws eks update-kubeconfig` fails (no creds / no such cluster), report the error verbatim and stop. Do not fall back silently.

Confirm connectivity:
```
kubectl get --raw /readyz
kubectl get nodes -o name | head -1
```

### Step 3 — Discover versions

Current control-plane version:
```
aws eks describe-cluster --region <region> --name <cluster> --query 'cluster.version' --output text
```

Target version:
- If `--target` was provided, validate it is exactly one minor above current; if not, FAIL and tell the user EKS only supports single-minor jumps.
- Otherwise, default `target = current_minor + 1` (e.g. `1.33` → `1.34`).

Also fetch supported versions for sanity-check:
```
aws eks describe-addon-versions --region <region> --kubernetes-version <target> --query 'addons[0].addonVersions[0].compatibilities[0].clusterVersion' --output text
```
A non-empty answer is evidence the target is supported.

### Step 4 — Run PoC readiness checks

Trim to the PoC essentials — about 60 % of the full cc-skill catalogue. Each check produces one of: `PASS`, `WARN`, `FAIL`. Collect them; you'll render a single status (`READY` / `READY WITH WARNINGS` / `NOT READY`) in Step 5.

**VER — Version & upgrade path**
| ID | Check | Tool / criterion |
|---|---|---|
| VER-01 | Single-minor step | computed in Step 3 |
| VER-02 | Target supported by EKS | `aws eks describe-addon-versions` returned a compatible version |
| VER-03 | Current version EOS awareness | informational — print a link to https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html |

**API — Deprecated / removed APIs**
Cumulative removal schedule (check every minor between current+1 and target):
| Minor | Removed |
|---|---|
| 1.22 | `extensions/v1beta1`: Ingress, NetworkPolicy, DaemonSet, Deployment, ReplicaSet, StatefulSet; `networking.k8s.io/v1beta1`: Ingress, IngressClass; `rbac.authorization.k8s.io/v1beta1`+`v1alpha1`; `admissionregistration.k8s.io/v1beta1`; `apiextensions.k8s.io/v1beta1`: CRD; `authentication.k8s.io/v1beta1`, `authorization.k8s.io/v1beta1`; `scheduling.k8s.io/v1beta1`+`v1alpha1`: PriorityClass |
| 1.25 | `policy/v1beta1`: PodSecurityPolicy, PodDisruptionBudget; `batch/v1beta1`: CronJob; `autoscaling/v2beta1`: HPA; `discovery.k8s.io/v1beta1`: EndpointSlice; `events.k8s.io/v1beta1`: Event; `node.k8s.io/v1beta1`: RuntimeClass; `storage.k8s.io/v1beta1`: StorageClass, VolumeAttachment, CSINode, CSIDriver |
| 1.26 | `autoscaling/v2beta2`: HPA; `flowcontrol.apiserver.k8s.io/v1beta1` |
| 1.27 | `storage.k8s.io/v1beta1`: CSIStorageCapacity |
| 1.29 | `flowcontrol.apiserver.k8s.io/v1beta2` |
| 1.32 | `flowcontrol.apiserver.k8s.io/v1beta3` |

Detection:
- If helm is available, for each release: `helm get manifest <release> -n <ns>` and grep for the removed `apiVersion:` strings.
- Optionally: `kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis` to spot live use of deprecated APIs.

FAIL on any in-use removed API. WARN if you can't conclusively determine (e.g. helm not present and no metrics endpoint).

**NOD — Node health**
```
kubectl get nodes -o json
```
| ID | Criterion |
|---|---|
| NOD-01 | Every node has `Ready=True` — else FAIL, list offending nodes |
| NOD-02 | No node has `MemoryPressure / DiskPressure / PIDPressure = True` — else WARN |
| NOD-03 | No node has `.spec.unschedulable: true` — else WARN |
| NOD-04 | Every `.status.nodeInfo.kubeletVersion` is within 1 minor of control-plane — 2 minor = WARN, 3+ = FAIL |

**WRK — Workload health**
```
kubectl get deploy,sts,ds -A -o json
kubectl get pods -n kube-system -o json
```
| ID | Criterion |
|---|---|
| WRK-01 | Deployments: `.status.availableReplicas >= .spec.replicas` — else FAIL |
| WRK-02 | StatefulSets: `.status.readyReplicas >= .spec.replicas` — else FAIL |
| WRK-03 | DaemonSets: `.status.numberReady == .status.desiredNumberScheduled` — else FAIL |
| WRK-04 | No CrashLoopBackOff pods in `kube-system` — else FAIL |

**PDB — Pod disruption budgets**
```
kubectl get pdb -A -o json
```
| ID | Criterion |
|---|---|
| PDB-01 | `status.disruptionsAllowed == 0` — WARN (currently blocking drain) |
| PDB-02 | `spec.maxUnavailable == 0` — FAIL (will always block drain) |

**HLM — Helm releases**
```
helm list -A -o json
```
| ID | Criterion |
|---|---|
| HLM-01 | No releases in `failed` or `pending-*` status — else WARN |
| HLM-02 | Manifest scan from API check above — re-use those findings |

**ADD — EKS add-ons**
```
aws eks list-addons --region <region> --cluster-name <cluster>
aws eks describe-addon --region <region> --cluster-name <cluster> --addon-name <each>
aws eks describe-addon-versions --region <region> --kubernetes-version <target> --addon-name <each>
```
For each installed add-on (typically `vpc-cni`, `coredns`, `kube-proxy`, and possibly `aws-ebs-csi-driver`), check that the currently installed version appears in the target version's compatibility list. If not, flag WARN with the recommended target add-on version. Also scan `helm list -A` for `aws-load-balancer-controller` and flag if its version is older than the latest minor compatible with the target (informational WARN — exact compatibility version is operator's call).

Print live progress between categories using simple section headers, so the user sees the skill is working:
```
[VER]  ...
[API]  ...
[NOD]  ...
[WRK]  ...
[PDB]  ...
[HLM]  ...
[ADD]  ...
```

### Step 5 — Synthesize the execution plan

Build a numbered list of steps **in execution order**. The right order for a typical EKS minor upgrade:

1. Upgrade incompatible add-ons / controllers BEFORE the control plane (most commonly `aws-load-balancer-controller`, `cert-manager`, anything using removed APIs).
2. Upgrade EKS managed add-ons that have target-compatible versions (`vpc-cni`, `coredns`, `kube-proxy`).
3. Upgrade EKS control plane.
4. Upgrade managed node groups (one at a time).
5. Verify cluster health.

Each step is an object with: `title`, `type` (`mutating` | `read-only`), `risk` (`low` | `medium` | `high`), `command` (the exact CLI line), `rollback` (text or `not supported by EKS`).

Skip the add-on steps if no incompatibilities were found. Always include the control-plane bump, the node-group bumps (one entry per node group), and the final verify step.

Discover managed node groups:
```
aws eks list-nodegroups --region <region> --cluster-name <cluster>
```

Typical command lines to use as the `command:` field:
- Helm chart upgrade:
  `helm upgrade <release> <chart> -n <ns> --version <new-chart-version> --reuse-values`
  Rollback: `helm rollback <release> <prev-rev> -n <ns>`
- EKS managed add-on:
  `aws eks update-addon --region <region> --cluster-name <cluster> --addon-name <name> --addon-version <ver> --resolve-conflicts PRESERVE`
  Rollback: `aws eks update-addon ... --addon-version <prev-ver>`
- EKS control plane:
  `aws eks update-cluster-version --region <region> --name <cluster> --kubernetes-version <target>` followed by polling `aws eks describe-cluster ... --query 'cluster.status'` until `ACTIVE`.
  Rollback: not supported by EKS.
- Managed node group:
  `aws eks update-nodegroup-version --region <region> --cluster-name <cluster> --nodegroup-name <ng>` followed by polling.
  Rollback: create/restore the previous launch-template version of the node group (manual).
- Final verify (read-only):
  `kubectl get nodes -o wide && kubectl get pods -A | grep -vE 'Running|Completed'`

### Step 6 — Write the plan file

Path: `./.plan/eks-upgrade/<cluster>-<target>.md` (relative to the Hermes session cwd). Create the directory if it does not exist (`mkdir -p`).

Format (copy this template exactly — `eks-upgrade-execute` parses it):

```markdown
---
cluster: <cluster>
region: <region>
current_version: "<x.y>"
target_version: "<x.y>"
approved: false
generated_at: <ISO 8601 timestamp>
---

# Preflight summary

Status: READY | READY WITH WARNINGS | NOT READY

Blocking issues:
- <one line per FAIL, or "None">

Warnings:
- <one line per WARN, or "None">

# Proposed execution plan

[1] <Title>
    Type: mutating | read-only
    Risk: low | medium | high
    Command: <exact CLI command>
    Rollback: <command, or "not supported by EKS">

[2] ...
```

Status logic:
- Any FAIL → `NOT READY`.
- Zero FAIL, ≥1 WARN → `READY WITH WARNINGS`.
- Zero FAIL, zero WARN → `READY`.

If `NOT READY`, **still write the plan file** — but emit only the verify step and prepend a top step labelled `[0] BLOCKED — resolve issues above before APPROVE`. This keeps the file shape stable for the execute skill.

### Step 7 — Print the summary and prompt for approval

Print to the chat (not just the file):

```
Plan written: ./.plan/eks-upgrade/<cluster>-<target>.md

Preflight: <status>
Blocking issues: <count>
Warnings: <count>
Steps: <count>

Reply APPROVE to mark this plan executable, or describe changes you'd like (e.g. "drop step 3").
If there are blockers, fix them and re-run /eks-upgrade-plan to refresh.
```

### Step 8 — Handle the follow-up turn

After the plan has been written, the next user message is one of:

- **APPROVE** (case-insensitive, possibly with surrounding whitespace) — refuse if status is `NOT READY`; otherwise edit the plan file's frontmatter `approved: false` → `approved: true` and confirm: `Plan approved. Run /eks-upgrade-execute <region> <cluster> to apply.`
- **Change request** — re-enter Step 4 with the feedback applied (e.g. skip a category, add a step, change a command). Write a fresh file (overwrite) and re-prompt.
- **Anything else** — ask whether they want to approve, change, or abandon.

## Pitfalls

- EKS only supports single-minor upgrades. Reject `--target` that skips a minor.
- The deprecated-API check must cover **every** intermediate minor (current+1 … target), not just the target.
- `helm get manifest` only returns what helm tracks. Resources applied with `kubectl apply` directly will be missed by HLM but caught by the live `apiserver_requested_deprecated_apis` metric — surface that limitation in the WARN text when relevant.
- `aws eks update-kubeconfig --alias <cluster>` overwrites any pre-existing context with that name. Acceptable for the PoC; warn the user if their kubeconfig had a custom context for this cluster.
- Control-plane upgrades cannot be rolled back. Record this verbatim in the plan's Rollback field so the execute skill displays it before mutating.

## Verification

1. Against a real cluster: `/eks-upgrade-plan us-east-1 my-cluster` should produce `./.plan/eks-upgrade/my-cluster-<next>.md` with valid YAML frontmatter and at least one step.
2. Edge: missing AWS creds — Step 2 should fail loudly with the underlying error.
3. Edge: target two minors above current — Step 3 should reject.
4. APPROVE turn — re-read the file; the frontmatter line should now be `approved: true`.
5. Change-request turn — file is overwritten and `approved: false` again.
