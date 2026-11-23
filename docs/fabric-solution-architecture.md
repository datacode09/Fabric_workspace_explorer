# Solution Architecture: Gold Layer and Semantic Models on Microsoft Fabric

_This document describes **how the solution is built**: its structure, components, data flow, environments and design decisions. For **how it is delivered** (phases, gates, Go/No-Go, rollback), see the [migration runbook](fabric-capacity-migration-plan.md). For a short introduction, see the [overview](fabric-migration-overview.md) and the [architecture overview](fabric-architecture-overview.md)._

| | |
|---|---|
| **Document type** | Solution architecture |
| **Version** | 2.0 (split-capacity design: new gold on capacity B, orchestration and semantic layer on capacity A) |
| **Status** | Final draft for review |
| **Last updated** | 2026-11-23 |
| **Owner** | Data Services, Enterprise Technology |
| **Audience** | Architects, BI team, platform/capacity admins, Slalom, data governance |

---

## 1. Purpose and scope

This document sets out the target architecture for the **new gold layer**, the **semantic models** and the **release tooling** of the enterprise Fabric solution. It also covers the interim architecture in which the **old and new stacks run side by side**.

The design spreads the work across two capacities:

- **Capacity B** runs the **new silver-to-gold processing** (`transf-v2`) in its own dev, test and prod workspaces.
- **Capacity A** keeps **orchestration** (elt), the **old gold layer** until it is retired, and **all semantic models and reports**, old and new.

**In scope:** Fabric capacities, workspace layout, gold layer, orchestration across the two capacities, semantic models and reports, environments, release tooling (Azure DevOps and Fabric), security, operations, naming.

**Out of scope:** bronze/silver ingestion design, source systems, report visual design, and delivery planning (covered in the runbook).

## 2. Architecture principles

| # | Principle | What it means in practice |
|---|---|---|
| P1 | **Each workload on the right capacity** | Heavy Spark transformation runs on B; orchestration and the semantic layer (refreshes and interactive queries) stay on A |
| P2 | **Efficiency over capacity** | Load is reduced by making the gold layer and models cheaper to run, as well as by moving Spark off A |
| P3 | **Reports keep working if B has a problem** | The new models read new gold through **shortcuts on A**. If B is unavailable, reports keep serving the last loaded data |
| P4 | **One source of truth for definitions** | Every item definition lives in Azure DevOps Repos. Nobody edits workspaces in the UI outside development |
| P5 | **No hard-coded environment values** | Workspace, lakehouse, notebook and model IDs come from variable libraries or deployment rules |
| P6 | **Report identity is preserved** | Reports are rebound to new models, never recreated, so IDs, URLs, apps and subscriptions survive |
| P7 | **Explicit over automatic binding** | Where Fabric doesn't rebind references reliably (other workspaces, Direct Lake, refresh activities, links between capacities), the binding is set explicitly and checked |
| P8 | **Prod on B is production-grade** | B is never paused; it has named admins, monitoring, alerting and on-call cover, like A |
| P9 | **Security in the semantic layer** | Row-level security is defined and tested in the semantic models |

## 3. Context

```
                 ┌──────────────────────────────────────────────┐
  Source data ──▶│ Bronze / Silver (ingestion, capacity A)      │
                 └──────────────────────┬───────────────────────┘
                                        │ prod silver (read via shortcut)
                 ┌──────────────────────▼───────────────────────┐
                 │ NEW gold: transf-v2 on capacity B            │◀── started by elt on A
                 └──────────────────────┬───────────────────────┘
                                        │ shortcuts on A (lh_gold_v2_ref)
                 ┌──────────────────────▼───────────────────────┐
                 │ Semantic models + reports on capacity A      │
                 │ (cons-bg1, cons-bg2: old and new models)     │
                 └──────────────────────┬───────────────────────┘
                                        ▼
                      Business users BG1, BG2 (reports, apps, Excel)
```

**Actors:** business users (BG1, BG2), the BI team (owns release and operations on A and B), Data Services (architecture and delivery owner), capacity admins for A and B, and Slalom (SI partner, develops in `transf-v2-dev` and its branched workspaces on B).

## 4. Capacity architecture

### 4.1 Capacity roles

