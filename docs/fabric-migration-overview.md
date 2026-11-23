# Overview: New Gold Layer and Semantic Models on Microsoft Fabric

_Read this first. The design is described in the [architecture overview](fabric-architecture-overview.md) and the [solution architecture](fabric-solution-architecture.md). The step-by-step procedures, checklists and scripts are in the [migration runbook](fabric-capacity-migration-plan.md)._

| | |
|---|---|
| **Document type** | Overview (companion to the architecture and the migration runbook) |
| **Version** | 2.0 (split-capacity design) |
| **Status** | Final draft for review |
| **Last updated** | 2026-11-23 |
| **Owner** | Data Services, Enterprise Technology |
| **Audience** | Approvers, BI team, capacity admins (A and B), business owners (BG1, BG2), Slalom |

---

## 1. Executive summary

Our Fabric capacity **A** runs both production and non-production workloads and is **being throttled**. Data volume keeps growing, and our semantic models are getting slower and using more compute. Slalom is rebuilding the **gold layer** and the **semantic models** so they cost less to run.

The design splits the work across two capacities:

- **The new silver-to-gold processing runs on capacity B**, in its own dev, test and prod workspaces (`transf-v2`). This takes the heaviest Spark work off A.
- **Orchestration and all semantic models, old and new, stay on capacity A.** Report users see no change of location.
- **The new models read the new gold through shortcuts on A.** If B is ever unavailable, reports keep working on the last loaded data.

This overview explains how we bring the work into production without disrupting the business:

- **We build and prove the work first**, before any report depends on it.
- **We promote it through one release path**: Azure DevOps → dev → test → prod, using Fabric deployment pipelines started from Azure DevOps. There's no shortcut route to prod.
- **We move reports in waves.** For a while, the old and new versions run side by side. Reports are **rebound**, not replaced, so users keep the same links, apps and subscriptions.
- **Every step can be undone.** Each wave has a Go/No-Go decision, a rehearsed rollback and a hypercare period.

**What we need now:** approval of this approach, named owners for each role (including a capacity admin for B), and decisions on the items in §10.

## 2. Problem statement

| Symptom | Impact |
|---|---|
| Semantic models getting slower as data grows | Reports load slowly; refreshes take longer |
| Rising compute (CU) use on capacity A | **Throttling**, which slows reports for everyone, including during business-critical periods |
| Heavy Spark transformation and the semantic layer sharing one capacity | Gold processing competes with report queries and refreshes |

Buying more capacity alone would only put off the problem. The lasting fix is to make the **gold layer and models more efficient** and to run the heavy transformation on a separate capacity.

## 3. Objectives and success measures

| Objective | How we measure it | Target |
|---|---|---|
| Reduce load on capacity A | CU use on A: gold Spark moved to B, plus model CU before vs after | No gold Spark on A after close-out; model reduction agreed in D-1 |
| Remove throttling for report users | Throttling events in the Capacity Metrics app (A) | No user-facing throttling after the final wave |
| Reliable new gold on B | New gold loaded by the agreed time | Freshness target met; alert on any miss |
| Keep the numbers correct | Reconciliation of new gold vs current gold; KPI cross-check | Exact match, or a documented and approved correction |
| No disruption for users | Report URLs, apps, bookmarks, subscriptions | Unchanged |
| Keep security intact | RLS test cases; access to prod on B | 100% pass; no developer access to prod on B |
| Controlled, auditable change | Every prod change has an Azure DevOps pull request, an approval and a release tag | 100% |

## 4. Scope

**In scope**

- New silver-to-gold notebooks and a new gold lakehouse on capacity B (`transf-v2` dev, test and prod)
- Changes to the elt orchestration pipeline on A, to start the new processing on B
- New semantic models for business groups 1 and 2 on A, reading new gold through shortcuts on A
- Rebinding the existing prod reports to the new models, in waves
- Retiring the old gold layer (on A) and the old models once nothing uses them
- Making capacity B production-ready (admins, monitoring, alerting, access)

**Out of scope**

- Changes to bronze/silver ingestion
- Creating new reports (these go through the normal release process)
- Moving the semantic layer or orchestration off capacity A

## 5. Current state and target state

| | **Current** | **During the migration** | **Target** |
|---|---|---|---|
| Capacity A | Prod and non-prod: orchestration, gold, models, reports | Orchestration; old gold; old and new models; reports | Orchestration; all models and reports; no gold processing |
| Capacity B | – | `transf-v2` dev/test/prod (new gold); Slalom development | `transf-v2` dev/test/prod (all gold processing) |
| Gold layer | Old gold on A | **Old gold (A) and new gold (B) running side by side** | New gold on B only |
| Semantic models (A) | Old models | Old models for reports not yet moved; new models for moved reports | New models only |
| Reports (A) | Bound to old models | Moving in waves | All bound to new models, with the same IDs and URLs |
| Release path | Git (dev) → deployment pipelines | Azure DevOps → deployment pipelines started from ADO, with approvals | Same as during the migration |

