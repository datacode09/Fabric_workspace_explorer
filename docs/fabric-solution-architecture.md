# Solution Architecture: Gold Layer and Semantic Models on Microsoft Fabric

_This document describes **how the solution is built**: its structure, components, data flow, environments and design decisions. For **how it is delivered** (phases, gates, Go/No-Go, rollback), see the [migration runbook](fabric-capacity-migration-plan.md). For a short introduction, see the [overview](fabric-migration-overview.md)._

| | |
|---|---|
| **Document type** | Solution architecture |
| **Version** | 1.0 |
| **Status** | Draft for review |
| **Last updated** | 2026-11-12 |
| **Owner** | Data Services, Enterprise Technology |
| **Audience** | Architects, BI team, platform/capacity admin, Slalom, data governance |

---

## 1. Purpose and scope

This document sets out the target architecture for the **gold layer**, the **semantic models** and the **release tooling** of the enterprise Fabric solution. It also covers the interim architecture in which the **old and new stacks run side by side**.

**In scope:** Fabric capacities, workspace layout, gold layer, orchestration, semantic models and reports, environments, release tooling (Azure DevOps and Fabric), security, observability, naming.

**Out of scope:** bronze/silver ingestion design, source systems, report visual design, and delivery planning (covered in the runbook).

## 2. Architecture principles

| # | Principle | What it means in practice |
|---|---|---|
| P1 | **Production stays on capacity A** | All prod and non-prod workspaces live on A. Capacity B is a development area and nothing on A depends on it |
| P2 | **Efficiency over capacity** | Load is reduced by making the gold layer and models cheaper to run, not by adding compute |
| P3 | **One source of truth for definitions** | Every item definition lives in Azure DevOps Repos. Nobody edits workspaces in the UI outside development |
| P4 | **No hard-coded environment values** | Workspace, lakehouse, notebook and model IDs come from variable libraries or deployment rules |
| P5 | **Report identity is preserved** | Reports are rebound to new models, never recreated, so IDs, URLs, apps and subscriptions survive |
| P6 | **Separation of layers** | Orchestration, transformation and consumption live in separate workspaces with clear contracts between them |
| P7 | **Explicit over automatic binding** | Where Fabric doesn't rebind references reliably (other workspaces, Direct Lake, refresh activities), the binding is set explicitly and checked |
| P8 | **Security in the semantic layer** | Row-level security is defined and tested in the semantic models |
| P9 | **Observable by design** | CU use, pipeline runs, refreshes and data consistency are monitored and alerted on |

## 3. Context

```
                ┌───────────────────────────────────────────────┐
  Source data ─▶│ Bronze / Silver (ingestion – out of scope)    │
                └───────────────────────┬───────────────────────┘
                                        │ prod silver
                ┌───────────────────────▼───────────────────────┐
                │ Gold layer (transf)  ◀── orchestration (elt)  │
                └───────────────────────┬───────────────────────┘
                                        │ Direct Lake
                ┌───────────────────────▼───────────────────────┐
                │ Semantic models + reports (cons-bg1, cons-bg2)│
                └───────────────────────┬───────────────────────┘
                                        ▼
                     Business users BG1, BG2 (reports, apps, Excel)
```

**Actors:** business users (BG1, BG2), the BI team (owns release and operations), Data Services (architecture and delivery owner), the platform/capacity admin, and Slalom (SI partner, develops on capacity B).

## 4. Capacity architecture

### 4.1 Capacity roles

| Capacity | Role | Hosts |
|---|---|---|
| **A** | Production and non-production | elt, transf, cons-bg1 and cons-bg2 workspaces: dev, test and prod |
| **B** | Development area | Slalom's branched development workspaces; heavy validation and load testing |

Both capacities **must be in the same Fabric region**. Direct Lake models must be in the same region as their source lakehouse, and keeping both capacities in one region keeps cross-capacity shortcuts simple.

### 4.2 Who pays for what (CU attribution)

| Operation | Capacity charged |
|---|---|
| Notebook / Spark job | The capacity of the workspace where the notebook runs |
| Pipeline activities | The capacity of the workspace that holds the pipeline (and of the item each activity runs) |
| Reading data through a **OneLake shortcut** | The capacity of the workspace that holds the **shortcut** (the reader), not where the data is stored |
| OneLake **storage** | Billed per GB where the data is stored; doesn't use CUs |
| Semantic model refresh and queries | The capacity of the workspace that holds the **semantic model** |