| Capacity | Role | Hosts |
|---|---|---|
| **A** | Orchestration, semantic layer, old gold (until retired), other non-prod | `elt-<env>`, `transf-<env>` (old gold, retiring), `cons-bg1-<env>`, `cons-bg2-<env>` (old and new models, reports, `lh_gold_v2_ref`) |
| **B** | **New gold processing, prod and non-prod** | `transf-v2-dev`, `transf-v2-test`, `transf-v2-prod`; Slalom's branched development workspaces |

**Hard constraint: A and B are in the same Fabric region.** Direct Lake models on A must be in the same region as the lakehouse they read, and the shortcuts on A point at `lh_gold_v2` on B.

### 4.2 Who pays for what (CU attribution)

| Operation | Capacity charged |
|---|---|
| New gold notebooks (Spark) in `transf-v2-*` | **B** |
| Reading silver from B through OneLake shortcuts | **B** (the reading workspace's capacity) |
| elt orchestration pipeline activities | **A** |
| Old gold notebooks in `transf-*` (until retired) | **A** |
| Semantic model refresh/framing and report queries, old and new | **A** |
| Reading new gold through `lh_gold_v2_ref` shortcuts on A | **A** |
| OneLake storage of `lh_gold_v2` | Billed per GB to B's storage; doesn't use CUs |

**Consequence:** A loses the Spark load of the new gold, but keeps orchestration and the entire semantic layer. Once the old gold is retired, A's Spark load from gold processing falls to zero.

### 4.3 Throttling and protection

- Fabric spreads **background** work (refreshes, Spark, pipelines) over 24 hours and **interactive** work (report queries) over a few minutes. Throttling starts when that spread-out use goes over the capacity's limit, and it slows **everything** on the capacity.
- **Capacity A:** surge protection on; CU budget while both stacks run (§11); Capacity Metrics app as the system of record.
- **Capacity B:** surge protection on, so dev/test spikes can't starve prod. Dev and test Spark pools are capped (§9.2). **Spark autoscale billing** (pay-as-you-go Spark outside B's fixed pool) is the preferred relief valve if B gets crowded.
- **B is never paused.** Pausing B stops new-gold loads; reports keep working on the last loaded data through the shortcuts on A, but the data goes stale.

## 5. Logical architecture

### 5.1 Workspace layout

| Layer | Workspace (per env) | Capacity | Contents |
|---|---|---|---|
| Orchestration | `elt-<env>` | A | Master orchestration pipeline (old and new branches); `vl_env` |
| Old transformation | `transf-<env>` | A | Old gold notebooks; `lh_gold` (retired at the end of the migration) |
| **New transformation** | **`transf-v2-<env>`** | **B** | New gold notebooks; `lh_gold_v2`; `nb_gold_v2_setup`; Spark environment; `vl_env` |
| Consumption – BG1 | `cons-bg1-<env>` | A | Old and new semantic models; reports; app; **`lh_gold_v2_ref`** (shortcuts to `lh_gold_v2`); `vl_env` |
| Consumption – BG2 | `cons-bg2-<env>` | A | Same as BG1 |

`<env>` = dev, test, prod. Slalom's development also uses **branched workspaces** on B, one per layer it changes.

### 5.2 Layer contracts

| Interface | Contract |
|---|---|
| Silver (A) → new gold (B) | `transf-v2` notebooks read silver through **OneLake shortcuts** in `lh_gold_v2`, with targets set from variable-library item references for each environment |
| New gold (B) → semantic layer (A) | `lh_gold_v2_ref` in each cons workspace holds **shortcuts** to `lh_gold_v2` tables. Gold table names, column names, data types and grain are the **gold contract** |
| `lh_gold_v2_ref` → new semantic models | Models read the shortcut tables in **Direct Lake** mode |
| Semantic model → report | Reports use model **field names**. These names are a contract: renaming one breaks reports unless a compatibility measure is provided |
| Orchestration (A) → new transformation (B) | elt calls `transf-v2` notebooks in **another workspace on another capacity**, using IDs from the variable library |

### 5.3 Why the semantic layer reads new gold through shortcuts on A

- **Keeps working when B is unavailable:** a consumer can still read the data through a shortcut on an active capacity when the capacity holding it is paused. Reports keep serving the last loaded gold.
- **Same cost:** reads are charged to A whether the model reads B directly or through the shortcut.
- **Simpler binding:** each model reads a lakehouse in its **own** workspace. Only the shortcut targets cross workspaces and capacities, and those are set by variables.

## 6. Data flow and orchestration

### 6.1 Flow

