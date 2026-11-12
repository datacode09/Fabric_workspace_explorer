# Overview: New Gold Layer and Semantic Models on Microsoft Fabric

_Read this first. The design is described in the [solution architecture](fabric-solution-architecture.md), and the step-by-step procedures, checklists and scripts are in the [full runbook](fabric-capacity-migration-plan.md)._

| | |
|---|---|
| **Document type** | Overview (companion to the migration runbook) |
| **Version** | 1.2 |
| **Status** | Draft for review |
| **Last updated** | 2026-11-12 |
| **Owner** | Data Services, Enterprise Technology |
| **Audience** | Approvers, BI team, platform/capacity admin, business owners (BG1, BG2), Slalom |

---

## 1. Executive summary

Our Fabric capacity **A** runs both production and non-production workloads and is **being throttled**. Data volume keeps growing, and our semantic models are getting slower and using more compute. Slalom is rebuilding the **gold layer** and the **semantic models** on a separate development capacity (**B**) so they cost less to run.

This overview explains how we bring that work into production **on capacity A** without disrupting the business:

- **We build and prove the work on B** before it touches A.
- **We promote it through our existing release path**: Azure DevOps → non-prod on A → Fabric deployment pipelines → prod. There's no shortcut route to prod.
- **We move reports in waves.** For a while, the old and new versions run side by side. Reports are **rebound**, not replaced, so users keep the same links, apps and subscriptions.
- **Every step can be undone.** Each wave has a Go/No-Go decision, a rehearsed rollback and a hypercare period.

**What we need now:** approval of this approach, named owners for each role, and decisions on the items in §10.

## 2. Problem statement

| Symptom | Impact |
|---|---|
| Semantic models getting slower as data grows | Reports load slowly; refreshes take longer |
| Rising compute (CU) use on capacity A | **Throttling**, which slows reports for everyone, including during business-critical periods |
| Prod and non-prod sharing the same capacity | Development and testing add to the load on prod |

Buying more capacity alone would only put off the problem. The lasting fix is to make the **gold layer and models more efficient**, which is what Slalom is building.

## 3. Objectives and success measures

| Objective | How we measure it | Target |
|---|---|---|
| Reduce load on capacity A | CU use of the in-scope models (refresh + queries), before vs after | Reduction agreed in decision D-1 |
| Remove throttling for report users | Throttling events in the Capacity Metrics app | No user-facing throttling after the final wave |
| Keep the numbers correct | Reconciliation of new gold vs current gold; KPI cross-check | Exact match, or a documented and approved correction |
| No disruption for users | Report URLs, apps, bookmarks, subscriptions | Unchanged |
| Keep security intact | RLS test cases | 100% pass |
| Controlled, auditable change | Every prod change has an Azure DevOps pull request, an approval and a release tag | 100% |

## 4. Scope

**In scope**

- New silver-to-gold notebooks and a new gold lakehouse
- Changes to the elt orchestration pipeline
- New semantic models for business groups 1 and 2
- Rebinding the existing prod reports to the new models, in waves
- Retiring the old gold layer and old models once nothing uses them

**Out of scope**

- Changes to bronze/silver ingestion
- Creating new reports (these go through the normal release process)
- Changing capacity sizes, or moving workspaces between capacities

## 5. Current state and target state

| | **Current** | **During the migration** | **Target** |
|---|---|---|---|
| Capacity A | Prod and non-prod | Prod and non-prod | Prod and non-prod (unchanged) |
| Capacity B | – | Slalom development and heavy testing | Development area; future use decided at close-out |
| Gold layer | Old gold | **Old and new gold running side by side** | New gold only |
| Semantic models | Old models | Old models for reports not yet moved; new models for moved reports | New models only |
| Reports | Bound to old models | Moving in waves | All bound to new models, with the same IDs and URLs |
| Release path | Git (dev) → deployment pipelines | Azure DevOps → deployment pipelines started from ADO, with approvals | Same as during the migration |

**Workspace layout (unchanged):** elt (orchestration), transf (silver → gold), cons-bg1 and cons-bg2 (semantic models and reports), each with dev/test and prod.

## 6. Approach

### 6.1 Release path

```
 Capacity B (dev area)                 Capacity A (non-prod → prod)
 ┌──────────────────────┐             ┌──────────────────────────────────────┐
 │ Slalom branched      │  Azure      │ dev ──▶ test ──▶ prod                 │
 │ workspaces: build +  │  DevOps     │ (deployment pipelines, started from  │
 │ heavy testing        │ ──────────▶ │  Azure DevOps with approvals)        │
 └──────────────────────┘  pull req.  └──────────────────────────────────────┘
```

### 6.2 Phases and gates