**Consequence:** Slalom's validation on B reads prod silver on A through shortcuts and is charged to **B**, so it adds nothing to A's load.

### 4.3 Throttling and protection

- Fabric spreads **background** work (refreshes, Spark, pipelines) over 24 hours and **interactive** work (report queries) over a few minutes. Throttling starts when spread-out use goes over the capacity's limit. So heavy background jobs on A also slow report users down.
- **Surge protection** is turned on for A, to cap background spikes.
- **Spark autoscale billing** (pay-as-you-go Spark outside the capacity) is the preferred relief valve if Spark remains the main load. It comes before a capacity scale-up.
- The **Capacity Metrics app** is the system of record for CU baselines and budgets.

## 5. Logical architecture

### 5.1 Workspace layout

| Layer | Workspace (per environment) | Contents |
|---|---|---|
| Orchestration | `elt-<env>` | Master orchestration pipeline; variable library |
| Transformation | `transf-<env>` | Silver-to-gold notebooks; gold lakehouse(s); table setup notebook; variable library |
| Consumption – BG1 | `cons-bg1-<env>` | Semantic models; reports; workspace app; variable library |
| Consumption – BG2 | `cons-bg2-<env>` | Semantic models; reports; workspace app; variable library |

`<env>` = dev, test, prod on capacity **A**. Slalom's development uses **branched workspaces, one per layer**, on capacity **B**.

### 5.2 Layer contracts

| Interface | Contract |
|---|---|
| Silver → gold | Gold notebooks read prod-grade silver through **OneLake shortcuts** in the gold lakehouse, defined with variable-library item references per environment |
| Gold → semantic model | Models read gold Delta tables in **Direct Lake** mode. Gold table names, column names, data types and grain form the **gold contract**. Changes to it are reviewed for impact through lineage |
| Semantic model → report | Reports use model **field names** (tables, columns, measures). These names are a contract: renaming one breaks reports unless a compatibility measure is provided |
| Orchestration → transformation | elt calls transf notebooks **in another workspace**, using IDs from the variable library |

## 6. Data flow and orchestration

### 6.1 Flow

```
 elt master pipeline
   │
   ├─▶ wait for silver complete
   │
   ├─▶ OLD branch (run_old_<area>) ─▶ old gold notebooks ─▶ lh_gold    ─▶ refresh old models
   │
   └─▶ NEW branch (run_new_<area>) ─▶ new gold notebooks ─▶ lh_gold_v2 ─▶ refresh new models
                                                                 │
                                                                 └─▶ KPI cross-check (old vs new)
```

### 6.2 Orchestration design rules

- **Independent branches:** the old and new branches depend only on silver being finished. A failure in one branch doesn't stop the other (separate child pipelines, or "on completion" dependencies).
- **Flags by subject area:** `run_old_<area>` and `run_new_<area>` come from the variable library, so each subject area can be switched without a code change.
- **Scheduling:** branches are staggered in the off-peak window so their CU peaks don't overlap.
- **Explicit IDs:** invoke-notebook activities that call transf, and semantic-model refresh activities, take workspace and item IDs from **GUID variables** in the variable library. They are never rebound automatically.
- **Idempotency:** gold notebooks can be re-run for a load date without creating duplicates (merge or replace by partition).

## 7. Gold layer design

### 7.1 Storage

- **Lakehouse:** `lh_gold_v2` in `transf-<env>`, next to the old `lh_gold` while both stacks run.
- **Table setup:** `nb_gold_v2_setup` is an idempotent notebook that creates every table with its schema, partitioning and table properties. It runs in each environment before the first load. Deployment and Git carry lakehouse **metadata and shortcuts only, never tables or data**.
- **Shortcuts:** shortcuts to silver in another workspace are defined with **variable-library item references**. Shortcuts are never created by hand in test or prod, because a deployment or Git update resets the lakehouse's shortcuts to match the source.

### 7.2 Table design standards

| Concern | Standard |
|---|---|
| Format | Delta, with V-Order on |
| Maintenance | Scheduled OPTIMIZE (and VACUUM in line with retention) |
| Partitioning | Only on large fact tables, by a low-cardinality date column; avoid many small files |
| Columns | Only the columns the semantic models use; avoid high-cardinality text columns where possible |
| History | SCD type and keys documented per dimension; reconciliation covers current and historical rows |
| Direct Lake guardrails | File, row-group and row counts per table kept within the limits for A's SKU (the runbook's acceptance criteria) |

## 8. Semantic layer

### 8.1 Model design