```
 elt master pipeline (A)
   │
   ├─▶ wait for silver complete (A)
   │
   ├─▶ OLD branch (run_old_<area>) ─▶ old gold notebooks (A) ─▶ lh_gold (A)    ─▶ refresh old models (A)
   │
   └─▶ NEW branch (run_new_<area>) ─▶ new gold notebooks (B) ─▶ lh_gold_v2 (B)
                                                                    │
                                     lh_gold_v2_ref shortcuts (A) ◀─┘
                                                │
                                                └─▶ frame/refresh new models (A) ─▶ KPI cross-check
```

### 6.2 Orchestration design rules

- **Independent branches:** the old and new branches depend only on silver being finished. A failure in one branch, including B being unavailable, doesn't stop the other.
- **Flags by subject area:** `run_old_<area>` and `run_new_<area>` come from the variable library, so each subject area can be switched without a code change.
- **Explicit IDs:** notebook activities that call `transf-v2` on B, and semantic-model refresh activities, take workspace and item IDs from **GUID variables** in elt's `vl_env`, with a value for each environment.
- **Run identity:** scheduled runs use a **service principal or workspace identity**, never a personal account. OneLake shortcuts use the caller's identity, so that identity needs read access to silver on A, write access to `lh_gold_v2` on B, and read access through `lh_gold_v2_ref`.
- **Scheduling:** the new branch's Spark runs on B, so it doesn't add to A's Spark peak. The old branch stays in A's off-peak window. Model framing on A runs after the new gold load finishes.
- **Idempotency:** gold notebooks can be re-run for a load date without creating duplicates (merge or replace by partition).
- **Freshness:** an alert fires if `lh_gold_v2` isn't loaded by the agreed time.

## 7. Gold layer design

### 7.1 Storage

- **Lakehouse:** `lh_gold_v2` in `transf-v2-<env>` on B. The old `lh_gold` stays in `transf-<env>` on A until it is retired.
- **Table setup:** `nb_gold_v2_setup` is an idempotent notebook that creates every table with its schema, partitioning and table properties. It runs in each environment before the first load. Deployment and Git carry lakehouse **metadata and shortcuts only, never tables or data**.
- **Shortcuts:**
  - `lh_gold_v2` (B) → silver (A): item-reference variables for each environment.
  - `lh_gold_v2_ref` (A) → `lh_gold_v2` (B): item-reference variables for each environment. Every gold table the new models use has a shortcut.
  - Shortcuts are never created by hand in test or prod. A deployment or Git update resets the target lakehouse's shortcuts to match the source, and deletes any that aren't in the source.

### 7.2 Table design standards

| Concern | Standard |
|---|---|
| Format | Delta, with V-Order on |
| Maintenance | Scheduled OPTIMIZE (and VACUUM in line with retention), run on B |
| Partitioning | Only on large fact tables, by a low-cardinality date column; avoid many small files |
| Columns | Only the columns the semantic models use; avoid high-cardinality text columns where possible |
| History | SCD type and keys documented per dimension; reconciliation covers current and historical rows |
| Direct Lake guardrails | File, row-group and row counts per table kept within the limits for **A's SKU**, because that is where the models run |

## 8. Semantic layer

### 8.1 Model design

- **Location:** all semantic models, old and new, are on **capacity A** in `cons-bg1-<env>` / `cons-bg2-<env>`.
- **Storage mode:** Direct Lake on `lh_gold_v2_ref`. The flavour (**Direct Lake on OneLake** or **on SQL endpoint**) is chosen per model and recorded.
- **Granularity:** models by business group and subject area (`sm_<bg>_<area>_v2`), not one large model.
- **Measures:** business logic lives in measures. Compatibility measures keep the old field names where existing reports need them.
- **Fallback:** for Direct Lake on SQL endpoint, DirectQuery fallback is set on purpose and monitored. Unplanned fallback is a performance defect.

### 8.2 Binding to the gold lakehouse

A Direct Lake model **stays bound to the source environment's lakehouse when deployed**. The binding is therefore explicit:

| Flavour | Binding mechanism |
|---|---|
| Direct Lake on SQL endpoint | **Data source rule** per environment, pointing at that environment's `lh_gold_v2_ref` SQL endpoint |
| Direct Lake on OneLake | **M parameters** (workspace, lakehouse) with **parameter rules** per environment |

