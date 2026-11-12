# Runbook: Promoting the New Gold Layer and Semantic Models to Production

| | |
|---|---|
| **Document type** | Migration runbook |
| **Version** | 2.0 |
| **Status** | Draft – for review |
| **Last updated** | 2026-11-12 |
| **Owner** | Data Services (Enterprise Technology) |
| **Approvers** | BI team lead · Platform/capacity admin · Business owners (BG1, BG2) |
| **Applies to** | Microsoft Fabric capacity A (prod + non-prod) and capacity B (development area) |

---

## Table of contents

1. [Purpose and scope](#1-purpose-and-scope)
2. [Current and target state](#2-current-and-target-state)
3. [Assumptions and constraints](#3-assumptions-and-constraints)
4. [Roles and responsibilities](#4-roles-and-responsibilities)
5. [Guiding principles](#5-guiding-principles)
6. [Phase overview](#6-phase-overview)
7. [Phase 0 – Readiness and baseline](#phase-0--readiness-and-baseline)
8. [Phase 1 – Build and validate on capacity B](#phase-1--build-and-validate-on-capacity-b)
9. [Phase 2 – Merge into the non-prod lanes on capacity A](#phase-2--merge-into-the-non-prod-lanes-on-capacity-a)
10. [Phase 3 – Run the new gold in prod without consumers](#phase-3--run-the-new-gold-in-prod-without-consumers)
11. [Phase 4 – Promote the semantic models and rebind the reports](#phase-4--promote-the-semantic-models-and-rebind-the-reports)
12. [Phase 5 – Cutover and hypercare](#phase-5--cutover-and-hypercare)
13. [Phase 6 – Retire the old items and close out](#phase-6--retire-the-old-items-and-close-out)
14. [Go/No-Go checklist](#14-gono-go-checklist)
15. [Rollback plan](#15-rollback-plan)
16. [Risk register](#16-risk-register)
17. [Communication plan](#17-communication-plan)
18. [Acceptance criteria](#18-acceptance-criteria)
19. [Open decisions](#19-open-decisions)
20. [Appendix A – Verification scripts](#appendix-a--verification-scripts)
21. [Appendix B – References](#appendix-b--references)

---

## 1. Purpose and scope

**Purpose.** Move Slalom's work into production with **no unplanned outage, no data loss, and a tested rollback at every step**. The work is a new gold layer, updated orchestration, and new semantic models for the existing reports. It is built on capacity B and must reach production on capacity A.

**In scope**

- New silver-to-gold notebooks and the **new gold lakehouse**
- Changes to the elt orchestration pipeline
- New semantic models for business groups 1 and 2
- **Rebinding the existing prod reports** to the new semantic models
- Retiring the old gold items and old semantic models

**Out of scope**

- Changes to bronze/silver ingestion
- Creating new reports (any new reports go through the normal release process)
- Changing capacity SKUs or moving workspaces between capacities

## 2. Current and target state

### 2.1 Current state

**Capacity A** holds **both non-prod and prod** workspaces:

| Layer | Non-prod (A) | Prod (A) |
|---|---|---|
| ELT / orchestration | elt-dev (/test) | elt-prod |
| Transformation (silver → gold) | transf-dev (/test) | transf-prod |
| Consumption – BG1 | cons-bg1-dev (/test) | cons-bg1-prod |
| Consumption – BG2 | cons-bg2-dev (/test) | cons-bg2-prod |

- The end-to-end orchestration lives in **elt**. It runs notebooks in several workspaces to build the gold tables used by **cons**.
- **Fabric deployment pipelines** promote content from non-prod to prod, all within A.
- **Problem:** data volume keeps growing, so the semantic models are getting slower and using more CUs, and A is **throttled**.

**Capacity B** is a **development area only**. Slalom works in **`trans-dev-2`** on B, reading data from A, and is building a new gold layer and new semantic models for the existing prod reports.

### 2.2 Target state

**Every workspace stays on capacity A.** The workspace layout and the deployment pipelines don't change. What changes is the content inside them:

| Layer (non-prod and prod on A) | Change |
|---|---|
| elt | Orchestration calls the new notebooks; the old notebook activities are turned off, then removed |
| transf | New notebooks plus the **new gold lakehouse**; the old gold is retired after cutover |
| cons-bg1 / cons-bg2 | **New semantic models**; the **existing reports are rebound** to them (report IDs, URLs, apps, bookmarks and subscriptions are kept) |

Capacity B stays a development area, and **no prod or non-prod item on A depends on it**. A gets relief because the new models and gold are **cheaper to run**, not because work moves off A. So the CU check in Phase 1 is a **hard gate**.

## 3. Assumptions and constraints

Each assumption must be **checked in Phase 0** and recorded as passed or failed. If any fails, stop and update this runbook before going further.

| # | Assumption / constraint | How to check | If false |
|---|---|---|---|
| A1 | A and B are in the **same Fabric region** | Admin portal → Capacity settings → Region | Shortcuts still work, but Direct Lake models must be in the same region as their source lakehouse. Build only on A non-prod, or rebuild B in A's region. |
| A2 | Existing non-prod workspaces on A are connected to Git (or can be) | Workspace settings → Git integration | Connect them in Phase 0 and commit a baseline. If Git is blocked by policy, use **Fallback F2-b** (`fabric-cicd`) or **F2-c** (copy by hand). |
| A3 | Deployment pipelines exist for elt, transf, cons-bg1 and cons-bg2 | Deployment pipelines list | Create or repair the pipeline before Phase 2. |
| A4 | Deployment pipelines **pair items by name and type**; new or renamed items are created as new items in the next stage | Microsoft docs; dry run in test | – (design accounts for it) |
| A5 | References to items in **other workspaces** (elt → transf notebooks, model → lakehouse) are **not remapped automatically** on deploy | Verify with a test deploy in Phase 2 | If they *are* remapped, keep the variable library anyway; it does no harm. |
| A6 | The new semantic models can serve the existing reports **once field names are reconciled** | Field comparison script, Appendix A.2 | Compatibility measures, or report changes, or replacement reports (Phase 4 fallbacks) |
| A7 | Reads through a shortcut are charged to the **capacity of the workspace doing the reading** | Microsoft docs; Capacity Metrics app | – |
| A8 | Prod has a known **refresh/ETL window** and an off-peak window | Pipeline schedules; Capacity Metrics app | Agree windows with the business before Phase 3 |
| A9 | Slalom has **no write access** to prod workspaces | Workspace access lists | Remove access before Phase 3 |

## 4. Roles and responsibilities

R = Responsible · A = Accountable · C = Consulted · I = Informed

| Activity | Slalom | BI team | Data Services (owner) | Platform / capacity admin | Business owners |
|---|---|---|---|---|---|
| Build and validate on B | **R** | C | **A** | I | I |
| Code review / pull request approval | R (author) | **R** | **A** | I | – |
| Deployment rules and variable libraries | C | **R** | **A** | C | – |
| Deployments to test and prod | – | **R** | **A** | C | I |
| Reconciliation and CU sign-off | R | **R** | **A** | C | I |
| UAT | C | R | A | – | **R** |
| Go/No-Go decision | C | C | **A/R** | C | **C (must agree)** |
| Rollback execution | C | **R** | **A** | C | I |
| Retiring old items | – | **R** | **A** | I | I |

Named individuals and backups are listed in the [Communication plan](#17-communication-plan).

## 5. Guiding principles

1. **One route to prod.** Every prod change goes Git → non-prod on A → deployment pipeline → prod. `trans-dev-2` is never promoted directly.
2. **Validate where it's cheap.** Heavy reconciliation and load testing run on B. A only gets smoke tests and the final run.
3. **Add before you replace.** New items run **alongside** the old ones until cutover. Old items are disabled, not deleted, until hypercare ends.
4. **Keep report identity.** Rebind the existing reports instead of replacing them, so URLs, apps, bookmarks and subscriptions survive.
5. **Every phase has a gate.** No phase starts until the previous phase's exit criteria are met and recorded.
6. **Every step can be reversed.** Each phase lists its fallback. The rollback in §15 is **rehearsed in test** before cutover.
7. **No hard-coded environment values.** Workspace, lakehouse and notebook IDs and paths come from variable libraries or deployment rules.
8. **Never run both versions on A for longer than planned.** Old gold and old model refreshes are turned off at cutover.

## 6. Phase overview

| Phase | Name | Capacity | Gate to exit |
|---|---|---|---|
| 0 | Readiness and baseline | A, B | Assumptions checked; baselines captured; Git ready |
| 1 | Build and validate on B | B | Reconciliation passed; CU cost of new models ≤ old; field comparison clean |
| 2 | Merge into non-prod lanes on A | A (non-prod) | Merged; parameters set; smoke test and UAT passed in test |
| 3 | Run the new gold in prod without consumers | A (prod) | New gold built in prod; reconciles against old gold |
| 4 | Promote semantic models and rebind reports | A (prod) | New models refreshed; reports rebound and visually checked |
| 5 | Cutover and hypercare | A (prod) | Old pipelines off; hypercare period passed with no Sev-1/Sev-2 issues |
| 6 | Retire old items and close out | A, B | Old items deleted; environments in sync; decision on B recorded |

---

## Phase 0 – Readiness and baseline

**Objective.** Confirm the assumptions, record the "before" picture, and prepare source control.

**Entry criteria**

- This runbook is approved by the approvers listed at the top
- Roles in §4 have named people and backups

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 0.1 | Check assumptions A1–A9 (§3) and record each as passed or failed | Data Services | Completed §3 table |
| 0.2 | **Record the CU baseline** for A: 14-day peak, average, and throttling events, broken down by item (models, notebooks, pipelines) from the Capacity Metrics app | Platform admin | Metrics export |
| 0.3 | **Record the data baseline:** row counts, key totals and distinct key counts for every gold table in scope (Appendix A.1) | BI team | Baseline file in the repo |
| 0.4 | **Record the report inventory:** every prod report bound to the old models, with workspace, report ID, model ID, app membership, subscriptions, and Excel/Analyze-in-Excel use | BI team | Inventory sheet |
| 0.5 | **Record the schema of the old models:** tables, columns, measures, relationships and roles (RLS) | BI team | Model schema export (Appendix A.2) |
| 0.6 | **Back up the old models** (export the definitions to Git/PBIP; for Import models also take an XMLA/.abf backup if allowed) | BI team | Backup location recorded |
| 0.7 | Connect the existing non-prod workspaces on A to Git (if not already) and **commit the baseline** | BI team | Baseline commit hash |
| 0.8 | Connect `trans-dev-2` to a feature branch (e.g. `feature/slalom-gold-v2`) in the same repo | Slalom | Branch exists |
| 0.9 | Set branch protection: pull requests required, at least one internal approver, no direct pushes to the dev/main branches | BI team | Repo settings |
| 0.10 | Remove any Slalom write access to prod workspaces (A9) | Platform admin | Access review |
| 0.11 | Agree the change windows (deployment, smoke test, cutover) and the freeze period | Data Services + business | Calendar holds |

**Exit criteria (gate G0)**

- [ ] A1–A9 recorded; any failed item has a decision and this runbook is updated
- [ ] CU, data, report and schema baselines are stored in the repo or a shared location
- [ ] Old models backed up
- [ ] Git baseline committed; feature branch and branch protection in place

**Fallback**

- **F0-a:** If Git integration can't be turned on for a workspace, export its item definitions (notebooks `.ipynb`, reports/models as PBIP) to the repo by hand as the baseline, and use **F2-b** or **F2-c** in Phase 2.

---

## Phase 1 – Build and validate on capacity B

**Objective.** Prove that the new gold is correct and the new models are cheaper, **before anything touches A**.

**Entry criteria**

- Gate G0 passed
- Slalom reports that development is complete

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 1.1 | `trans-dev-2` reads **prod silver** on A through OneLake shortcuts (reads are charged to B) | Slalom | Shortcut list |
| 1.2 | Run the new silver-to-gold notebooks for **at least 3 consecutive load cycles**, including one month-end or other peak period if the timeline allows | Slalom | Run history |
| 1.3 | **Reconcile new gold against old prod gold** for each cycle: row counts, key totals, distinct keys, null rates, SCD current/historical row counts (Appendix A.1) | Slalom + BI team | Reconciliation report |
| 1.4 | Explain **every variance**: either a bug (fix it and run again) or an intended correction (document it and get business sign-off) | Slalom + business | Variance log |
| 1.5 | **Compare field names** between the old and new semantic models, and between the new models and the fields each report uses (Appendix A.2) | Slalom + BI team | Gap list |
| 1.6 | Close each gap: keep the old name, add a **compatibility measure/column**, or log a report change | Slalom | Gap list closed |
| 1.7 | Rebind **copies** of the prod reports in `trans-dev-2` to the new models; open every page and check for broken visuals | Slalom + BI team | Visual check log with screenshots |
| 1.8 | **Load-test** the new models: refresh plus a typical query load (or replay of the top report pages). Record CU from the Metrics app on B | Slalom | CU comparison vs baseline 0.2 |
| 1.9 | Check model hygiene: V-Order/OPTIMIZE on gold tables, incremental refresh or partitioning set up, unused and high-cardinality columns removed, Direct Lake guardrails respected for A's SKU | Slalom + BI team | Checklist |
| 1.10 | Check that **RLS/OLS roles** in the new models match the old ones (same role names, same filter logic, same membership) | BI team | Role comparison |
| 1.11 | Remove all hard-coded references to B (default lakehouse, GUIDs, ABFS paths). Every environment value comes from a **variable library** or parameter | Slalom | Code review checklist |

**Exit criteria (gate G1)**

- [ ] Reconciliation passed for ≥ 3 cycles within the agreed tolerance (§18), with every variance explained and signed off
- [ ] **CU cost of the new models ≤ the old models** at the same load (target: a measurable reduction agreed in §19)
- [ ] Field-name gap list closed; every report copy renders with no broken visuals
- [ ] RLS/OLS roles match
- [ ] No hard-coded references to B in the code

**Fallback**

- **F1-a (reconciliation fails):** Slalom fixes and re-runs. Phase 2 doesn't start. The timeline moves; prod is unaffected.
- **F1-b (new models aren't cheaper):** Stop. Tune the models (see 1.9) and test again. If they still aren't cheaper, take it to the approvers: promote only the gold fixes, or rethink the model design. Don't promote a model that adds load to A.
- **F1-c (a field-name gap can't be closed):** Mark the affected reports for **replacement** instead of rebinding (see F4-b), and plan the app update and user communication for them.

---

## Phase 2 – Merge into the non-prod lanes on capacity A

**Objective.** Bring the validated work into the **existing** non-prod workspaces on A through Git, with all environment values parameterized, and confirm it in test.

**Entry criteria**

- Gate G1 passed
- Change window agreed; non-prod freeze announced to other developers

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 2.1 | Slalom opens a **pull request** from `feature/slalom-gold-v2` into the dev branch. Each item goes into its target workspace folder: notebooks/lakehouse → transf, orchestration → elt, models → cons-bg1/bg2 | Slalom | Pull request link |
| 2.2 | **Code review:** notebook diffs, gold schema changes, new or renamed items, variable-library use, no secrets, no B references | BI team | Approved review |
| 2.3 | Merge and **sync each dev workspace on A from Git**. Confirm item counts and names match what's expected | BI team | Sync log; item list |
| 2.4 | Set up the **variable library** (or pipeline parameters) for each stage: workspace IDs, lakehouse IDs, notebook IDs used by elt, silver source, new-gold target | BI team | Variable library values for each stage |
| 2.5 | Set **deployment rules**: notebook default lakehouse for each stage; semantic model data source/parameters pointing at that stage's new-gold | BI team | Rule screenshots |
| 2.6 | In **elt-dev**, add the new notebook activities to the orchestration. **Keep** the old activities for now; control which path runs with a pipeline parameter or variable (e.g. `gold_version = old \| new \| both`) | BI team | Pipeline definition |
| 2.7 | Rebind the **dev copies** of the existing reports to the new models (Appendix A.3) | BI team | Rebind log |
| 2.8 | Deploy **dev → test** in this order: **transf → elt → cons** | BI team | Deployment history |
| 2.9 | **Check references after deploying:** open the test lakehouse shortcuts, the notebook default lakehouses, the elt notebook activities and the model data source; each must point at **test** items (A5) | BI team | Checklist with screenshots |
| 2.10 | **Smoke test in test** at **off-peak** hours: one full orchestration run with `gold_version = new`, a light reconciliation (row counts only), and one model refresh | BI team | Run results |
| 2.11 | **UAT in test** by each business group, using the agreed UAT script (key reports, key numbers, RLS spot-checks with test users) | Business owners | UAT sign-off |
| 2.12 | **Rehearse the rollback** (§15) in test: rebind to the old model, switch `gold_version = old`, confirm the reports work, then return to new | BI team | Rehearsal log with timings |

**Exit criteria (gate G2)**

- [ ] Pull request merged; dev workspaces synced; item inventory matches
- [ ] All references in test point at test items (2.9)
- [ ] Smoke test passed; UAT signed off by both business groups
- [ ] Rollback rehearsed successfully, with timings recorded

**Fallback**

- **F2-a (sync or merge conflicts):** Fix them in Git, not in the workspace. If an item becomes unusable, restore it from the Phase 0 baseline commit.
- **F2-b (Git unavailable or blocked):** Deploy with **`fabric-cicd`** from the repo into each dev workspace, using a `parameter.yml` that swaps IDs per environment. Then continue with the normal deployment pipeline.
- **F2-c (last resort):** Copy items by hand into the dev workspaces (import notebooks; publish PBIP for models and reports), following a checklist with a second person checking. Re-check all references afterwards (2.9).
- **F2-d (smoke test or UAT fails):** Stay in test. Fix in the feature branch → new pull request → resync → redeploy to test. Prod is unaffected.

---

## Phase 3 – Run the new gold in prod without consumers

**Objective.** Build the new gold lakehouse in prod and run it **alongside** the old gold, **with nothing reading from it yet**.

**Entry criteria**

- Gate G2 passed
- Change window booked (off-peak); Go/No-Go for Phase 3 recorded

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 3.1 | Deploy **test → prod**: **transf**, then **elt**, with prod `gold_version = old` (the new path is deployed but not running) | BI team | Deployment history |
| 3.2 | **Check references after deploying** (as in 2.9): prod notebooks read **prod silver** and write to **prod new-gold**; elt activities point at **prod** notebooks | BI team | Checklist with screenshots |
| 3.3 | Note: the new gold lakehouse is **empty after deployment** (definitions only, no data). Run the **initial load or backfill** of new gold off-peak | BI team | Run log; duration; CU used |
| 3.4 | Switch prod to `gold_version = both` for **1–2 cycles** at most. Watch CU on A throughout | BI team + platform admin | Metrics screenshots |
| 3.5 | **Reconcile prod new-gold against prod old-gold** (Appendix A.1) | BI team | Reconciliation report |

**Exit criteria (gate G3)**

- [ ] New gold built in prod; reconciliation passes within tolerance (§18)
- [ ] CU on A during the side-by-side run stayed below the throttling threshold
- [ ] No consumer reads from new gold yet

**Fallback**

- **F3-a (deploy or reference error):** Set `gold_version = old`. Nothing in prod reads new gold, so **users are unaffected**. Fix in dev and redeploy.
- **F3-b (CU pressure during the side-by-side run):** Stop the side-by-side run (`gold_version = old`). Rerun the new path only in the next off-peak window, or go straight from the initial-load reconciliation to Phase 4 if the B and test evidence is strong. The approvers decide.
- **F3-c (reconciliation fails in prod but passed on B):** Look first for environment differences: silver source, shortcut targets, parameters. Stay in Phase 3 until it's resolved.

---

## Phase 4 – Promote the semantic models and rebind the reports

**Objective.** Put the new semantic models in prod and move the **existing** reports onto them without changing report identity.

**Entry criteria**

- Gate G3 passed
- **Final Go/No-Go (§14) approved**
- Cutover communication sent (§17)

**Steps (recommended path: rebind in dev, deploy together)**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 4.1 | Confirm the cons-test stage holds the new models and the **rebound** reports (from 2.7/2.8) | BI team | Item list |
| 4.2 | Deploy **cons-bg1 test → prod** with the **semantic model and its reports selected together**. The pipeline binds the prod reports to the prod copy of the new model. Report IDs stay the same | BI team | Deployment history |
| 4.3 | Check the deployment rule: the prod model points at **prod new-gold** | BI team | Screenshot |
| 4.4 | **Refresh (or reframe) the new prod model**; confirm it succeeds and record how long it takes | BI team | Refresh history |
| 4.5 | **Check each rebound report:** open every page, compare key numbers with the baseline (0.3), test RLS with test users | BI team + business | Check log |
| 4.6 | Repeat 4.2–4.5 for **cons-bg2** | BI team | As above |
| 4.7 | For any reports marked for **replacement** (F1-c): publish the replacement, update the **workspace app**, move subscriptions | BI team | App version; subscription list |
| 4.8 | Repoint **Excel / Analyze-in-Excel** connections and other models connected to the old model (from inventory 0.4); tell the owners | BI team | Owner confirmations |

**Exit criteria (gate G4)**

- [ ] New models refreshed in prod; all inventoried reports bound to the new models (or replaced, per F1-c)
- [ ] Every report page renders; key numbers match within tolerance; RLS checked
- [ ] Excel and other connected consumers repointed or their owners told

**Fallback**

- **F4-a (pipeline didn't rebind the prod report):** Rebind directly in prod with the **Rebind Report In Group** REST API or `sempy_labs.report.report_rebind` (Appendix A.3). Then make the same change in dev so dev and prod stay in step.
- **F4-b (a report can't be rebound cleanly):** Publish the replacement report, update the workspace app, move subscriptions, and tell its users. Keep the old report (bound to the old model) hidden from the app for the rollback period.
- **F4-c (new model refresh fails in prod):** Rebind the affected reports back to the old model (§15, steps R1–R3). Old models are still refreshing during Phase 4, so users still see current data.

---

## Phase 5 – Cutover and hypercare

**Objective.** Make the new path the only one running on A, and watch it closely through a defined hypercare period.

**Entry criteria**

- Gate G4 passed

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 5.1 | Switch prod to `gold_version = new` (old gold notebooks stop running) | BI team | Pipeline parameter value |
| 5.2 | **Pause scheduled refresh on the old semantic models.** Don't delete them | BI team | Refresh settings |
| 5.3 | Turn on **surge protection** on A, if not already on | Platform admin | Capacity settings |
| 5.4 | **Hypercare** – at least **one full reporting cycle** (including a month-end if possible): check daily that the pipeline succeeds, refreshes succeed, CU stays healthy, and no business incidents come in | BI team | Daily hypercare log |
| 5.5 | Compare CU on A with the baseline (0.2) at the end of hypercare | Platform admin | Before/after report |
| 5.6 | Business owners confirm the cutover worked | Business owners | Written sign-off |

**Exit criteria (gate G5)**

- [ ] Hypercare complete with **no open Sev-1 or Sev-2** incidents
- [ ] CU on A at or below the target (§18)
- [ ] Business sign-off received

**Fallback**

- **F5-a:** Any rollback trigger in §15.1 → run the **rollback** (§15.2).

---

## Phase 6 – Retire the old items and close out

**Objective.** Remove the old items, bring dev, test and prod back in step, and decide what happens to capacity B.

**Entry criteria**

- Gate G5 passed (the rollback option ends here; record this in the decision log)

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 6.1 | Take a final backup of the old model definitions and an export of the old gold table schemas | BI team | Backup location |
| 6.2 | In **dev**: remove the old notebook activities and the `gold_version` switch from elt; delete the old gold notebooks, old gold lakehouse and old models. Commit through a pull request | BI team | Pull request link |
| 6.3 | Deploy dev → test → prod so the deletions reach every stage. Confirm the prod item inventory | BI team | Deployment history; inventory |
| 6.4 | Check that **no prod or non-prod item on A references B** | BI team | Lineage view check |
| 6.5 | **Decide B's future:** keep it for future development, scale it down, or pause it between engagements. A paused capacity still pays for stored data, so clean up `trans-dev-2` if it's no longer needed | Data Services + platform admin | Decision log entry |
| 6.6 | Update architecture documentation and this runbook's status to **Complete** | Data Services | Updated docs |
| 6.7 | Lessons-learned review with Slalom and the BI team | Data Services | Notes |

**Exit criteria (gate G6)**

- [ ] Old items removed in every stage; environments in sync
- [ ] Nothing on A depends on B
- [ ] Decision on B recorded; documentation updated

---

## 14. Go/No-Go checklist

Must be completed and signed **before Phase 4 (cutover)**. A single "No" means **No-Go**.

| # | Criterion | Yes/No | Evidence |
|---|---|---|---|
| G-1 | Gates G0–G3 all passed and recorded | | |
| G-2 | Reconciliation passed on B (≥ 3 cycles), in test, and in prod | | |
| G-3 | New models' CU cost ≤ old (Phase 1.8) | | |
| G-4 | UAT signed off by BG1 and BG2 | | |
| G-5 | Field-name gaps closed; every report page checked in test | | |
| G-6 | RLS/OLS roles match and were tested with test users | | |
| G-7 | Rollback rehearsed in test; timing within the agreed limit | | |
| G-8 | Old model backups exist and were checked | | |
| G-9 | No conflicting change or freeze-breaking release in the window | | |
| G-10 | Support people available for the cutover window and hypercare | | |
| G-11 | Business communication sent (§17) | | |
| G-12 | Capacity A has headroom for the window (Metrics app check on the day) | | |

**Decision:** Go / No-Go · **Decided by:** __________ · **Date/time:** __________

## 15. Rollback plan

### 15.1 Rollback triggers

Run the rollback if **any** of these happen after cutover and can't be fixed within the agreed time limit (§19):

- Key numbers differ from the agreed tolerance (§18) on a business-critical report
- The new model's refresh fails twice in a row, or data is late beyond the agreed limit
- Report visuals broken on pages listed as business-critical
- An RLS/OLS defect (users see data they shouldn't, or are missing data they should see) – **immediate rollback, no fix-forward**
- CU on A rises above the agreed threshold because of the new items, with throttling seen by users
- The business owner withdraws sign-off

### 15.2 Rollback procedure

Available until the end of Phase 5. Rehearsed in step 2.12.

| Step | Action | Notes |
|---|---|---|
| R1 | **Resume scheduled refresh on the old semantic models** and run one refresh now | Old models were paused, not deleted |
| R2 | **Rebind the prod reports back to the old models** (Appendix A.3, with the old model IDs from inventory 0.4) | Report IDs and URLs stay the same |
| R3 | For any **replaced** reports: restore the old report in the workspace app; move subscriptions back | |
| R4 | Switch prod to `gold_version = old` (old gold notebooks run again) | If old gold missed cycles, run them now to catch up |
| R5 | Check the key reports against the baseline numbers | |
| R6 | Tell users the rollback is done (§17) | |
| R7 | Root-cause analysis; fix in the feature branch; go back to the right phase (usually Phase 2) | |

**Time to complete a rollback:** use the time measured in the rehearsal (2.12). Fill it in here: ______ minutes.

### 15.3 What rollback does *not* undo

- The new gold lakehouse and new models stay in prod (unused). They're cleaned up later or reused on the next attempt.
- Downstream copies or exports users made from the new models during the cutover window.

## 16. Risk register

| ID | Risk | Likelihood | Impact | Mitigation | Fallback |
|---|---|---|---|---|---|
| R-01 | References to other workspaces still point at dev/test after deploying | Medium | High | Variable library; reference check after every deploy (2.9, 3.2) | Fix the values and redeploy; prod gold isn't consumed until Phase 4 |
| R-02 | Field-name gaps break report visuals | Medium | High | Comparison scripts (A.2); compatibility measures; visual check in B and test | F4-a / F4-b; rollback |
| R-03 | New gold numbers differ from old | Medium | High | ≥ 3-cycle reconciliation on B; variance log with business sign-off | F1-a; rollback |
| R-04 | New models aren't cheaper; A stays throttled | Low–Medium | High | Hard CU gate G1; model hygiene checklist | F1-b: don't promote |
| R-05 | Side-by-side run in prod overloads A | Medium | Medium | Off-peak only; ≤ 2 cycles; surge protection | F3-b |
| R-06 | RLS/OLS not carried over correctly | Low | Critical | Role comparison (1.10); RLS tests with test users in UAT | Immediate rollback |
| R-07 | Item pairing by name creates duplicates or overwrites the wrong item | Low–Medium | High | Naming review in the pull request (2.2); inventory check (2.3) | Restore from the Git baseline |
| R-08 | Region mismatch between A and B blocks Direct Lake or shortcut use | Low | Medium | Check A1 in Phase 0 | Build in A non-prod instead |
| R-09 | Excel / other consumers of the old model break silently | Medium | Medium | Inventory 0.4; owner outreach (4.8) | Keep the old model during hypercare; repoint |
| R-10 | Someone changes the workspace directly in the UI instead of in Git | Medium | Medium | Branch protection; workspace access review; freeze | Resync from Git; review the change |
| R-11 | Key people unavailable during cutover or hypercare | Low | Medium | Named backups (§17); calendar holds | Delay cutover (No-Go G-10) |
| R-12 | Slalom engagement ends before hypercare | Medium | Medium | Handover pack and runbook walk-through before Phase 4 | Internal BI team owns hypercare |

## 17. Communication plan

| When | Audience | Message | Channel | Owner |
|---|---|---|---|---|
| Runbook approved | BI team, platform admin, Slalom | Plan, roles, timeline | Meeting + this document | Data Services |
| Non-prod freeze starts (Phase 2) | Developers on A | Freeze window, contacts | Team channel | BI team |
| 5 business days before cutover | BG1, BG2 report users | What's changing (nothing visible except possible data corrections), cutover date, contacts | Email + app banner | Business owners |
| Go/No-Go | Approvers | Decision and evidence | Meeting; record in the decision log | Data Services |
| Cutover start / end | BG1, BG2, support desk | Started / completed; how to report issues | Email / team channel | BI team |
| Daily during hypercare | Approvers | Status (green/amber/red), incidents | Short status note | BI team |
| Rollback (if needed) | Everyone above | Rollback started / done; next steps | Email / team channel | Data Services |
| Close-out | Everyone above | Done; B decision; lessons learned | Email | Data Services |

**Contacts and backups:** _fill in names_ – Owner: ______ (backup ______) · BI lead: ______ (backup ______) · Platform admin: ______ (backup ______) · BG1 owner: ______ · BG2 owner: ______ · Slalom lead: ______

## 18. Acceptance criteria

Tolerances must be agreed with the business owners before Phase 1 ends. Defaults are suggested below.

| Check | Default tolerance | Applies in |
|---|---|---|
| Row counts per gold table | **Exact match**, unless an explained and signed-off correction | B, test, prod |
| Key totals (amounts, quantities) | **Exact match** to the agreed rounding; any difference explained | B, prod |
| Distinct business keys | Exact match | B, prod |
| Null rate per key column | Same or lower than old gold | B |
| Report key numbers vs baseline | Exact match on the agreed KPI list | Test, prod |
| Model refresh duration | ≤ old model's refresh duration | B, prod |
| CU cost of models (refresh + query) | **≤ old**; target reduction agreed in §19 | B, prod (hypercare) |
| Broken visuals | **Zero** on every page of every in-scope report | B, test, prod |
| RLS test cases | **100% pass** | Test, prod |

## 19. Open decisions

| # | Decision | Options | Owner | Due |
|---|---|---|---|---|
| D-1 | Target CU reduction for the new models | e.g. ≥ 20% / ≥ 30% / any reduction | Data Services + platform admin | Before G1 |
| D-2 | Reconciliation tolerances (confirm §18 defaults) | As listed / custom | Business owners | Before G1 |
| D-3 | Hypercare length | 1 reporting cycle / 2 cycles / 30 days | Data Services + business | Before G4 |
| D-4 | Fix-forward time limit before rollback | e.g. 2 h / 4 h / next business day | Data Services | Before G4 |
| D-5 | Side-by-side run in prod: 1 or 2 cycles, or skip it (F3-b) | | Approvers | Before Phase 3 |
| D-6 | Future of capacity B after close-out | Keep / scale down / pause / delete | Data Services + platform admin | Phase 6 |

---

## Appendix A – Verification scripts

> Run these in a Fabric notebook. Replace the placeholders. **Check them in a non-prod workspace before using them in prod.**

### A.1 Gold reconciliation (Spark SQL / PySpark)

```python
# Compare old and new gold tables: row counts, distinct keys, key totals.
# Both lakehouses can be reached through shortcuts in the workspace running this notebook.
checks = [
    # (old_table, new_table, key_columns, sum_columns)
    ("old_gold.fact_example", "new_gold.fact_example", ["example_key"], ["amount"]),
]

from pyspark.sql import functions as F

results = []
for old_t, new_t, keys, sums in checks:
    o, n = spark.table(old_t), spark.table(new_t)
    row = {"table": new_t,
           "rows_old": o.count(), "rows_new": n.count(),
           "keys_old": o.select(*keys).distinct().count(),
           "keys_new": n.select(*keys).distinct().count()}
    for c in sums:
        row[f"sum_{c}_old"] = o.agg(F.sum(c)).first()[0]
        row[f"sum_{c}_new"] = n.agg(F.sum(c)).first()[0]
    results.append(row)

display(spark.createDataFrame(results))

# Rows present in one table but not the other (on keys) - check these by hand:
# display(o.select(*keys).exceptAll(n.select(*keys)))
# display(n.select(*keys).exceptAll(o.select(*keys)))
```

### A.2 Semantic model field comparison (semantic link)

```python
import sempy.fabric as fabric

def model_objects(dataset, workspace):
    cols = fabric.list_columns(dataset=dataset, workspace=workspace)
    meas = fabric.list_measures(dataset=dataset, workspace=workspace)
    c = set(cols["Table Name"] + "[" + cols["Column Name"] + "]")
    m = set("[" + meas["Measure Name"] + "]")
    return c | m

old = model_objects("<old model name>", "<old model workspace>")
new = model_objects("<new model name>", "<new model workspace>")

missing_in_new = sorted(old - new)   # likely to break reports unless they're unused
print(f"{len(missing_in_new)} objects in OLD model not found in NEW model:")
for x in missing_in_new:
    print("  ", x)
```

> Column names in the `list_columns` / `list_measures` output can differ between semantic-link versions. Check them with `.columns` first. For report-level field usage, use the report-inspection helpers in **semantic-link-labs** (`sempy_labs.report`), or check each report by hand in the visual check (1.7).

### A.3 Rebinding a report

**Option 1 – semantic-link-labs (notebook)**

```python
import sempy_labs.report as rep

rep.report_rebind(
    report="<report name>",            # or a list of report names
    dataset="<target model name>",
    report_workspace="<report workspace>",
    dataset_workspace="<model workspace>",
)
```

**Option 2 – Power BI REST API**

```http
POST https://api.powerbi.com/v1.0/myorg/groups/{reportWorkspaceId}/reports/{reportId}/Rebind
Content-Type: application/json

{ "datasetId": "<target semantic model id>" }
```

For **rollback**, use the same call with the **old** model's ID from the Phase 0 inventory.

### A.4 Checking references after a deploy (by hand)

For every stage you deploy to, confirm and record (with screenshots):

1. Each new-gold lakehouse **shortcut** → that stage's silver source
2. Each notebook's **default lakehouse** → that stage's lakehouse
3. Each **elt notebook activity** → that stage's transf notebook (workspace ID + notebook ID)
4. Each **semantic model** data source / parameter → that stage's new-gold
5. **Variable library** values match the expected values for that stage

---

## Appendix B – References

- [Capacity reassignment restrictions and common issues](https://learn.microsoft.com/fabric/admin/portal-workspace-capacity-reassignment)
- [Fabric capacity and OneLake consumption – shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview – capacity guardrails and limitations](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Fabric deployment pipelines](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/intro-to-deployment-pipelines)
- [Fabric Git integration](https://learn.microsoft.com/fabric/cicd/git-integration/intro-to-git-integration)
- [Power BI REST API – Rebind Report In Group](https://learn.microsoft.com/rest/api/power-bi/reports/rebind-report-in-group)
- [Semantic link labs (GitHub)](https://github.com/microsoft/semantic-link-labs)
- [fabric-cicd (GitHub)](https://github.com/microsoft/fabric-cicd)