**Workspace layout:** on A, `elt`, `transf` (old gold, retiring), `cons-bg1` and `cons-bg2`. On B, the new **`transf-v2`**. Every layer has dev, test and prod.

## 6. Approach

### 6.1 How it fits together

```
 Capacity A  (orchestration + semantic layer)             Capacity B  (new gold processing)
 ┌───────────────────────────────────────────┐             ┌──────────────────────────────┐
 │ elt (master pipeline) ── starts ──────────┼───────────▶ │ transf-v2: new gold notebooks │
 │ silver ◀─────────── read via shortcut ────┼──────────── │   writes lh_gold_v2          │
 │ cons-bg1/bg2: lh_gold_v2_ref ◀ shortcuts ─┼──────────── │                              │
 │   new + old models, reports               │             │ dev / test / prod            │
 └───────────────────────────────────────────┘             └──────────────────────────────┘
   Release: Azure DevOps ─▶ dev ─▶ test ─▶ prod (deployment pipelines started from ADO)
```

### 6.2 Phases and gates

Each phase ends with a **gate**. The next phase doesn't start until the gate's evidence has been checked and recorded.

| Phase | What happens | Gate (exit criteria, summarised) |
|---|---|---|
| **0. Readiness** | Confirm assumptions (including **A and B in the same region**); capture baselines; set up Azure DevOps, the `transf-v2` workspaces on B, variable libraries, run identity, and B's operations | Baselines stored; tooling and B in place; key decisions made |
| **1. Build and prove** | In dev on B, run new gold against prod silver for ≥ 3 cycles; reconcile; load-test the new models; close field-name gaps | Numbers reconcile; **new models use fewer CUs**; no broken visuals |
| **2. Merge and test** | Pull requests in Azure DevOps; deploy to test (transf-v2 on B, elt and cons on A); UAT; rehearse the rollback | UAT signed off; rollback rehearsed; all references point at test |
| **3. New gold in prod, no reports reading it** | Deploy `transf-v2-prod` on B; run it from elt on A; reconcile against old gold | Prod reconciliation passed; B's failure path proven; CU budget approved |
| **4–5. Waves** (repeated) | Move one wave of reports to the new models; hypercare; switch off old items nothing uses any more | Go/No-Go per wave; hypercare clean; business sign-off |
| **6. Close-out** | Remove the rest of the old stack on A; bring environments in sync | Old stack gone; gold processing only on B |

### 6.3 Running both stacks side by side (waves)

The new gold and models **won't serve every report at first**. During the migration:

- **Wave 1 is a pilot** of 1–3 low-risk reports. Later waves are grouped by subject area, **highest CU savings first**.
- **Reports not yet moved stay on the old models**, which keep refreshing from the old gold on A.
- **The same KPI must give the same number in both stacks.** An automated cross-check runs after every load.
- **A dependency map** shows when each old table, notebook or model has nothing left using it. It's then switched off, and deleted after its rollback period. A's load falls with every wave.
- **A CU budget** caps the load on A while both run, and is reviewed weekly. Running both stacks has a **planned end date** (decision D-7).

## 7. Tools

| Need | Tool |
|---|---|
| Version control and code review | **Azure DevOps Repos**: one repo, one folder per workspace (including `transf-v2`); branch policies; required reviewers; automated checks for hard-coded values |
| Slalom's development | `transf-v2-dev` and **branched workspaces per target folder** on capacity B |
| Moving to test and prod | **Fabric deployment pipelines, one per layer, started from an Azure DevOps pipeline** with approvals; every prod release is tagged |
| Per-environment settings | **Fabric variable libraries** with dev, test and prod values, including IDs and shortcut targets that cross capacities |
| Checks after each deploy | Automated in the ADO pipeline: references across capacities, **which lakehouse each Direct Lake model reads from**, shortcut targets, report bindings |
| Inventory and lineage | Admin scanner API (through semantic-link-labs) |
| Monitoring and alerts | Capacity Metrics app (**A and B**), monitoring hub, Activator alerts, new gold freshness alert |

**Fabric pitfalls this plan handles explicitly** (details in runbook §5b):

