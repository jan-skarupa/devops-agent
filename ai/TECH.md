# Tech Stack

## Stack

| Layer | Choice | Notes |
|---|---|---|
| Agent Runtime | Hermes Agent (nousresearch/hermes-agent) | Fixed by user; Python-based, slash-command skills, supports bash/MCP/Python RPC |
| Skill Format | Markdown prompts (agentskills.io standard) | Hermes convention; LLM-driven, no compiled code per skill |
| Bundle Language | Python (for any helper scripts / config tooling) | Matches Hermes ecosystem; user is expert |
| Package Mgmt | uv | Hermes installer uses it; standard for the runtime |
| AWS / K8s CLIs | aws, kubectl, helm, eksctl | Invoked from skills via Hermes' bash tool |
| Auth | Standard AWS credential chain (env / profile / IAM role) | No custom auth layer |
| Plan Storage | Markdown files on disk under `./.plan/eks-upgrade/<cluster>-<target>.md` | Human-readable; serves as the APPROVE gate between plan and execute |

## Architecture

- User invokes `hermes run devops-agent` from their shell — Hermes loads this repo as an agent bundle.
- Bundle exposes two skills as slash commands: `/eks-upgrade-plan` and `/eks-upgrade-execute`.
- Each skill is a markdown prompt that instructs the LLM to shell out (aws, kubectl, helm, eksctl), reason over the raw output, and produce/consume a plan file.
- `/eks-upgrade-plan` writes a markdown plan to `./.plan/eks-upgrade/<cluster>-<target>.md`; user types `APPROVE` in chat to mark it executable.
- `/eks-upgrade-execute` reads the approved plan and drives the upgrade step-by-step, calling rollback paths on failure where supported.
- AWS credentials are inherited from the invoking shell; no secrets in the bundle.

## Key Decisions

- **Shell-out over MCP server**: transparency for ops work, no extra infra, leverages tools the user already knows — picked over wrapping EKS operations in a typed MCP server.
- **LLM-driven preflight, no helper scripts in v1**: faster iteration; deterministic helpers can be added later for any check that proves fragile in practice.
- **eksctl alongside aws CLI**: aws CLI for control-plane operations; eksctl where it materially simplifies node-group / add-on upgrades.
- **Markdown plan files as the approval boundary**: human-readable artifact, decouples plan and execute steps, no database needed.