Each phase ends with a **gate**. The next phase doesn't start until the gate's evidence has been checked and recorded.

| Phase | What happens | Gate (exit criteria, summarised) |
|---|---|---|
| **0. Readiness** | Confirm assumptions; capture CU, data and report baselines; set up Azure DevOps, variable libraries and Slalom's workspaces | Baselines stored; tooling in place; key decisions made |
| **1. Build and prove on B** | Run new gold against prod silver for ≥ 3 cycles; reconcile; load-test the new models; close field-name gaps | Numbers reconcile; **new models use fewer CUs**; no broken visuals |
| **2. Merge into non-prod on A** | Pull requests in Azure DevOps; deploy to test; UAT; rehearse the rollback | UAT signed off; rollback rehearsed; all references point at test |
| **3. New gold in prod, no reports reading it** | Deploy, build and reconcile the new gold in prod alongside the old | Prod reconciliation passed; CU budget for running both approved |
| **4–5. Waves** (repeated) | Move one wave of reports to the new models; hypercare; switch off old items nothing uses any more | Go/No-Go per wave; hypercare clean; business sign-off |
| **6. Close-out** | Remove the rest of the old stack; bring environments in sync; decide B's future | Old stack gone; nothing depends on B |

### 6.3 Running both stacks side by side (waves)

The new gold and models **won't serve every report at first**. During the migration:

- **Wave 1 is a pilot** of 1–3 low-risk reports. Later waves are grouped by subject area, **highest CU savings first**.
- **Reports not yet moved stay on the old models**, which keep refreshing.
- **The same KPI must give the same number in both stacks.** An automated cross-check runs after every load.
- **A dependency map** shows when each old table, notebook or model has nothing left using it. It's then switched off, and deleted after its rollback period.
- **A CU budget** caps the combined load on A while both run, and is reviewed weekly. Running both stacks has a **planned end date** (decision D-7).

## 7. Tools

| Need | Tool |
|---|---|
| Version control and code review | **Azure DevOps Repos**: one folder per workspace; branch policies; required reviewers; automated checks for hard-coded values |
| Slalom's development | One **branched workspace per target folder** on capacity B |
| Moving to test and prod | **Fabric deployment pipelines, started from an Azure DevOps pipeline** with approval steps; every prod release is tagged |
| Per-environment settings | **Fabric variable libraries** with dev, test and prod values |
| Checks after each deploy | Automated in the ADO pipeline: references, **which lakehouse each Direct Lake model reads from**, report bindings |
| Inventory and lineage | Admin scanner API (through semantic-link-labs) |
| Monitoring and alerts | Capacity Metrics app, monitoring hub, Activator alerts |

**Fabric pitfalls this plan handles explicitly** (details in runbook §5b):

- **Direct Lake models don't repoint to the right lakehouse when deployed** → stage-specific rules, plus a hard check after every deployment.
- **References to items in other workspaces never repoint automatically** (for example, the elt pipeline calling transf notebooks) → IDs come from variable libraries.
- **Deployments and Git updates can delete shortcuts** in the target lakehouse → no manual shortcuts in test or prod; changes reviewed before every deploy.
- **Deployment pipelines pair items by name** → new items use `_v2` names, and renaming is banned during the migration.

## 8. How we keep it safe

| Control | What it means |
|---|---|
| **Gates** | Evidence-based exit criteria at the end of every phase |
| **Go/No-Go** | A checklist before every wave; a single "No" stops that wave |
| **Rollback** | Per wave: rebind the wave's reports to the old models (still refreshing). Rehearsed in test, with timings. The same change is committed in Azure DevOps so the next deployment doesn't undo it |
| **Immediate rollback rule** | Any RLS/security defect triggers rollback, with no attempt to fix it in place |
| **Disable before delete** | Old items are switched off first and deleted only after their rollback period |
| **Separation of duties** | Slalom builds and opens pull requests; the internal BI team reviews, approves and deploys |
| **Audit trail** | Pull request, approval and release tag for every prod change |

## 9. Governance and roles

| Role | Responsibilities | Named owner (backup) |
|---|---|---|
| **Data Services** (plan owner) | Owns the plan; chairs Go/No-Go; accountable for delivery | ______ (______) |
| **BI team** | Reviews and merges, deploys, runs checks, runs rollbacks, retires old items | ______ (______) |
| **Platform / capacity admin** | Capacity settings, CU monitoring, CU budget | ______ (______) |
| **Business owner – BG1** | UAT, wave sign-off, user communication | ______ |
| **Business owner – BG2** | UAT, wave sign-off, user communication | ______ |
| **Slalom** | Builds and validates on B; opens pull requests; supports hypercare until handover | ______ |