- **Direct Lake models don't repoint to the right lakehouse when deployed** → stage-specific rules, plus a hard check after every deployment.
- **References to items in other workspaces never repoint automatically**: elt on A calling notebooks on B, shortcuts from B to silver, shortcuts from A to new gold → IDs and targets come from variable libraries.
- **Deployments and Git updates can delete shortcuts** in the target lakehouse → no manual shortcuts in test or prod; changes reviewed before every deploy.
- **Deployment pipelines pair items by name** → new items use `_v2` names, and renaming is banned during the migration.

## 8. How we keep it safe

| Control | What it means |
|---|---|
| **Gates** | Evidence-based exit criteria at the end of every phase |
| **Go/No-Go** | A checklist before every wave; a single "No" stops that wave |
| **Rollback** | Per wave: rebind the wave's reports to the old models (still refreshing from the old gold on A). Rehearsed in test, with timings. The same change is committed in Azure DevOps so the next deployment doesn't undo it |
| **Reports keep working if B has a problem** | New models read new gold through shortcuts on A; reports keep the last loaded data; the old branch is unaffected; a freshness alert fires |
| **Immediate rollback rule** | Any RLS/security defect triggers rollback, with no attempt to fix it in place |
| **Disable before delete** | Old items are switched off first and deleted only after their rollback period |
| **Protecting prod on B** | B is never paused; surge protection; dev/test Spark capped; no developer access to `transf-v2-prod` |
| **Separation of duties** | Slalom builds and opens pull requests; the internal BI team reviews, approves and deploys |
| **Audit trail** | Pull request, approval and release tag for every prod change |

## 9. Governance and roles

| Role | Responsibilities | Named owner (backup) |
|---|---|---|
| **Data Services** (plan owner) | Owns the plan; chairs Go/No-Go; accountable for delivery | ______ (______) |
| **BI team** | Reviews and merges, deploys (A and B), runs checks, runs rollbacks, retires old items; on-call for new gold | ______ (______) |
| **Capacity admin – A** | A's settings, CU monitoring, CU budget | ______ (______) |
| **Capacity admin – B** | B's settings and size, Spark pools, surge protection, monitoring; B is never paused | ______ (______) |
| **Business owner – BG1** | UAT, wave sign-off, user communication | ______ |
| **Business owner – BG2** | UAT, wave sign-off, user communication | ______ |
| **Slalom** | Builds and validates in dev on B; opens pull requests; supports hypercare until handover | ______ |

**Decision rights:** Data Services makes the Go/No-Go call. Each business owner must agree for waves that affect their reports. Escalation goes to the approvers listed in the runbook.

**Reporting:** a weekly status covering wave progress, CU on A and B against budget, new gold freshness, KPI cross-check and the end-date forecast. A daily status during hypercare.

## 10. Decisions needed

| # | Decision | Owner | Needed by |
|---|---|---|---|
| D-1 | Target CU reduction for the new models | Data Services + capacity admin A | Before Phase 1 sign-off |
| D-2 | Reconciliation tolerances (default: exact match) | Business owners | Before Phase 1 sign-off |
| D-7 | End date for running both stacks, and the number of waves | Approvers | Before Wave 1 |
| D-8 | CU budget on A while both stacks run | Capacity admin A + Data Services | Before Wave 1 |
| D-10 | Which reports go in the pilot wave | BI team + business owners | Phase 0 |
| D-11 | Promotion route: deployment pipelines started from ADO (default), or `fabric-cicd` | Data Services + BI team | Phase 0 |
| D-12 | How notebooks find their default lakehouse in each environment | BI team + Slalom | Phase 0 |
| D-13 | B's SKU size, and whether to turn on Spark autoscale billing on B | Capacity admin B + Data Services | Phase 0 |
| D-14 | Run identity for scheduled runs: service principal or workspace identity | BI team + capacity admins | Phase 0 |
| D-15 | Agreed new gold freshness time (load deadline) | BI team + business owners | Before Phase 3 |

The full list is in runbook §19.

## 11. Top risks

| Risk | Mitigation |
|---|---|
| **B unavailable or throttled during the load window** | Shortcuts on A keep reports working on the last loaded data; freshness alert; old branch unaffected; rerun |
| **Dev/test work on B slows the prod gold load** | Capped dev/test Spark pools; prod load window kept clear; surge protection; autoscale billing option; dev/test can move to a separate capacity |
| A **Direct Lake model in prod reads the test lakehouse** after deployment | Stage-specific binding rules; hard check after every deploy |
| **The same KPI shows different numbers** in old and new reports during the migration | KPI cross-check after every load; intended corrections told to users in advance |
| **The run identity lacks access on one capacity** | Service principal or workspace identity with documented access on A and B; tested before go-live |
| **A rollback is undone** by the next deployment | Rollback is also committed in Azure DevOps; the cons pipeline is frozen until then |