- **Storage mode:** Direct Lake on the gold lakehouse. The flavour (**Direct Lake on OneLake** or **Direct Lake on SQL endpoint**) is chosen per model and recorded in the model's documentation.
- **Granularity:** models by business group and subject area (`sm_<bg>_<area>_v2`), not one large model, to keep memory use and refresh scope small.
- **Measures:** business logic lives in measures. Compatibility measures keep the old field names where existing reports need them.
- **Fallback:** for Direct Lake on SQL, DirectQuery fallback behaviour is set on purpose and monitored. Unplanned fallback is a performance defect.

### 8.2 Binding to the gold lakehouse

A Direct Lake model **stays bound to the source environment's lakehouse when deployed**, and the gold lakehouse is in a different workspace (transf). The binding is therefore explicit:

| Flavour | Binding mechanism |
|---|---|
| Direct Lake on SQL endpoint | **Data source rule** per environment in the deployment pipeline |
| Direct Lake on OneLake | **M parameters** (workspace, lakehouse) with **parameter rules** per environment |

An automated check after each deploy confirms each model reads **its own environment's** `lh_gold_v2`.

### 8.3 Reports

- Reports live in the same cons workspace as their model. Within one workspace, the deployment pipeline keeps a report bound to the matching model in the next environment.
- Moving a report to a new model is done by **rebinding** (in dev, carried through deployment). Report ID, URL, app placement, bookmarks and subscriptions are kept.
- Each report is bound to **exactly one** model at a time.

### 8.4 Endorsement

New models are marked **Promoted**, then **Certified** once the old stack is gone. Old models are labelled "(legacy)" in their description, and no new reports may be built on them.

## 9. Environments and release architecture

### 9.1 Environments

| Environment | Capacity | Purpose | Changed by |
|---|---|---|---|
| Slalom development (branched workspaces) | B | Build, heavy validation, load testing | Slalom, through feature branches |
| dev | A | Integration of all changes | Sync from Azure DevOps (`dev` branch) |
| test | A | UAT, smoke tests, rollback rehearsal | Deployment pipeline only |
| prod | A | Business use | Deployment pipeline only, with approval |

### 9.2 Source control (Azure DevOps)

```
 ADO repo
 ├── /elt         ◀── elt-dev           (dev branch)
 ├── /transf      ◀── transf-dev        (dev branch)
 ├── /cons-bg1    ◀── cons-bg1-dev      (dev branch)
 └── /cons-bg2    ◀── cons-bg2-dev      (dev branch)

 feature/* branches ◀── Slalom branched workspaces on B (one per folder)
```

- **Connection:** the Azure DevOps source-control connection, using a **service principal**.
- **Branching:** short-lived `feature/*` branches → pull request → `dev`. Prod releases are tagged (`release/<wave>-<date>`).
- **Branch policies on `dev`:** required reviewer from the BI team; linked work item; **build validation** that rejects hard-coded GUIDs, workspace names, ABFS paths and references to capacity B.
- **Formats:** notebooks as source files; semantic models as TMDL; reports as PBIR; lakehouses as metadata plus `shortcuts.metadata.json`.

### 9.3 Promotion

```
 feature/* ──PR──▶ dev (ADO) ──sync──▶ dev workspace ──▶ test ──▶ prod
                                              ▲                ▲         ▲
                          ADO pipeline (REST API) with approvals + post-deploy checks
```

- **Fabric deployment pipelines** (dev → test → prod) per layer, **started from an Azure DevOps pipeline** through the REST API. **ADO environments** supply the approvals and the audit trail.
- **Post-deploy checks** run in the ADO pipeline: references, Direct Lake source, report bindings, variable-library value set.
- All deployment pipelines have the **same number of stages**, because binding across pipelines matches stages by position.
- **Alternative (decision D-11):** `fabric-cicd` deploying from ADO release branches. One promotion route is used, never both.

### 9.4 Configuration (variable libraries)

Each workspace has a variable library `vl_env` with **dev, test and prod value sets**:

| Reference | Variable type / mechanism |
|---|---|
| Notebook default lakehouse | *Lakehouse Auto-Binding in Git* **or** `%%configure` with name/ID variables (decision D-12) |
| Lakehouse paths in notebook code | `notebookutils.variableLibrary` |
| Shortcuts to other workspaces | Item reference variables |
| Pipeline → notebook in another workspace | GUID variables (workspace ID, notebook ID) |
| Pipeline → semantic model refresh | GUID variables (workspace ID, model ID) |
| Direct Lake model → lakehouse | Deployment rules (§8.2), not variable libraries |