An automated check after each deploy confirms each model reads **its own environment's** `lh_gold_v2_ref`, and that those shortcuts point at **the same environment's** `lh_gold_v2` on B.

### 8.3 Reports

- Reports live in the same cons workspace as their model. Within one workspace, the deployment pipeline keeps a report bound to the matching model in the next environment.
- Moving a report to a new model is done by **rebinding** (in dev, carried through deployment). Report ID, URL, app placement, bookmarks and subscriptions are kept.
- Each report is bound to **exactly one** model at a time.

### 8.4 Endorsement

New models are marked **Promoted**, then **Certified** once the old stack is gone. Old models are labelled "(legacy)" in their description, and no new reports may be built on them.

## 9. Environments and release architecture

### 9.1 Environments

| Layer | dev | test | prod |
|---|---|---|---|
| elt | A | A | A |
| transf (old, retiring) | A | A | A |
| **transf-v2 (new gold)** | **B** | **B** | **B** |
| cons-bg1 / cons-bg2 | A | A | A |

Slalom's branched development workspaces are on B. test and prod are changed **only** by deployment pipelines.

### 9.2 Keeping non-prod from crowding prod on B

`transf-v2` dev, test and prod share B. To stop non-prod work slowing the prod gold load:

1. **Capped Spark pools** for `transf-v2-dev` and `transf-v2-test`: custom pools with a low maximum node count.
2. **The prod load window is kept clear:** backfills, load tests and reconciliation runs in dev/test happen outside it.
3. **Surge protection** on B.
4. **Spark autoscale billing** on B as the next step if contention shows up in the Metrics app.
5. **Exit route:** move `transf-v2-dev/test` to a small separate capacity. Within the same region this is a simple reassignment, and the deployment pipeline keeps working.

### 9.3 Source control (Azure DevOps)

```
 ADO repo (branch: dev)
 ├── /workspaces/elt         ◀── elt-dev        (A)
 ├── /workspaces/transf      ◀── transf-dev     (A)   old gold, until retired
 ├── /workspaces/transf-v2   ◀── transf-v2-dev  (B)   new gold
 ├── /workspaces/cons-bg1    ◀── cons-bg1-dev   (A)
 └── /workspaces/cons-bg2    ◀── cons-bg2-dev   (A)

 feature/* branches ◀── Slalom branched workspaces on B (one per folder it changes)
```

- **One repo, one folder per workspace:** every dev workspace connects to the same repo and branch, each with its own **Git folder**. Only dev workspaces connect to Git.
- **Connection:** the Azure DevOps source-control connection, using a **service principal**.
- **Branching:** short-lived `feature/*` branches → pull request → `dev`. Prod releases are tagged (`release/<wave>-<date>`).
- **Branch policies on `dev`:** required reviewer from the BI team; linked work item; **build validation** that rejects hard-coded GUIDs, workspace names and ABFS paths.
- **Formats:** notebooks as source files; semantic models as TMDL; reports as PBIR; lakehouses as metadata plus `shortcuts.metadata.json`.

### 9.4 Promotion

```
 feature/* ──PR──▶ dev (ADO) ──sync──▶ dev workspaces ──▶ test ──▶ prod
                                                │            ▲         ▲
                 ADO pipeline (REST API) with approvals + post-deploy checks
```

- One **Fabric deployment pipeline per layer** (elt, transf, **transf-v2**, cons-bg1, cons-bg2), each dev → test → prod, **started from an Azure DevOps pipeline** through the REST API. **ADO environments** supply the approvals and the audit trail.
- All deployment pipelines have the **same number of stages**, because binding across pipelines matches stages by position.
- **Deployment order for a release:** transf-v2 (B) → elt (A) → cons (A).
- **Post-deploy checks** in the ADO pipeline: references across workspaces and capacities, Direct Lake source, `lh_gold_v2_ref` shortcut targets, report bindings, active variable-library value set.
- **Alternative (decision D-11):** `fabric-cicd` deploying from ADO release branches. One promotion route is used, never both.

### 9.5 Configuration (variable libraries)

Each workspace has a variable library `vl_env` with **dev, test and prod value sets**:

