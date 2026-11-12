# Runbook: Promoting the New Gold Layer and Semantic Models to Production

> **New to this?** Start with the 5-minute [overview](fabric-migration-overview.md).

| | |
|---|---|
| **Document type** | Migration runbook |
| **Version** | 2.2 (adds Fabric tooling and Azure DevOps standards from the tooling review) |
| **Status** | Draft – for review |
| **Last updated** | 2026-11-12 |
| **Owner** | Data Services (Enterprise Technology) |
| **Approvers** | BI team lead · Platform/capacity admin · Business owners (BG1, BG2) |
| **Applies to** | Microsoft Fabric capacity A (prod + non-prod) and capacity B (development area) · Version control: **Azure DevOps (ADO) Repos** |

---

## Table of contents

1. [Purpose and scope](#1-purpose-and-scope)
2. [Current and target state](#2-current-and-target-state)
3. [Assumptions and constraints](#3-assumptions-and-constraints)
4. [Roles and responsibilities](#4-roles-and-responsibilities)
5. [Guiding principles](#5-guiding-principles)
5a. [Running both stacks side by side](#5a-running-both-stacks-side-by-side)
5b. [Fabric and Azure DevOps tooling standards](#5b-fabric-and-azure-devops-tooling-standards)
6. [Phase overview](#6-phase-overview)
7. [Phase 0 – Readiness and baseline](#phase-0--readiness-and-baseline)
8. [Phase 1 – Build and validate on capacity B](#phase-1--build-and-validate-on-capacity-b)
9. [Phase 2 – Merge into the non-prod lanes on capacity A](#phase-2--merge-into-the-non-prod-lanes-on-capacity-a)
10. [Phase 3 – Run the new gold in prod without consumers](#phase-3--run-the-new-gold-in-prod-without-consumers)
11. [Phase 4 – Promote the semantic models and rebind the reports (per wave)](#phase-4--promote-the-semantic-models-and-rebind-the-reports-per-wave)
12. [Phase 5 – Cutover and hypercare (per wave)](#phase-5--cutover-and-hypercare-per-wave)
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
- **Rebinding the existing prod reports** to the new semantic models, **in waves**
- Running the old and new stacks side by side until every report has moved
- Retiring the old gold items and old semantic models once the **last** wave is done

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

### 2.2 Interim state: both stacks running side by side

The new gold layer and new semantic models **won't serve every report at first**. Until the last wave is done, both stacks run in prod on A:

| | **Old stack** | **New stack** |
|---|---|---|
| Gold | Old gold lakehouse + old notebooks (still running) | New gold lakehouse + new notebooks |
| Semantic models | Old models (still refreshing) | New models |
| Reports | Reports not yet moved | Reports moved in completed waves |
| Orchestration | Old path in elt (on) | New path in elt (on) |

Both stacks read the same **prod silver**. They are run, monitored and rolled back **per wave** (see §5a). While both run, A carries the cost of both, so the CU budget in §5a.4 applies.

### 2.3 Target state

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
| A5 | **Known behaviour (confirmed in Microsoft docs):** references to items in **another workspace** never rebind automatically; they stay pointed at the source object ID (elt → transf notebooks, cons model → transf lakehouse, any shortcut whose target is in another workspace) | Microsoft docs; test deploy in Phase 2 | – (design accounts for it: §5b.3) |
| A6 | The new semantic models can serve the existing reports **once field names are reconciled** | Field comparison script, Appendix A.2 | Compatibility measures, or report changes, or replacement reports (Phase 4 fallbacks) |
| A7 | Reads through a shortcut are charged to the **capacity of the workspace doing the reading** | Microsoft docs; Capacity Metrics app | – |
| A8 | Prod has a known **refresh/ETL window** and an off-peak window | Pipeline schedules; Capacity Metrics app | Agree windows with the business before Phase 3 |
| A9 | Slalom has **no write access** to prod workspaces | Workspace access lists | Remove access before Phase 3 |
| A10 | **Known behaviour:** a **Direct Lake** semantic model **doesn't** rebind on deployment; it stays bound to the source stage's lakehouse unless rules are set (data source rules for Direct Lake on SQL; M parameters + parameter rules for Direct Lake on OneLake) | Microsoft docs; check after every deploy (A.5) | – (design accounts for it: §5b.4) |
| A11 | elt, transf, cons-bg1 and cons-bg2 deployment pipelines all have the **same number of stages** (needed for binding across pipelines, which uses stage position, not stage name) | Deployment pipelines list | Align stage counts before Phase 2 |
| A12 | Fabric workspaces connect to **Azure DevOps Repos** through the Azure DevOps source-control connection, using a **service principal** (not a personal account) | Workspace Git settings; connection owner | Create the service principal connection in Phase 0 |
| A13 | A **semantic model refresh** activity in a pipeline refers to the model by item ID, **even within the same workspace**, so it never rebinds | Microsoft docs | – (pass the ID in from a variable library: §5b.3) |

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

1. **One route to prod.** Every prod change goes Azure DevOps (Git) → non-prod on A → deployment pipeline (started from ADO, with approvals) → prod. `trans-dev-2` is never promoted directly.
2. **Validate where it's cheap.** Heavy reconciliation and load testing run on B. A only gets smoke tests and the final run.
3. **Add before you replace.** New items run **alongside** the old ones until cutover. Old items are disabled, not deleted, until hypercare ends.
4. **Keep report identity.** Rebind the existing reports instead of replacing them, so URLs, apps, bookmarks and subscriptions survive.
5. **Every phase has a gate.** No phase starts until the previous phase's exit criteria are met and recorded.
6. **Every step can be reversed.** Each phase lists its fallback. The rollback in §15 is **rehearsed in test** before cutover.
7. **No hard-coded environment values.** Workspace, lakehouse and notebook IDs and paths come from variable libraries or deployment rules.
8. **Running both stacks has an end date.** Both run only as long as needed, against a planned end date and a CU budget (§5a). Each old item is retired **as soon as nothing depends on it**, not all at once at the end.
9. **The two stacks must agree.** While both run, a KPI shown in an old-stack report and in a new-stack report must give the same number, or the difference must be documented and explained to users.

## 5a. Running both stacks side by side

### 5a.1 Wave plan

Every in-scope prod report is assigned to a **wave** in the Phase 0 report inventory (step 0.4). Recommended order:

| Wave | Content | Purpose |
|---|---|---|
| **Wave 1 (pilot)** | 1–3 reports with low risk but real use, ideally one per business group | Prove the end-to-end path, the rebind, the rollback and the cost of running both stacks in prod |
| **Wave 2..n** | Groups of reports by **subject area / semantic model**, highest CU savings first | Bring the CU reduction as early as possible |
| **Final wave** | The rest, including reports that need replacing (F1-c) | Lets the old stack be retired completely |

Rules:

- A report moves **as a whole**. It is either on the old model or the new one, never split.
- Reports that share a model, a workspace app or a shared dataset with Excel users move **together** where possible.
- Each wave repeats **Phase 4 → Phase 5**, with its own Go/No-Go, hypercare and rollback. Phases 0–3 run once; later waves only need Phase 1–2 again if they bring new gold tables or model changes.

### 5a.2 Dependency map

Keep a **dependency map** (in the repo next to the inventory) and update it at the end of every wave:

| Old item | Used by (old models / reports) | Replaced by (new item) | Wave in which it becomes unused | Retired? |
|---|---|---|---|---|
| old_gold.fact_x | old_model_bg1 → reports R1, R2 | new_gold.fact_x / new_model_bg1 | Wave 2 | ☐ |

An old item can be retired **only when its "Used by" column is empty** (checked with the lineage view and the report inventory). This means old notebooks, gold tables and models are retired **gradually**, not only at the end.

### 5a.3 Orchestration while both stacks run

- In elt, keep the old and new paths as **separate branches** of the orchestration, each with its own on/off flag by **subject area** (e.g. variable-library values `run_old_<area>`, `run_new_<area>`), instead of one global `gold_version` switch.
- Both paths depend on **silver being finished**; neither depends on the other. A failure in one path **must not stop the other** (set the activity dependencies to "on completion" or use separate child pipelines, as appropriate).
- Schedule the **heavier path first** in the off-peak window, or spread the two paths out, so their CU peaks don't land on top of each other.
- Alerts are set up for **both** paths, with the owning stack shown in the alert name.

### 5a.4 CU budget while both stacks run

- Before Wave 1, measure the **combined** CU of both stacks (Phase 3 side-by-side run) and set a **budget**: a ceiling on A's peak and average use, below the throttling threshold.
- Track it **weekly** in the Capacity Metrics app. If the budget is exceeded:
  1. Move to the next wave sooner (to retire old items sooner), **or**
  2. Lower refresh frequency on the old models whose reports move in the next wave, **or**
  3. **Converge:** where a new gold table can stand in for an old one (same grain, same business logic, field names matched), point the old model at the new gold table and **retire the old notebook early** (see 5a.5), **or**
  4. Escalate to the approvers for a temporary capacity scale-up for the rest of the period.

### 5a.5 Pointing old models at new gold (optional)

To cut duplicate compute, an **old** model can be pointed at a **new** gold table **before** its reports move, as long as all of these are true:

- The new table has the **same grain and business logic**, and reconciliation (Appendix A.1) passed with exact match
- The column names the old model needs exist (directly or through a view or shortcut set up to match)
- The change is deployed through the normal pipeline (dev → test → prod) and the old model's reports are checked

This is optional. Use it only where the CU budget needs it, because it adds one more change to keep track of.

### 5a.6 Keeping the numbers the same across both stacks

- Keep a **KPI cross-check list**: KPIs that show up in both old-stack and new-stack reports. After every load, run an automated check that the old and new numbers match, or report the difference (Appendix A.1 run against model-level KPIs, or a DAX query against both models).
- **Mark which model to use:** mark new models as **Promoted** (later **Certified**) once their wave is done; add "(legacy)" to the description of old models that are being retired, and don't allow new reports to be built on them.
- If an **intended correction** in the new stack changes a KPI (logged in Phase 1.4), tell users of **both** stacks before the wave goes live, so the difference isn't reported as a defect.

### 5a.7 Planned end date

- The approvers set an **end date** for running both stacks (decision D-7).
- If it slips by more than one wave, Data Services escalates with a revised plan and CU forecast.

## 5b. Fabric and Azure DevOps tooling standards

These standards come from the Fabric tooling review. They are **mandatory**, and every pull request and every deployment is checked against them.

### 5b.1 Repository and workspace layout (ADO)

- **One ADO repo**, with **one folder per workspace**: `/elt`, `/transf`, `/cons-bg1`, `/cons-bg2`. Each dev workspace on A connects to **its own folder** on the `dev` branch.
- **A Fabric workspace syncs to exactly one branch and folder.** A single Slalom workspace that mixes notebooks, lakehouses, pipelines and models **can't** sync into this layout. Slalom therefore works in **one branched workspace per target folder** (for example `trans-dev-2-transf`, `trans-dev-2-elt`, `trans-dev-2-cons-bg1`, `trans-dev-2-cons-bg2`) on capacity B, each connected to the same feature branch in the matching folder. The existing `trans-dev-2` content is moved into these workspaces (by committing it to the right folders) before Phase 2.
- Git connections use the **Azure DevOps source-control connection with a service principal**, so that sync and commit don't depend on one person's account.
- **No edits in the UI** of dev workspaces on A while the migration runs: a Git update overwrites workspace items with what's in the branch. All changes go through a branch and pull request.

### 5b.2 Naming

Deployment pipelines pair items **by name and type**, and both stacks live in the same workspaces. New items **must** have names that can never clash with old ones:

| Item | Old (example) | New (required pattern) |
|---|---|---|
| Gold lakehouse | `lh_gold` | `lh_gold_v2` |
| Gold notebooks | `nb_gold_<area>` | `nb_gold_v2_<area>` |
| Semantic models | `sm_bg1_<area>` | `sm_bg1_<area>_v2` |
| Table setup notebook | – | `nb_gold_v2_setup` |
| Variable library | – | `vl_env` (one per workspace) |

Renaming an existing item is **not allowed** during the migration: in the next stage it would show up as a new item next to the old one.

### 5b.3 Environment values: variable libraries

Each workspace has a variable library (`vl_env`) with **value sets for dev, test and prod**. The active value set is chosen per stage. What goes where:

| Reference | Mechanism |
|---|---|
| Notebook default lakehouse | **Either** turn on *Lakehouse Auto-Binding in Git* in **every** notebook's settings (off by default), **or** set it with `%%configure` reading variable-library name/ID variables. Choose **one** approach for the whole solution (decision D-12) and check it in code review. *(`%%configure` doesn't support item-reference variables; use name and ID variables.)* |
| Lakehouse paths and IDs inside notebook code | `notebookutils.variableLibrary` (item-reference or name/ID variables). No ABFS paths with literal workspace or lakehouse names |
| **OneLake shortcuts** whose target is in **another workspace** (e.g. new-gold → silver) | Shortcut defined with **variable-library item references**. Shortcuts within the same workspace are repointed per stage automatically |
| elt pipeline → transf notebook activities (another workspace) | **GUID variables** for workspace ID and notebook ID, read by the pipeline |
| Pipeline **semantic model refresh** activities | **GUID variables** for workspace ID and model ID (never rebinds, even in the same workspace, per A13) |
| Direct Lake model → gold lakehouse | See §5b.4 |

### 5b.4 Direct Lake binding (hard gate)

A Direct Lake model **stays bound to the lakehouse in the source stage** when deployed (A10). Your models are also in a different workspace from the gold lakehouse, which never rebinds (A5). So:

- **Direct Lake on SQL endpoint:** create a **data source rule** for each stage, pointing at that stage's `lh_gold_v2` SQL endpoint.
- **Direct Lake on OneLake:** build the model with **M parameters** for the workspace and lakehouse, and create **parameter rules** (Text type) for each stage.
- **After every deployment**, run the Direct Lake source check (Appendix A.5). **A model whose source isn't its own stage's lakehouse fails the gate**, and the deployment is not finished until it's corrected.

### 5b.5 Lakehouse deployment behaviour

- Deployment and Git sync carry **lakehouse metadata and shortcut definitions only**. **Tables, Files folders and data are never deployed.** `lh_gold_v2` arrives empty in each stage.
- **Deployments and Git updates overwrite the target lakehouse's shortcuts:** shortcuts that aren't in the source are **deleted**. Never create shortcuts in test or prod by hand. **Review the changes in the deployment pipeline before every deploy.**
- `nb_gold_v2_setup` is an **idempotent** notebook that creates every gold table with its schema, partitioning and table properties (V-Order and so on). It runs **once per stage before the first load**, and again whenever a table definition changes.
- RLS stays in the **semantic models**. OneLake security data access roles are in preview and aren't used for this migration.

### 5b.6 Promotion with Azure DevOps

**Chosen route (default):** Git (ADO) for dev → **Fabric deployment pipelines** for test and prod, **started from an ADO pipeline** through the Fabric deployment pipelines REST API.

- **ADO branch policies** on `dev`: pull request required; at least one internal reviewer (BI team); linked work item; **build validation** that fails on hard-coded GUIDs, workspace names, ABFS paths or B references in item definitions.
- **ADO environments** `fabric-test` and `fabric-prod` with **approvals and checks** (prod: Data Services + business owner for wave deployments). The ADO pipeline:
  1. Starts the deployment for the selected items and stage through the REST API.
  2. Runs the post-deploy checks (A.4 references, A.5 Direct Lake source, A.3 report bindings).
  3. On a prod deployment, **tags the release** in ADO (e.g. `release/wave-1-yyyymmdd`), so there is a record of what runs in prod.
- **Alternative (decision D-11):** deploy straight from ADO with **`fabric-cicd`** (using the Fabric automation tools ADO extension), with release branches and a `parameter.yml` per environment. Prod then has full Git-based version history. **Use one route, not both,** on the same workspaces.
- Cross-pipeline binding (cons reports ↔ models) relies on matching **stage positions** (A11).

### 5b.7 Inventory, monitoring and alerts

- The **report inventory and dependency map** (0.4, 0.4a, §5a.2) are built from the **admin scanner API** (lineage across all workspaces), using semantic-link-labs or a scheduled notebook. They're refreshed at the end of each wave. This replaces hand-built lineage lists.
- **Alerts** on both orchestration branches and on old and new model refreshes, through **Activator** / the monitoring hub. Each alert name shows which stack it belongs to.
- **Capacity Metrics app** for the CU baseline, budget and per-wave comparisons.

## 6. Phase overview

| Phase | Name | Capacity | Gate to exit |
|---|---|---|---|
| 0 | Readiness and baseline | A, B | Assumptions checked; baselines captured; Git ready |
| 1 | Build and validate on B | B | Reconciliation passed; CU cost of new models ≤ old; field comparison clean |
| 2 | Merge into non-prod lanes on A | A (non-prod) | Merged; parameters set; smoke test and UAT passed in test |
| 3 | Run the new gold in prod without consumers | A (prod) | New gold built in prod; reconciles against old gold |
| 4 | Promote semantic models and rebind reports — **per wave** | A (prod) | The wave's reports rebound and visually checked |
| 5 | Cutover and hypercare — **per wave** | A (prod) | Hypercare passed for the wave; newly unused old items retired |
| ↻ | **Repeat 4–5 for each wave** (and 1–2 again if a wave brings new gold or model changes) | | Both stacks running, within the CU budget |
| 6 | Retire the rest of the old items and close out — **after the final wave** | A, B | Old stack fully retired; environments in sync; decision on B recorded |

---

## Phase 0 – Readiness and baseline

**Objective.** Confirm the assumptions, record the "before" picture, and prepare source control.

**Entry criteria**

- This runbook is approved by the approvers listed at the top
- Roles in §4 have named people and backups

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 0.1 | Check assumptions A1–A13 (§3) and record each as passed or failed | Data Services | Completed §3 table |
| 0.2 | **Record the CU baseline** for A: 14-day peak, average, and throttling events, broken down by item (models, notebooks, pipelines) from the Capacity Metrics app | Platform admin | Metrics export |
| 0.3 | **Record the data baseline:** row counts, key totals and distinct key counts for every gold table in scope (Appendix A.1) | BI team | Baseline file in the repo |
| 0.4 | **Record the report inventory:** every prod report bound to the old models, with workspace, report ID, model ID, app membership, subscriptions, Excel/Analyze-in-Excel use, and **its assigned wave** (§5a.1) | BI team | Inventory sheet with wave plan |
| 0.4a | Build the **dependency map** (§5a.2) from old gold tables → old models → reports, using the **admin scanner API** (§5b.7) | BI team | Dependency map in the repo |
| 0.4b | Build the **KPI cross-check list** (§5a.6) of KPIs that will appear in both stacks | BI team + business | KPI list |
| 0.5 | **Record the schema of the old models:** tables, columns, measures, relationships and roles (RLS) | BI team | Model schema export (Appendix A.2) |
| 0.6 | **Back up the old models** (export the definitions to Git/PBIP; for Import models also take an XMLA/.abf backup if allowed) | BI team | Backup location recorded |
| 0.7 | Connect the existing non-prod (dev) workspaces on A to their **ADO repo folders** (§5b.1) with the **service principal connection**, and **commit the baseline** | BI team | Baseline commit hash |
| 0.8 | Create Slalom's **branched workspaces, one per target folder** (§5b.1) on B, connected to `feature/slalom-gold-v2`. Move the existing `trans-dev-2` content into them by committing it to the right folders | Slalom + BI team | Branch and workspace list |
| 0.9 | Set **ADO branch policies** on `dev` (§5b.6): pull request required, internal reviewer, linked work item, build validation for hard-coded values | BI team | Repo policy settings |
| 0.10 | Remove any Slalom write access to prod workspaces (A9) | Platform admin | Access review |
| 0.11 | Agree the change windows (deployment, smoke test, cutover) and the freeze period | Data Services + business | Calendar holds |
| 0.12 | Set up **ADO environments** `fabric-test` / `fabric-prod` with approvals, and the **ADO deployment pipeline** that calls the Fabric REST API and runs the post-deploy checks (§5b.6). Do a dry run on a trivial item | BI team + platform admin | Pipeline run log |
| 0.13 | Check that all four deployment pipelines have the **same number of stages** (A11) | BI team | Screenshot |
| 0.14 | Create the **variable library** `vl_env` in each workspace with dev, test and prod value sets (§5b.3) | BI team | Committed variable library |
| 0.15 | Record decisions **D-11** (promotion route) and **D-12** (notebook default lakehouse approach) | Data Services | Decision log |

**Exit criteria (gate G0)**

- [ ] A1–A13 recorded; any failed item has a decision and this runbook is updated
- [ ] CU, data, report and schema baselines are stored in the repo or a shared location
- [ ] Old models backed up
- [ ] Git baseline committed; Slalom's branched workspaces (one per folder) in place; ADO branch policies, environments and deployment pipeline dry run done
- [ ] Variable libraries created; stage counts match; D-11 and D-12 recorded

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
| 1.11 | Remove all hard-coded references to B (default lakehouse, GUIDs, ABFS paths). Every environment value follows **§5b.3**; the notebook default lakehouse follows the D-12 approach; new items follow the **naming rule** (§5b.2); Direct Lake models use the binding method in §5b.4 | Slalom | Code review checklist; ADO build validation passes |
| 1.12 | Build and test `nb_gold_v2_setup` (§5b.5): run it twice in a row on an empty lakehouse and confirm the second run changes nothing | Slalom | Run log |

**Exit criteria (gate G1)**

- [ ] Reconciliation passed for ≥ 3 cycles within the agreed tolerance (§18), with every variance explained and signed off
- [ ] **CU cost of the new models ≤ the old models** at the same load (target: a measurable reduction agreed in §19)
- [ ] Field-name gap list closed; every report copy renders with no broken visuals
- [ ] RLS/OLS roles match
- [ ] No hard-coded references to B in the code; ADO build validation passes; naming rule followed

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
| 2.1 | Slalom opens **ADO pull requests** from `feature/slalom-gold-v2` into `dev`, one per folder (`/transf`, `/elt`, `/cons-bg1`, `/cons-bg2`) | Slalom | Pull request links |
| 2.2 | **Code review** against §5b: notebook diffs, gold schema changes, naming (§5b.2), variable-library use (§5b.3), Direct Lake binding method (§5b.4), no secrets, no B references; ADO build validation green | BI team | Approved pull requests |
| 2.3 | Merge and **update each dev workspace on A from Git**. Confirm item counts and names match what's expected, and that **no existing item was renamed or overwritten** | BI team | Sync log; item list |
| 2.4 | Fill in the `vl_env` **value sets** for dev, test and prod (§5b.3): workspace IDs, lakehouse item references, notebook IDs used by elt, model IDs used by refresh activities, shortcut targets | BI team | Committed variable library |
| 2.5 | Set up the **deployment rules** each stage needs: Direct Lake **data source rules** (on SQL) or **parameter rules** (on OneLake) per §5b.4; notebook rules if D-12 requires them | BI team | Rule screenshots |
| 2.5a | Run `nb_gold_v2_setup` in **dev**, then in **test** after the first deployment (§5b.5) | BI team | Run log |
| 2.6 | In **elt-dev**, add the new notebook activities as a **separate branch** of the orchestration. **Keep** the old branch. Control each with on/off flags **by subject area** (`run_old_<area>`, `run_new_<area>`) from the variable library. A failure in one branch must not stop the other (§5a.3) | BI team | Pipeline definition |
| 2.7 | Rebind the **dev copies** of the existing reports to the new models (Appendix A.3) | BI team | Rebind log |
| 2.8 | Deploy **dev → test** **through the ADO deployment pipeline** (§5b.6), in this order: **transf → elt → cons**. **Review the changes** (especially shortcut changes) before approving | BI team | ADO run; deployment history |
| 2.9 | **Post-deploy checks** (run by the ADO pipeline, and confirmed by a person): references (A.4), **Direct Lake source (A.5)**, report bindings (A.3). Everything must point at **test** items | BI team | Check output |
| 2.10 | **Smoke test in test** at **off-peak** hours: one full orchestration run with **both branches on**, a light reconciliation (row counts only), and a refresh of both an old and a new model. Also test that a forced failure in one branch leaves the other branch running | BI team | Run results |
| 2.11 | **UAT in test** by each business group, using the agreed UAT script (key reports, key numbers, RLS spot-checks with test users) | Business owners | UAT sign-off |
| 2.12 | **Rehearse the wave rollback** (§15) in test: rebind the Wave 1 reports to the old model, confirm they work, then rebind them back to new | BI team | Rehearsal log with timings |

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
| 3.1 | Deploy **test → prod** **through the ADO deployment pipeline** (prod approvals): **transf**, then **elt**, with every `run_new_<area>` flag **off**. Review the shortcut changes before approving. Tag the release (§5b.6) | BI team | ADO run; release tag |
| 3.2 | **Post-deploy checks** (A.4, A.5): prod notebooks read **prod silver** and write to prod `lh_gold_v2`; elt activities and refresh activities point at **prod** items through `vl_env` | BI team | Check output |
| 3.3 | Run `nb_gold_v2_setup` in **prod** (§5b.5), then the **initial load or backfill** of new gold off-peak | BI team | Run log; duration; CU used |
| 3.4 | Turn on the `run_new_<area>` flags for the Wave 1 subject areas. **From here on, both branches run in prod** (§2.2). Watch CU on A for at least 2 cycles | BI team + platform admin | Metrics screenshots |
| 3.5 | **Reconcile prod new-gold against prod old-gold** (Appendix A.1) and run the KPI cross-check (§5a.6) | BI team | Reconciliation report |
| 3.6 | **Set the CU budget** for running both stacks (§5a.4) from the measured combined CU, and get it approved | Platform admin + Data Services | Approved budget |

**Exit criteria (gate G3)**

- [ ] New gold built in prod; reconciliation and KPI cross-check pass within tolerance (§18)
- [ ] CU on A with both stacks running stayed below the throttling threshold; CU budget approved
- [ ] No consumer reads from new gold yet

**Fallback**

- **F3-a (deploy or reference error):** Turn the `run_new_<area>` flags off. Nothing in prod reads new gold, so **users are unaffected**. Fix in dev and redeploy.
- **F3-b (combined CU too high):** Reschedule the branches so their peaks don't overlap (§5a.3). If it's still too high, take the options in §5a.4 to the approvers **before** Wave 1: a smaller first wave, lower old-model refresh frequency, pointing old models at new gold (§5a.5), or a temporary scale-up.
- **F3-c (reconciliation fails in prod but passed on B):** Look first for environment differences: silver source, shortcut targets, parameters. Stay in Phase 3 until it's resolved.

---

## Phase 4 – Promote the semantic models and rebind the reports (per wave)

**Objective.** For **the current wave only**, put the new semantic models in prod (first wave) or extend them (later waves), and move **that wave's** existing reports onto them without changing report identity. Reports in later waves stay on the old models.

**Entry criteria**

- Gate G3 passed (first wave) or gate G5 of the previous wave passed (later waves)
- If the wave brings new gold tables or model changes: Phases 1–2 repeated for that scope
- **Go/No-Go for the wave (§14) approved**
- Wave communication sent (§17)

**Steps (recommended path: rebind in dev, deploy together)**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 4.1 | Confirm the cons-test stage holds the new models and **only this wave's** reports rebound (from 2.7/2.8). Reports in later waves must still be bound to the **old** model in test | BI team | Item list with each report's binding |
| 4.2 | Deploy **cons-bg1 test → prod** **through the ADO deployment pipeline** (prod approvals), selecting **only** the new semantic model and **this wave's** reports. Reports bound to the new model are rebound by the pipeline (same workspace). **Don't select reports from later waves.** Tag the release | BI team | ADO run; list of selected items; release tag |
| 4.3 | **Direct Lake source check (A.5): the prod model must read prod `lh_gold_v2`.** If it reads the test lakehouse, fix the rule and redeploy; the wave doesn't go ahead until this passes | BI team | Check output |
| 4.4 | **Refresh (or reframe) the new prod model**; confirm it succeeds and record how long it takes | BI team | Refresh history |
| 4.5 | **Check each rebound report:** open every page, compare key numbers with the baseline (0.3) **and with the matching old-stack reports** (KPI cross-check, §5a.6), test RLS with test users | BI team + business | Check log |
| 4.6 | Check that **reports not in this wave are still bound to the old models** and working | BI team | Binding check (Appendix A.3, list bindings) |
| 4.7 | Repeat 4.2–4.6 for **cons-bg2** if this wave includes its reports | BI team | As above |
| 4.8 | For any reports in this wave marked for **replacement** (F1-c): publish the replacement, update the **workspace app**, move subscriptions | BI team | App version; subscription list |
| 4.9 | Repoint **Excel / Analyze-in-Excel** connections that belong to this wave (inventory 0.4); tell the owners | BI team | Owner confirmations |
| 4.10 | Mark the new model **Promoted** once the first wave is live; update the old model's description to show which reports are still on it | BI team | Endorsement setting |

**Exit criteria (gate G4, per wave)**

- [ ] This wave's reports bound to the new models (or replaced, per F1-c); **all other reports still on the old models and working**
- [ ] Every page of every report in the wave renders; key numbers match within tolerance in both stacks; RLS checked
- [ ] Excel and other connected consumers in this wave repointed, or their owners told

**Fallback**

- **F4-a (pipeline didn't rebind the prod report):** Rebind directly in prod with the **Rebind Report In Group** REST API or `sempy_labs.report.report_rebind` (Appendix A.3). Then make the same change in dev so dev and prod stay in step.
- **F4-b (a report can't be rebound cleanly):** Move it to a **later wave** and leave it on the old model, or publish a replacement report, update the workspace app, move subscriptions, and tell its users.
- **F4-c (new model refresh fails in prod):** Rebind **this wave's** reports back to the old model (§15.2). The old models keep refreshing the whole time both stacks run, so users still see current data.
- **F4-d (a report from a later wave was rebound by mistake):** Rebind it back to the old model right away (Appendix A.3) and correct the selection in test before the next deploy.

---

## Phase 5 – Cutover and hypercare (per wave)

**Objective.** Watch the wave closely through hypercare, then **retire only the old items that nothing uses any more** (§5a.2). The old stack keeps serving the reports that haven't moved yet.

**Entry criteria**

- Gate G4 for this wave passed

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 5.1 | Turn on **surge protection** on A (first wave), if not already on | Platform admin | Capacity settings |
| 5.2 | **Hypercare for the wave** – at least **one full reporting cycle** (including a month-end if possible): check daily that **both** orchestration branches succeed, old and new model refreshes succeed, the KPI cross-check passes, CU stays within the budget (§5a.4), and no business incidents come in | BI team | Daily hypercare log |
| 5.3 | **Update the dependency map** (§5a.2): mark the old models, gold tables and notebooks that **nothing uses any more** after this wave | BI team | Updated map |
| 5.4 | For the newly unused old items: **pause refresh** on old models with no remaining reports, and turn off `run_old_<area>` for old gold areas with no remaining users. **Disable, don't delete**, for the rest of the rollback period | BI team | Refresh settings; flag values |
| 5.5 | Compare CU on A with the baseline (0.2) and the CU budget at the end of hypercare | Platform admin | Before/after report |
| 5.6 | Business owners confirm the wave worked | Business owners | Written sign-off |
| 5.7 | Plan the next wave: confirm its scope, dates and Go/No-Go; check the planned end date (§5a.7) | Data Services | Updated wave plan |

**Exit criteria (gate G5, per wave)**

- [ ] Hypercare complete with **no open Sev-1 or Sev-2** incidents
- [ ] CU on A within the budget for running both stacks (or at or below the final target after the last wave, §18)
- [ ] Dependency map updated; newly unused old items disabled
- [ ] Business sign-off received

**Fallback**

- **F5-a:** Any rollback trigger in §15.1 → run the **wave rollback** (§15.2). Earlier waves that are stable stay on the new stack.
- **F5-b (CU budget exceeded):** Apply §5a.4.

**→ Repeat Phases 4–5 for each wave. Once the final wave passes G5, go to Phase 6.**

---

## Phase 6 – Retire the old items and close out

**Objective.** After the **final wave**, remove what is left of the old stack, bring dev, test and prod back in step, and decide what happens to capacity B. (Old items that became unused in earlier waves can be deleted at the end of each wave's rollback period through the same steps 6.1–6.3, using the dependency map.)

**Entry criteria**

- Gate G5 passed for the **final wave**. Every report in the inventory is on the new stack, and the dependency map shows nothing using the old items (the rollback option ends here; record this in the decision log)

**Steps**

| # | Action | Owner | Evidence |
|---|---|---|---|
| 6.1 | Take a final backup of the old model definitions and an export of the old gold table schemas | BI team | Backup location |
| 6.2 | In **dev**: remove the old orchestration branch and the `run_old_<area>` / `run_new_<area>` flags from elt; delete the old gold notebooks, old gold lakehouse and old models. Commit through a pull request | BI team | Pull request link |
| 6.3 | Deploy dev → test → prod so the deletions reach every stage. Confirm the prod item inventory | BI team | Deployment history; inventory |
| 6.4 | Check that **no prod or non-prod item on A references B**, and that nothing references the old gold or old models | BI team | Lineage view check |
| 6.4a | Mark the new models **Certified** (if your governance process allows it); retire the KPI cross-check | BI team | Endorsement setting |
| 6.5 | **Decide B's future:** keep it for future development, scale it down, or pause it between engagements. A paused capacity still pays for stored data, so clean up `trans-dev-2` if it's no longer needed | Data Services + platform admin | Decision log entry |
| 6.6 | Update architecture documentation and this runbook's status to **Complete** | Data Services | Updated docs |
| 6.7 | Lessons-learned review with Slalom and the BI team | Data Services | Notes |

**Exit criteria (gate G6)**

- [ ] Old stack fully removed in every stage; environments in sync
- [ ] Nothing on A depends on B
- [ ] Decision on B recorded; documentation updated

---

## 14. Go/No-Go checklist

Must be completed and signed **before each wave's Phase 4**. For later waves, criteria G-1 to G-3 apply to that wave's scope. A single "No" means **No-Go**.

| # | Criterion | Yes/No | Evidence |
|---|---|---|---|
| G-1 | Gates G0–G3 passed (first wave) or the previous wave's G5 passed (later waves) | | |
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
| G-12 | Capacity A has headroom for the window (Metrics app check on the day), and the CU budget for running both stacks is being met | | |
| G-13 | This wave's report list is final; **reports from other waves have been checked to still be on the old model in test** | | |
| G-14 | KPI cross-check passing; any intended differences explained to users of **both** stacks | | |
| G-15 | **Direct Lake source check (A.5) passed in test** for every model in this wave, and the prod rules are set up | | |
| G-16 | ADO deployment pipeline, approvals and post-deploy checks worked in the test deployment for this wave | | |

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

### 15.2 Rollback procedure (per wave)

The rollback covers **one wave at a time**. Earlier waves that are stable stay on the new stack. Rehearsed in step 2.12.

| Step | Action | Notes |
|---|---|---|
| R1 | Confirm the old model for the affected reports is **refreshing and current**. If it was paused after an earlier wave (5.4), **resume refresh** and run one refresh now | While both stacks run, the old models normally keep refreshing |
| R2 | If the old gold area behind that model was turned off (5.4), turn `run_old_<area>` back on and run the catch-up loads | Run missed cycles in order |
| R3 | **Rebind this wave's prod reports back to the old models** (Appendix A.3, with the old model IDs from inventory 0.4) | Report IDs and URLs stay the same |
| R4 | For any **replaced** reports in this wave: restore the old report in the workspace app; move subscriptions back | |
| R5 | Check the key reports against the baseline numbers | |
| R6 | Tell users the rollback is done (§17) | |
| R6a | **Make the same rebind in dev and commit it to Git** (a pull request reverting the wave's report bindings), and **freeze the cons deployment pipeline** until that's merged. Otherwise the next deployment rebinds the reports to the new model again | BI team |  |
| R7 | Move the wave back in the wave plan; root-cause analysis; fix in the feature branch; go back to the right phase (usually Phase 2) | |

**Escalation – roll back every wave:** only if the defect is in something every wave shares (for example a new gold table used by all the new models, or an RLS defect in the new models). Run R1–R6 for every completed wave, starting with the most recent. The approvers decide.

**Time to complete a wave rollback:** use the time measured in the rehearsal (2.12). Fill it in here: ______ minutes.

### 15.3 What rollback does *not* undo

- The new gold lakehouse and new models stay in prod, still serving earlier waves. They're fixed and reused for the rolled-back wave on the next attempt.
- Old items already **deleted** after an earlier wave's rollback period. Rollback for those waves needs the backups from 6.1, so delete old items only after their rollback period has ended.
- Downstream copies or exports users made from the new models during the cutover window.

## 16. Risk register

| ID | Risk | Likelihood | Impact | Mitigation | Fallback |
|---|---|---|---|---|---|
| R-01 | References to other workspaces (pipeline → notebook, model refresh activities, shortcuts) still point at dev/test after deploying (A5, A13) | **High** (by design without §5b.3) | High | `vl_env` per §5b.3; automated post-deploy checks (A.4) | Fix the values and redeploy; prod gold isn't consumed until Phase 4 |
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
| R-12 | Slalom engagement ends before the final wave | Medium | Medium | Handover pack and runbook walk-through before Phase 4 | Internal BI team owns hypercare |
| R-13 | **Running both stacks takes longer than planned**; A pays for both | Medium | High | Planned end date (§5a.7); waves ordered by CU savings; weekly CU budget review | §5a.4 options (point old models at new gold, lower refresh frequency, temporary scale-up) |
| R-14 | **The same KPI shows different numbers** in old-stack and new-stack reports | Medium | High | KPI cross-check after every load (§5a.6); intended corrections told to users of both stacks | Hold the wave; fix or document the difference |
| R-15 | **An old item is retired while something still uses it** | Low–Medium | High | Dependency map plus lineage check before every retirement (5.3); disable, don't delete | Turn the item back on (R1–R2) |
| R-16 | A failure in one orchestration branch **stops the other** | Medium | High | Separate branches with independent dependencies (§5a.3); failure tested in 2.10 | Run the stalled branch by hand; fix the dependency |
| R-17 | Developers build **new reports on the old models** while both stacks run | Medium | Medium | Mark old models "(legacy)" in their description; new models Promoted; review process for new reports | Add the report to a later wave |
| R-18 | **A Direct Lake model in prod reads the test lakehouse** (A10) | **High** without rules | Critical | §5b.4 rules; Direct Lake source check as a hard gate (A.5) | Fix the rule, redeploy, re-check before the wave goes ahead |
| R-19 | **A deployment or Git update deletes shortcuts** in the target lakehouse | Medium | High | No manual shortcuts in test or prod; review changes before every deploy (§5b.5) | Restore by redeploying from source; re-run catch-up loads |
| R-20 | Slalom's mixed workspace **can't be synced** into the folder-per-workspace layout | **High** if not addressed | Medium | Branched workspaces, one per target folder (§5b.1, step 0.8) | Move items into the right folders by hand in ADO before the pull request |
| R-21 | **A prod rollback is undone** by the next deployment | Medium | High | Rollback step R6a (mirror in dev, commit, freeze cons pipeline) | Rebind again; enforce the freeze |
| R-22 | Notebooks run against the **wrong default lakehouse** after sync or deploy | Medium | High | One D-12 approach checked in code review; post-deploy check (A.4) | Fix the setting or variable; rerun the affected loads |
| R-23 | Edits in the UI are overwritten by the next Git update, or drift from Git | Medium | Medium | No UI edits rule (§5b.1); access review | Recommit the change through a pull request |

## 17. Communication plan

| When | Audience | Message | Channel | Owner |
|---|---|---|---|---|
| Runbook approved | BI team, platform admin, Slalom | Plan, roles, timeline | Meeting + this document | Data Services |
| Non-prod freeze starts (Phase 2) | Developers on A | Freeze window, contacts | Team channel | BI team |
| 5 business days before **each wave** | Users of **that wave's** reports, plus users of old-stack reports that share KPIs with them | What's changing (nothing visible except possible data corrections), wave date, contacts | Email + app banner | Business owners |
| Weekly while both stacks run | Approvers | Wave progress, CU against budget, KPI cross-check status, end-date forecast | Short status note | Data Services |
| Go/No-Go | Approvers | Decision and evidence | Meeting; record in the decision log | Data Services |
| Wave start / end | Affected users, support desk | Started / completed; how to report issues | Email / team channel | BI team |
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
| CU cost of models (refresh + query) | **≤ old**; target reduction agreed in §19, measured **after the final wave** | B, prod |
| Broken visuals | **Zero** on every page of every in-scope report | B, test, prod |
| RLS test cases | **100% pass** | Test, prod |
| KPI cross-check between the stacks | **Exact match**, or a documented intended difference | Prod, while both stacks run |
| CU on A while both stacks run | Within the approved budget (D-8) | Prod, weekly |

## 19. Open decisions

| # | Decision | Options | Owner | Due |
|---|---|---|---|---|
| D-1 | Target CU reduction for the new models | e.g. ≥ 20% / ≥ 30% / any reduction | Data Services + platform admin | Before G1 |
| D-2 | Reconciliation tolerances (confirm §18 defaults) | As listed / custom | Business owners | Before G1 |
| D-3 | Hypercare length | 1 reporting cycle / 2 cycles / 30 days | Data Services + business | Before G4 |
| D-4 | Fix-forward time limit before rollback | e.g. 2 h / 4 h / next business day | Data Services | Before G4 |
| D-5 | Side-by-side run in prod: 1 or 2 cycles, or skip it (F3-b) | | Approvers | Before Phase 3 |
| D-6 | Future of capacity B after close-out | Keep / scale down / pause / delete | Data Services + platform admin | Phase 6 |
| D-7 | **End date for running both stacks** and the number of waves | e.g. 2 / 3 / 4 waves within N weeks | Approvers | Before Wave 1 |
| D-8 | **CU budget** for running both stacks (peak and average ceiling on A) | From the Phase 3 measurement | Platform admin + Data Services | Before Wave 1 (3.6) |
| D-9 | Point old models at new gold where possible (§5a.5)? | Yes, where safe / No | BI team + Data Services | Before Wave 1 |
| D-10 | Which reports go in the pilot (Wave 1) | | BI team + business owners | Phase 0 |
| D-11 | **Promotion route**: deployment pipelines started from ADO (default) **or** `fabric-cicd` from ADO release branches | Default / fabric-cicd | Data Services + BI team | Phase 0 |
| D-12 | **Notebook default lakehouse**: *Lakehouse Auto-Binding in Git* on every notebook **or** `%%configure` with variable-library variables | Auto-binding / %%configure | BI team + Slalom | Phase 0 |

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

**Listing which model each report is bound to (used in 4.1, 4.6 and G-13)**

Use the **admin scanner API** (lineage across workspaces), through semantic-link-labs or a scheduled notebook, to list every report with its bound model. Compare the output with the wave plan: every report must be on the model its wave says. Store each run in the repo next to the inventory. *(Check the semantic-link-labs admin helpers available in your installed version before scripting this.)*

### A.4 Checking references after a deploy (automated in the ADO pipeline, confirmed by a person)

For every stage you deploy to, confirm and record (with screenshots):

1. Each new-gold lakehouse **shortcut** → that stage's silver source
2. Each notebook's **default lakehouse** → that stage's lakehouse
3. Each **elt notebook activity** → that stage's transf notebook (workspace ID + notebook ID)
4. Each **semantic model** data source / parameter → that stage's new-gold
5. **Variable library**: the active value set is the right one for that stage, and its values match what's expected
6. Each **semantic model refresh activity** in elt → that stage's model ID
7. **Shortcut list** in each lakehouse is the same as before the deploy, with no unexpected deletions (§5b.5)

### A.5 Direct Lake source check (hard gate after every deploy)

```python
# Confirms which lakehouse each Direct Lake model is actually reading from.
# semantic-link-labs provides helpers for this (e.g. sempy_labs.directlake.get_direct_lake_source);
# check the exact function name and return shape in your installed version.
import sempy_labs.directlake as dl

models = ["<sm_bg1_area_v2>", "<sm_bg2_area_v2>"]
expected_ws = "<this stage's transf workspace>"
expected_lh = "lh_gold_v2"

for m in models:
    src = dl.get_direct_lake_source(dataset=m, workspace="<this stage's cons workspace>")
    print(m, "→", src)
    # FAIL the gate if the source workspace or lakehouse isn't this stage's expected_ws / expected_lh
```

If the check fails: fix the data source rule or parameter rule (§5b.4), redeploy the model, and run the check again.

---

## Appendix B – References

- [Capacity reassignment restrictions and common issues](https://learn.microsoft.com/fabric/admin/portal-workspace-capacity-reassignment)
- [Fabric capacity and OneLake consumption – shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview – capacity guardrails and limitations](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Fabric deployment pipelines](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/intro-to-deployment-pipelines)
- [Fabric Git integration](https://learn.microsoft.com/fabric/cicd/git-integration/intro-to-git-integration)
- [Power BI REST API – Rebind Report In Group](https://learn.microsoft.com/rest/api/power-bi/reports/rebind-report-in-group)
- [Deployment pipelines: autobinding](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/understand-the-deployment-process#autobinding)
- [Assign a workspace to a deployment pipeline: Direct Lake binding limitation](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/assign-pipeline#considerations-and-limitations)
- [Dependency binding in cross-workspace deployment](https://learn.microsoft.com/fabric/cicd/cross-workspace-dependency-binding)
- [Lakehouse Git integration and deployment pipelines](https://learn.microsoft.com/fabric/data-engineering/lakehouse-git-deployment-pipelines)
- [Variable library overview](https://learn.microsoft.com/fabric/cicd/variable-library/variable-library-overview)
- [Item reference variable type](https://learn.microsoft.com/fabric/cicd/variable-library/item-reference-variable-type)
- [Fabric CI/CD concepts and best practices](https://learn.microsoft.com/fabric/fundamentals/understand-best-practices-fabric-cicd)
- [CI/CD for Data Factory pipelines: known limitations](https://learn.microsoft.com/fabric/data-factory/cicd-pipelines#known-limitations)
- [Semantic link labs (GitHub)](https://github.com/microsoft/semantic-link-labs)
- [fabric-cicd (GitHub)](https://github.com/microsoft/fabric-cicd)