**Decision rights:** Data Services makes the Go/No-Go call. Each business owner must agree for waves that affect their reports. Escalation goes to the approvers listed in the runbook.

**Reporting:** a weekly status covering wave progress, CU against budget, KPI cross-check and the end-date forecast. A daily status during hypercare.

## 10. Decisions needed

| # | Decision | Owner | Needed by |
|---|---|---|---|
| D-1 | Target CU reduction for the new models | Data Services + platform admin | Before Phase 1 sign-off |
| D-2 | Reconciliation tolerances (default: exact match) | Business owners | Before Phase 1 sign-off |
| D-7 | End date for running both stacks, and the number of waves | Approvers | Before Wave 1 |
| D-8 | CU budget on A while both stacks run | Platform admin + Data Services | Before Wave 1 |
| D-10 | Which reports go in the pilot wave | BI team + business owners | Phase 0 |
| D-11 | Promotion route: deployment pipelines started from ADO (default), or `fabric-cicd` | Data Services + BI team | Phase 0 |
| D-12 | How notebooks find their default lakehouse in each environment | BI team + Slalom | Phase 0 |

The full list (D-1 to D-12) is in runbook §19.

## 11. Top risks

| Risk | Mitigation |
|---|---|
| A **Direct Lake model in prod reads the test lakehouse** after deployment | Stage-specific binding rules; hard check after every deploy |
| **The same KPI shows different numbers** in old and new reports during the migration | KPI cross-check after every load; intended corrections told to users in advance |
| **Running both stacks takes longer than planned**, and A pays for both | Planned end date; waves ordered by CU savings; weekly CU budget review |
| **The new models aren't cheaper** than the old ones | Hard CU gate before anything reaches A |
| **A rollback is undone** by the next deployment | Rollback is also committed in Azure DevOps; the cons pipeline is frozen until then |

The full register (23 risks, each with a mitigation and a fallback) is in runbook §16.

## 12. Indicative milestones

Dates are set at Phase 0 sign-off.

| Milestone | Marks |
|---|---|
| M0 – Plan approved | Runbook approved; owners named |
| M1 – Readiness complete | Gate G0: baselines, Azure DevOps and tooling in place |
| M2 – Proven on B | Gate G1: reconciliation and CU gate passed |
| M3 – Ready in test | Gate G2: UAT signed off; rollback rehearsed |
| M4 – New gold live in prod | Gate G3: new gold running in prod, no reports reading it yet |
| M5 – Pilot wave live | First reports on the new models; hypercare starts |
| M6 – Final wave live | All reports on the new models |
| M7 – Close-out | Old stack retired; decision on B recorded |

## 13. What we need from you

| Group | Ask |
|---|---|
| **Approvers** | Approve this approach; confirm owners; make decisions D-7 and D-8 |
| **BI team** | Set up Azure DevOps policies and environments, variable libraries and the post-deploy checks; own reviews and deployments |
| **Platform / capacity admin** | Capture the CU baseline; set the CU budget; turn on surge protection |
| **Business owners** | Confirm tolerances (D-2); pick the pilot reports (D-10); provide UAT testers and sign-off |
| **Slalom** | Split work into branched workspaces per folder; follow the tooling standards (runbook §5b); provide the validation evidence |

## 14. What report users will notice

Nothing should change except a possible **data correction**. Where Slalom's fixes change a number on purpose, users of the affected reports are told before their wave goes live. Users get a notice **5 business days** before their wave, and an update when it is complete.

## 15. Glossary

| Term | Meaning |
|---|---|
| **Capacity (A, B)** | The pool of Fabric compute (CUs) that workspaces run on. A = prod and non-prod; B = development area |
| **CU** | Capacity unit, the measure of Fabric compute use |
| **Throttling** | Fabric slowing or rejecting work when a capacity is over its limit |
| **Gold layer** | Curated tables that semantic models and reports use |
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
| [Solution architecture](fabric-solution-architecture.md) | How the solution is built: capacities, workspaces, data flow, semantic layer, environments, security, design decisions |
| [Migration runbook](fabric-capacity-migration-plan.md) | Full procedures, checklists, rollback, risk register, scripts |
| Runbook §5b | Fabric and Azure DevOps tooling standards |
| Runbook §14–§16 | Go/No-Go, rollback, risk register |

| If you are… | Read in the runbook |
|---|---|
| An approver | §14 Go/No-Go, §15 Rollback, §16 Risks, §19 Decisions |
| On the BI team or with Slalom | §5b Tooling standards, Phases 0–6, Appendix A (scripts) |
| A business owner | §5a.1 Wave plan, §17 Communication, §18 Acceptance criteria |