### 9.5 How Fabric handles bindings (design constraints)

| Behaviour | Design response |
|---|---|
| References to items in **another workspace** never rebind on deploy or Git sync | Variable-library IDs (§9.4) |
| **Direct Lake** models stay bound to the source environment's lakehouse | Data source or parameter rules; post-deploy check |
| Pipeline **semantic model refresh** activities use the item ID, even within the same workspace | GUID variables |
| Notebook → lakehouse binding through Git needs a per-notebook setting (off by default) | Decision D-12; code review check |
| Deployments and Git updates **reset the target lakehouse's shortcuts** (and delete any not in the source) | No manual shortcuts in test or prod; review changes before deploying |
| Deployment pipelines **pair items by name and type** | `_v2` naming; no renames during the migration |
| Tables and data are never deployed | Table setup notebook per environment |

## 10. Security and access

| Area | Design |
|---|---|
| Workspace roles | Prod: BI team Admin/Member; business users get **app access** or Viewer only. Test: BI team and UAT testers. Dev: BI team. Slalom: Contributor on its development workspaces on B only; **no access to prod workspaces** |
| Identities | Service principal for Git connections and the ADO deployment pipeline; personal accounts aren't used for automation |
| Data access from B | Slalom's shortcuts to prod silver are read-only. Access is granted for the engagement period, reviewed regularly, and removed at close-out |
| Row-level security | Defined in the semantic models; role names and filter logic match the old models; tested with test users for every model change |
| OneLake security roles | Not used for this solution (preview); RLS stays in the semantic layer |
| Separation of duties | Developers (including Slalom) commit and raise pull requests; the BI team reviews, approves and deploys |
| Secrets | No secrets in item definitions; connections use Fabric connection objects or Key Vault-backed configuration |

## 11. Running the old and new stacks side by side

During the migration the solution runs **two stacks** that share silver and nothing else:

| Component | Old stack | New stack | Shared |
|---|---|---|---|
| Silver | | | ✔ prod silver |
| Gold lakehouse | `lh_gold` | `lh_gold_v2` | |
| Gold notebooks | `nb_gold_<area>` | `nb_gold_v2_<area>` | |
| Orchestration | Old branch (`run_old_<area>`) | New branch (`run_new_<area>`) | Master pipeline |
| Semantic models | `sm_<bg>_<area>` | `sm_<bg>_<area>_v2` | |
| Reports | Not yet moved | Moved (rebound) | Same report items |

**Design mechanisms:**

- **Moving a report** is a rebind. Reports move in groups (waves), and each report belongs to exactly one stack at a time.
- **Consistency:** an automated KPI cross-check compares the same KPIs across the two stacks after every load.
- **Retiring old items:** a dependency map (built from the admin scanner API) shows when an old table, notebook or model has nothing using it. It's switched off with its flag or refresh schedule, then removed.
- **Optional convergence:** an old model may be pointed at a new gold table with the same grain and logic, so the old notebook can be retired early and the duplicate compute removed.
- **End state:** the old branch, `lh_gold`, the old models and the switch flags are removed, leaving a single stack.

## 12. Observability

| Signal | Tool | Used for |
|---|---|---|
| CU use, throttling, top items | Capacity Metrics app | Baseline, budget, before/after comparison |
| Pipeline and notebook runs | Monitoring hub | Run status, duration |
| Failures and SLA breaches | Activator alerts (one per branch, with the stack named in the alert) | Operations response |
| Model refreshes / reframes | Refresh history; alerts | Data freshness |
| Data consistency | Reconciliation and KPI cross-check notebooks | Correctness while both stacks run |
| Lineage and report bindings | Admin scanner API (semantic-link-labs) | Inventory, dependency map, binding checks |
| Change history | Azure DevOps (pull requests, pipeline runs, release tags) | Audit trail |

## 13. Naming conventions