| Reference | Workspace | Variable type / mechanism |
|---|---|---|
| Notebook default lakehouse | transf-v2 | *Lakehouse Auto-Binding in Git* **or** `%%configure` with name/ID variables (decision D-12) |
| Lakehouse paths in notebook code | transf-v2 | `notebookutils.variableLibrary` |
| `lh_gold_v2` → silver shortcuts | transf-v2 | Item reference variables (silver lakehouse on A, per environment) |
| `lh_gold_v2_ref` → `lh_gold_v2` shortcuts | cons-bg1 / cons-bg2 | Item reference variables (`lh_gold_v2` on B, per environment) |
| elt → `transf-v2` notebook activities | elt | GUID variables (B workspace ID, notebook IDs) |
| elt → semantic model refresh activities | elt | GUID variables (workspace ID, model ID) |
| Direct Lake model → `lh_gold_v2_ref` | cons | Deployment rules (§8.2), not variable libraries |

### 9.6 How Fabric handles bindings (design constraints)

| Behaviour | Design response |
|---|---|
| References to items in **another workspace** (and therefore another capacity) never rebind on deploy or Git sync | Variable-library IDs and item references (§9.5) |
| **Direct Lake** models stay bound to the source environment's lakehouse | Data source or parameter rules; post-deploy check |
| Pipeline **semantic model refresh** activities use the item ID, even within the same workspace | GUID variables |
| Notebook → lakehouse binding through Git needs a per-notebook setting (off by default) | Decision D-12; code review check |
| Deployments and Git updates **reset the target lakehouse's shortcuts** | No manual shortcuts in test or prod; review changes before deploying |
| Deployment pipelines **pair items by name and type** | `_v2` naming; no renames during the migration |
| Tables and data are never deployed | Table setup notebook per environment |
| A workspace syncs to exactly **one** Git folder | Slalom works in one branched workspace per layer |

## 10. Security and access

| Area | Design |
|---|---|
| Workspace roles – A | Prod: BI team Admin/Member; business users get **app access** or Viewer only. Test: BI team and UAT testers. Dev: BI team |
| Workspace roles – B | `transf-v2-prod`: BI team (Admin), deployment service principal, run identity. **No Slalom, no individual developers.** `transf-v2-test`: BI team, UAT testers. `transf-v2-dev` and branched workspaces: BI team, Slalom |
| Identities | Service principal for Git connections and the ADO deployment pipeline. Service principal or workspace identity for scheduled runs (needs silver read on A, `lh_gold_v2` write on B, and read through `lh_gold_v2_ref` on A) |
| Data across capacities | Shortcuts use the caller's identity. Access to silver from B is granted only to the run identity and to dev/test as agreed, and is reviewed regularly |
| Row-level security | Defined in the semantic models on A; role names and filter logic match the old models; tested with test users for every model change |
| OneLake security roles | Not used for this solution (preview); RLS stays in the semantic layer |
| Separation of duties | Developers (including Slalom) commit and raise pull requests; the BI team reviews, approves and deploys |
| Secrets | No secrets in item definitions; connections use Fabric connection objects or Key Vault-backed configuration |

## 11. Running the old and new stacks side by side

During the migration the solution runs **two stacks** that share silver, the master pipeline and the report items:

| Component | Old stack | New stack | Shared |
|---|---|---|---|
| Silver (A) | | | ✔ |
| Gold processing | `transf-<env>` on **A** | `transf-v2-<env>` on **B** | |
| Gold lakehouse | `lh_gold` (A) | `lh_gold_v2` (B), seen from A through `lh_gold_v2_ref` | |
| Orchestration | Old branch (`run_old_<area>`) | New branch (`run_new_<area>`) | Master pipeline (A) |
| Semantic models (A) | `sm_<bg>_<area>` | `sm_<bg>_<area>_v2` | Same cons workspaces |
| Reports (A) | Not yet moved | Moved (rebound) | Same report items |

**Design mechanisms:**

- **Moving a report** is a rebind. Reports move in groups (waves), and each report belongs to exactly one stack at a time.
- **Consistency:** an automated KPI cross-check compares the same KPIs across the two stacks after every load.
- **Load on A:** while both run, A still carries the old gold Spark plus refreshes for both sets of models. A CU budget applies. Each wave retires old notebooks and models as soon as nothing uses them, so A's load falls with every wave.
- **Retiring old items:** a dependency map (from the admin scanner API) shows when an old item has nothing using it. It's switched off with its flag or refresh schedule, then removed.
- **End state:** the old branch, `transf-<env>`, `lh_gold`, the old models and the switch flags are removed. Gold processing runs only on B; A keeps orchestration and the semantic layer.

## 12. Operations and observability

