# Overview: New Gold Layer and Semantic Models

_Read this first (5 minutes). The step-by-step detail is in the [full runbook](fabric-capacity-migration-plan.md)._

**Status:** Draft for review · **Last updated:** 2026-11-12 · **Owner:** Data Services (Enterprise Technology)

---

## Why we're doing this

Data volume keeps growing, so our semantic models are getting slower and using more compute. Capacity **A**, which runs both prod and non-prod, is being **throttled**. Slalom is rebuilding the **gold layer** and the **semantic models** so they cost less to run.

## What changes, and what doesn't

| Changes | Stays the same |
|---|---|
| New silver-to-gold notebooks and a **new gold lakehouse** | Every workspace stays on **capacity A** |
| **New semantic models** for BG1 and BG2 | The workspace layout (elt, transf, cons-bg1, cons-bg2) |
| Existing reports are **rebound** to the new models, in waves | **Report URLs, apps, bookmarks and subscriptions** |
| The old gold and old models are retired once nothing uses them | The deployment pipelines and the release process |

**Capacity B** is a development area only. Slalom builds and tests there, and nothing in prod depends on it.

## How it works

```
 Capacity B (dev area)                 Capacity A (non-prod → prod)
 ┌──────────────────────┐   Git PR    ┌──────────────────────────────────────┐
 │ trans-dev-2 (Slalom) │ ──────────▶ │ dev ──▶ test ──▶ prod                 │
 │ build + heavy tests  │  (Azure     │  (deployment pipelines, started from │
 └──────────────────────┘   DevOps)   │   Azure DevOps with approvals)       │
                                      └──────────────────────────────────────┘
```

1. **Build and prove it on B.** Slalom runs the new gold against prod silver and checks it against the current gold for at least 3 cycles. They also confirm the new models **use fewer CUs** than the old ones. If they don't, nothing goes further.
2. **Merge through Azure DevOps into the existing non-prod workspaces on A.** Pull requests are reviewed and checked automatically. No separate route to prod. Environment values (workspace, lakehouse and model IDs) come from variable libraries, not hard-coded values.
3. **Run the new gold in prod with no reports reading it.** It runs next to the old gold, and the two are compared.
4. **Move reports in waves.** We start with a small pilot (1–3 reports), then move groups of reports by subject area. Each report is **rebound**, not replaced, so users keep the same links.
5. **Both stacks run side by side for a while.** Reports not yet moved stay on the old models. We check that the same KPI gives the same number in both stacks, and we watch A's CU usage against a budget each week.
6. **Retire old items as they become unused**, then close out once the last wave is done.

## Tools

| Need | Tool |
|---|---|
| Version control and code review | **Azure DevOps Repos**: one folder per workspace, branch policies, required reviewers, automated checks for hard-coded values |
| Slalom's development | One branched workspace **per target folder** on capacity B |
| Moving to test and prod | **Fabric deployment pipelines, started from an Azure DevOps pipeline** with approvals; every prod release is tagged |
| Per-environment settings | **Fabric variable libraries** (dev, test and prod values) |
| Checks after each deploy | Automated scripts in the ADO pipeline: references, **which lakehouse each Direct Lake model reads from**, report bindings |
| Inventory and lineage | Admin scanner API (through semantic-link-labs) |
| Monitoring | Capacity Metrics app, monitoring hub, Activator alerts |

## How we keep it safe

- **Every phase has a gate.** The next phase doesn't start until the evidence for the current one has been checked.
- **Each wave can be rolled back** by rebinding its reports to the old models, which keep refreshing while both stacks run. We rehearse the rollback in test before the first wave.
- **Old items are switched off first and deleted later**, only after their rollback period ends.
- **Go/No-Go before every wave.** A single "No" stops that wave.
- **An RLS/security defect means immediate rollback.** We don't try to fix it in place.
- **Fabric's known pitfalls are covered:** Direct Lake models and references to other workspaces don't repoint by themselves when deployed, and deployments can delete shortcuts. Each has a rule, and an automated check after every deployment.
- **A rollback is also made in Azure DevOps**, so the next deployment doesn't undo it.

## Who does what

| Who | Role |
|---|---|
| **Slalom** | Builds and validates on B; opens pull requests |
| **BI team** | Reviews and merges, deploys, checks results, runs rollbacks |
| **Data Services** | Owns the plan; makes the Go/No-Go call |
| **Platform / capacity admin** | Capacity settings, CU monitoring and budget |
| **Business owners (BG1, BG2)** | UAT, sign-off for each wave, user communication |

## Decisions we need

| # | Decision | Needed by |
|---|---|---|
| D-1 | Target CU reduction for the new models | Before validation sign-off |
| D-2 | Reconciliation tolerances (default: exact match) | Before validation sign-off |
| D-7 | End date for running both stacks and the number of waves | Before Wave 1 |
| D-8 | CU budget on A while both stacks run | Before Wave 1 |
| D-10 | Which reports go in the pilot wave | During readiness |
| D-11 | Promotion route: deployment pipelines started from ADO (default), or `fabric-cicd` | During readiness |
| D-12 | How notebooks find their default lakehouse in each environment | During readiness |

The full list is in [§19 of the runbook](fabric-capacity-migration-plan.md#19-open-decisions).

## What report users will notice

Nothing should change except a possible **data correction**. Where Slalom's fixes change a number on purpose, users of the affected reports are told **before** their wave goes live. Users get a notice 5 business days before their wave.

## Where to go next

| If you are… | Read |
|---|---|
| An approver | §14 Go/No-Go, §15 Rollback, §16 Risks, §19 Decisions |
| On the BI team or with Slalom | §5b Tooling standards, Phases 0–6 and Appendix A (scripts) |
| A business owner | §5a.1 Wave plan, §17 Communication, §18 Acceptance criteria |

➡ **[Open the full runbook](fabric-capacity-migration-plan.md)**