| Item | Pattern | Example |
|---|---|---|
| Workspace | `<layer>-<env>` | `transf-prod` |
| Gold lakehouse | `lh_gold_v2` | `lh_gold_v2` |
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
| Cost / capacity | New models use fewer CUs than the old ones (target in D-1); no user-facing throttling | Efficiency principle; surge protection; CU budget while both stacks run |
| Correctness | Gold and KPIs match the old stack exactly, or the difference is a documented correction | Reconciliation; KPI cross-check |
| Availability | No outage for report users when a report moves | Rebind (report identity kept); old stack keeps running until retired |
| Recoverability | A moved report can go back to the old model quickly | Rebind back to the old model, which is still refreshing; the change is mirrored in ADO |
| Rebuildability | Any environment can be rebuilt | Definitions in ADO; tables from the setup notebook; data reloaded from silver |
| Security | Users see only the data they're entitled to | RLS in models; workspace roles; separation of duties |
| Auditability | Every prod change can be traced | ADO pull requests, approvals, release tags |
| Maintainability | No environment-specific values in code | Variable libraries; build validation |

## 15. Architecture decisions

| ID | Decision | Rationale | Alternatives considered |
|---|---|---|---|
| AD-01 | All prod and non-prod workspaces stay on capacity A | The fix is efficiency; the release path is unchanged | Move non-prod to B; split prod across capacities |
| AD-02 | Capacity B is a development area only | Development and heavy testing stay off A's CUs | Using B for prod workloads |
| AD-03 | Reports are **rebound**, not replaced | Keeps URLs, apps, bookmarks and subscriptions | Publish new reports and retire the old ones |
| AD-04 | Old and new stacks run side by side, with flags by subject area | Gradual moves, rollback per wave, early CU savings | Big-bang cutover |
| AD-05 | Azure DevOps Repos as the source of truth; folder per workspace | Existing VCS; Fabric's Git integration supports ADO | — |
| AD-06 | Fabric deployment pipelines started from ADO | Smallest change to the current release process; ADO adds approvals and audit | `fabric-cicd` from release branches (open: D-11) |
| AD-07 | Environment values in variable libraries | Fabric doesn't rebind references to other workspaces | Deployment rules only; post-deploy scripts |
| AD-08 | Direct Lake bound through deployment rules, checked after every deploy | Direct Lake doesn't rebind on deployment | Rebind by hand after deployment |
| AD-09 | `_v2` naming, and no renames during the migration | Deployment pipelines pair items by name | Reuse names (overwrites old items) |
| AD-10 | RLS in the semantic models | Mature, testable; OneLake security is in preview | OneLake security roles |
| AD-11 | Notebook default lakehouse mechanism | — | Auto-binding vs `%%configure` (open: D-12) |

## 16. Risks and constraints (architecture)

| Item | Type | Response |
|---|---|---|
| Direct Lake binding isn't automatic | Platform constraint | AD-08; post-deploy check |
| References to other workspaces aren't rebound | Platform constraint | AD-07 |
| Deployments reset shortcuts | Platform constraint | No manual shortcuts; review before deploy |
| Both stacks use A's CUs at once | Design risk | Staggered schedules; CU budget; optional convergence (§11) |
| Field-name drift between old and new models | Design risk | Field names treated as a contract; compatibility measures |
| Direct Lake guardrails for A's SKU | Platform constraint | Table design standards (§7.2) |
| B must stay in A's region | Constraint | Checked before any work on B |

## 17. Open questions

| # | Question | Owner |
|---|---|---|
| Q-1 | Direct Lake flavour (on OneLake / on SQL endpoint) per model | BI team + Slalom |
| Q-2 | Promotion route: deployment pipelines started from ADO, or `fabric-cicd` (D-11) | Data Services + BI team |
| Q-3 | Notebook default lakehouse mechanism (D-12) | BI team + Slalom |
| Q-4 | Use of Spark autoscale billing for gold notebooks | Platform admin |
| Q-5 | Future of capacity B after the migration | Data Services + platform admin |

## 18. References

- [Migration runbook](fabric-capacity-migration-plan.md) · [Overview](fabric-migration-overview.md)
- [Fabric capacity and OneLake consumption – shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Deployment pipelines: autobinding](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/understand-the-deployment-process#autobinding)
- [Assign a workspace to a deployment pipeline: Direct Lake limitation](https://learn.microsoft.com/fabric/cicd/deployment-pipelines/assign-pipeline#considerations-and-limitations)
- [Dependency binding in cross-workspace deployment](https://learn.microsoft.com/fabric/cicd/cross-workspace-dependency-binding)
- [Lakehouse Git integration and deployment pipelines](https://learn.microsoft.com/fabric/data-engineering/lakehouse-git-deployment-pipelines)
- [Variable library overview](https://learn.microsoft.com/fabric/cicd/variable-library/variable-library-overview)
- [Fabric CI/CD concepts and best practices](https://learn.microsoft.com/fabric/fundamentals/understand-best-practices-fabric-cicd)