| Signal | Tool | Used for |
|---|---|---|
| CU use, throttling, top items – **A and B** | Capacity Metrics app | Baseline, budget, contention between dev/test and prod on B |
| Pipeline and notebook runs | Monitoring hub | Run status, duration (elt on A; notebooks on B) |
| Failures and SLA breaches | Activator alerts (one per branch, with the stack and capacity named in the alert) | Operations response |
| **New gold freshness** | Freshness check on `lh_gold_v2` | Detects B problems before users notice stale data |
| Model refreshes / framing | Refresh history; alerts | Data freshness on A |
| Data consistency | Reconciliation and KPI cross-check notebooks | Correctness while both stacks run |
| Lineage and report bindings | Admin scanner API (semantic-link-labs) | Inventory, dependency map, binding checks |
| Change history | Azure DevOps (pull requests, pipeline runs, release tags) | Audit trail |

**Operational ownership:** both capacities have named admins and an on-call owner. **B is never paused.** B's run cost is part of the production budget.

**Failure modes:**

| Failure | Effect | Response |
|---|---|---|
| B unavailable or throttled during the load window | New gold not refreshed; new-stack reports show the last loaded data; old branch unaffected | Freshness alert; rerun the new branch when B recovers |
| A throttled | Report queries and refreshes slow down (old and new) | Surge protection; CU budget; speed up retiring old items |
| Shortcut target wrong after a deploy | New models read the wrong environment's gold | Post-deploy check blocks the release |
| Old branch fails | Old-stack reports stale | Same as today; new branch unaffected |

## 13. Naming conventions

| Item | Pattern | Example |
|---|---|---|
| Workspace | `<layer>-<env>` | `transf-v2-prod` |
| New gold lakehouse (B) | `lh_gold_v2` | |
| Shortcut lakehouse (A) | `lh_gold_v2_ref` | |
| Gold notebook | `nb_gold_v2_<area>` | `nb_gold_v2_billing` |
| Setup notebook | `nb_gold_v2_setup` | |
| Semantic model | `sm_<bg>_<area>_v2` | `sm_bg1_billing_v2` |
| Variable library | `vl_env` | |
| Orchestration flags | `run_old_<area>`, `run_new_<area>` | `run_new_billing` |
| Branches | `feature/<topic>`, `release/<wave>-<yyyymmdd>` | `release/wave-1-20270115` |

Items are never renamed once they've been deployed, because deployment pipelines pair items by name.

## 14. Non-functional requirements

| Quality | Requirement | How the architecture meets it |
|---|---|---|
| Performance | New models refresh and respond at least as fast as the old ones | Direct Lake; table design standards; smaller models per subject area |
| Cost / capacity | No user-facing throttling on A; new models use fewer CUs (target in D-1); B sized for prod gold plus capped non-prod | Spark moved to B; surge protection on A and B; CU budget while both stacks run |
| Correctness | Gold and KPIs match the old stack exactly, or the difference is a documented correction | Reconciliation; KPI cross-check |
| Availability | No outage for report users when a report moves; reports keep working if B is unavailable | Rebind; shortcuts on A (§5.3); old stack runs until retired |
| Recoverability | A moved report can go back to the old model quickly | Rebind back to the old model, which is still refreshing; the change is mirrored in ADO |
| Rebuildability | Any environment can be rebuilt | Definitions in ADO; tables from the setup notebook; data reloaded from silver |
| Security | Users see only the data they're entitled to; prod on B is locked down | RLS in models; workspace roles; separation of duties |
| Auditability | Every prod change can be traced | ADO pull requests, approvals, release tags |
| Maintainability | No environment-specific values in code | Variable libraries; build validation |

## 15. Architecture decisions