The full register is in runbook §16.

## 12. Indicative milestones

Dates are set at Phase 0 sign-off.

| Milestone | Marks |
|---|---|
| M0 – Plan approved | Runbook approved; owners named (including B's capacity admin) |
| M1 – Readiness complete | Gate G0: baselines, Azure DevOps, `transf-v2` workspaces and B operations in place |
| M2 – Proven in dev | Gate G1: reconciliation and CU gate passed |
| M3 – Ready in test | Gate G2: UAT signed off; rollback rehearsed |
| M4 – New gold live in prod on B | Gate G3: new gold running in prod, no reports reading it yet |
| M5 – Pilot wave live | First reports on the new models; hypercare starts |
| M6 – Final wave live | All reports on the new models |
| M7 – Close-out | Old stack retired from A; gold processing only on B |

## 13. What we need from you

| Group | Ask |
|---|---|
| **Approvers** | Approve this approach; confirm owners; make decisions D-7, D-8 and D-13 |
| **BI team** | Set up Azure DevOps policies and environments, the `transf-v2` deployment pipeline, variable libraries, the run identity and the post-deploy checks; own reviews, deployments and on-call |
| **Capacity admins** | A: CU baseline and budget, surge protection. B: confirm region and size, Spark pools, surge protection, monitoring; keep B running |
| **Business owners** | Confirm tolerances (D-2) and the freshness time (D-15); pick the pilot reports (D-10); provide UAT testers and sign-off |
| **Slalom** | Work in `transf-v2-dev` and branched workspaces per folder; follow the tooling standards (runbook §5b); provide the validation evidence |

## 14. What report users will notice

Nothing should change except a possible **data correction**. Reports stay in the same workspaces and apps on capacity A. Where Slalom's fixes change a number on purpose, users of the affected reports are told before their wave goes live. Users get a notice **5 business days** before their wave, and an update when it is complete.

## 15. Glossary

| Term | Meaning |
|---|---|
| **Capacity (A, B)** | The pool of Fabric compute (CUs) that workspaces run on. A = orchestration, semantic layer, old gold; B = new gold processing (dev, test and prod) |
| **CU** | Capacity unit, the measure of Fabric compute use |
| **Throttling** | Fabric slowing or rejecting work when a capacity is over its limit |
| **Gold layer** | Curated tables that semantic models and reports use |
| **`transf-v2`** | The new silver-to-gold workspaces on capacity B |
| **`lh_gold_v2_ref`** | A lakehouse on A holding shortcuts to the new gold on B; the new models read from it |
| **Shortcut** | A OneLake pointer to data stored elsewhere, without copying it |
| **Semantic model** | The Power BI model (tables, relationships, measures, RLS) that reports query |
| **Direct Lake** | A semantic model mode that reads Delta tables in OneLake directly |
| **Rebind** | Pointing an existing report at a different semantic model, keeping its ID and URL |
| **Wave** | A group of reports moved to the new models together |
| **Deployment pipeline** | Fabric's tool for promoting items from dev to test to prod |
| **Variable library** | A Fabric item that holds environment-specific values (dev, test and prod) |
| **Hypercare** | A period of close monitoring after a wave goes live |
| **Gate** | Evidence-based exit criteria that must be met before the next phase |

## 16. Related documents

| Document | Purpose |
|---|---|
| [Architecture overview](fabric-architecture-overview.md) | 3-page summary of the design |
| [Solution architecture](fabric-solution-architecture.md) | How the solution is built: capacities, workspaces, data flow, semantic layer, environments, security, operations, design decisions |
| [Cutover plan](fabric-cutover-plan.md) | Simple step-by-step cutover, including backfill, Go/No-Go and rollback |
| [Migration runbook](fabric-capacity-migration-plan.md) | Full procedures, checklists, rollback, risk register, scripts |
| Runbook §5b | Fabric and Azure DevOps tooling standards |
| Runbook §14–§16 | Go/No-Go, rollback, risk register |

| If you are… | Read |
|---|---|
| An approver | Architecture overview; runbook §14 Go/No-Go, §15 Rollback, §16 Risks, §19 Decisions |
| On the BI team or with Slalom | Solution architecture; runbook §5b Tooling standards, Phases 0–6, Appendix A (scripts) |
| A capacity admin | Solution architecture §4 and §12; runbook Phase 0 |
| A business owner | Runbook §5a.1 Wave plan, §17 Communication, §18 Acceptance criteria |
