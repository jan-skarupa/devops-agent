# DevOps Agent

## Overview
Acts as a DevOps engineer for AWS — performs a defined set of operational tasks against a single AWS account on the user's behalf, with the human in the loop for review and approval. First iteration focuses on EKS Kubernetes cluster upgrades.

## Core Functionality
- Inspects an EKS cluster and produces an upgrade readiness report listing blocking issues, warnings, and a proposed step-by-step plan
- Saves an approved upgrade plan to disk, keyed by cluster name and target version
- Executes an approved plan to upgrade the cluster (control plane, node groups, dependent add-ons), with rollback steps where supported

## Target Users
DevOps engineers running EKS who want to offload repetitive upgrade work. As the agent matures, small teams looking to outsource routine ops. Single-operator scale per invocation — one human, one cluster at a time.

## Non-Functional Requirements
- Mutating actions require explicit user approval (saved plan must be APPROVED before execute)
- Operates against a single configured AWS account; minimum read-only IAM, optionally write/execute
- Plan and execute are separate, idempotent steps — user can fix blockers and re-run plan

## Glossary
- **Upgrade plan**: a structured, approvable markdown document listing ordered steps, type (mutating/read-only), risk level, and rollback strategy for a specific cluster + target version
- **Skill**: a discrete agent capability invoked by name (e.g. plan vs execute) — keeps each task narrowly scoped