| ID | Decision | Rationale | Alternatives considered |
|---|---|---|---|
| AD-01 | **New silver-to-gold processing runs on capacity B** (`transf-v2` dev, test and prod) | Takes the heaviest Spark load off A | Keep it on A; scale A up |
| AD-02 | **Orchestration and all semantic models (old and new) stay on capacity A** | One place for scheduling, models and reports; no change for report users | Move models to B |
| AD-03 | New models read new gold through **shortcuts on A** (`lh_gold_v2_ref`) | Reports keep working if B is paused; same cost; simpler binding | Direct Lake straight onto `lh_gold_v2` on B |
| AD-04 | `transf-v2` dev and test also on B, with capped Spark pools, a protected load window and surge protection | Test matches prod; validation load stays off A; no extra capacity | Dev/test on A (adds load back); separate small capacity (extra cost; kept as exit route) |
| AD-05 | Reports are **rebound**, not replaced | Keeps URLs, apps, bookmarks and subscriptions | Publish new reports and retire the old ones |
| AD-06 | Old and new stacks side by side, with flags by subject area | Gradual moves, rollback per wave, A's load falls wave by wave | Big-bang cutover |
| AD-07 | Azure DevOps Repos as source of truth; one folder per workspace, including `transf-v2` | Existing VCS; Fabric supports one repo for many workspaces | One repo per workspace |
| AD-08 | Fabric deployment pipelines (one per layer) started from ADO | Smallest change to the current release process; ADO adds approvals and audit | `fabric-cicd` (open: D-11) |
| AD-09 | Environment values in variable libraries, including IDs and shortcut targets that cross capacities | Fabric doesn't rebind references to other workspaces | Deployment rules only; post-deploy scripts |
| AD-10 | Direct Lake bound through deployment rules, checked after every deploy | Direct Lake doesn't rebind on deployment | Rebind by hand after deployment |
| AD-11 | `_v2` naming; no renames during the migration | Deployment pipelines pair items by name | Reuse names |
| AD-12 | RLS in the semantic models | Mature, testable; OneLake security is in preview | OneLake security roles |
| AD-13 | Scheduled runs use a service principal or workspace identity | Shortcuts use the caller's identity across capacities; no dependency on personal accounts | Personal accounts |

## 16. Risks and constraints (architecture)

| Item | Type | Response |
|---|---|---|
| A and B must be in the same region | Hard constraint | Checked before any build on B |
| B unavailable or throttled during the load window | Operational risk | Shortcuts on A keep reports working; freshness alert; rerun |
| Dev/test crowding prod on B | Design risk | Capped pools; protected window; surge protection; autoscale billing; exit route (§9.2) |
| References across workspaces and capacities not rebound | Platform constraint | Variable libraries; post-deploy checks |
| Direct Lake binding isn't automatic | Platform constraint | AD-10; post-deploy check |
| Deployments reset shortcuts (`lh_gold_v2` and `lh_gold_v2_ref`) | Platform constraint | No manual shortcuts; review before deploy |
| Run identity lacks access on one capacity | Design risk | Service principal or workspace identity with documented access on A and B; tested before go-live |
| A still carries both models' refreshes while both stacks run | Design risk | CU budget; retire old items each wave |
| Field-name drift between old and new models | Design risk | Field names as a contract; compatibility measures |
| Direct Lake guardrails for A's SKU | Platform constraint | Table design standards (§7.2) |

## 17. Open questions

| # | Question | Owner |
|---|---|---|
| Q-1 | Direct Lake flavour (on OneLake / on SQL endpoint) per model | BI team + Slalom |
| Q-2 | Promotion route: deployment pipelines started from ADO, or `fabric-cicd` (D-11) | Data Services + BI team |
| Q-3 | Notebook default lakehouse mechanism (D-12) | BI team + Slalom |
| Q-4 | Turn on Spark autoscale billing on B from the start, or only if contention appears | B capacity admin |
| Q-5 | B's SKU size for prod gold plus capped non-prod | B capacity admin + Data Services |
| Q-6 | Run identity: service principal or workspace identity | BI team + platform admin |

## 18. References

- [Migration runbook](fabric-capacity-migration-plan.md) · [Overview](fabric-migration-overview.md) · [Architecture overview](fabric-architecture-overview.md)
- [Fabric capacity and OneLake consumption – shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Deployment pipelines: autobinding](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/understand-the-deployment-process#autobinding)
- [Assign a workspace to a deployment pipeline: Direct Lake limitation](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/assign-pipeline#considerations-and-limitations)
- [Dependency binding in cross-workspace deployment](https://learn.microsoft.com/fabric/cicd/cross-workspace-dependency-binding)
- [Lakehouse Git integration and deployment pipelines](https://learn.microsoft.com/fabric/data-engineering/lakehouse-git-deployment-pipelines)
- [Variable library overview](https://learn.microsoft.com/fabric/cicd/variable-library/variable-library-overview)
- [Fabric CI/CD concepts and best practices](https://learn.microsoft.com/fabric/fundamentals/understand-best-practices-fabric-cicd)
