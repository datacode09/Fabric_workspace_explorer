# Solution Architecture Overview: Gold Layer and Semantic Models

_A summary of the design. Full detail is in the [solution architecture](fabric-solution-architecture.md)._

| | |
|---|---|
| **Version / status** | 1.0 · Draft for review · 2026-11-12 |
| **Owner** | Data Services, Enterprise Technology |

## 1. What we're building

A more efficient **gold layer** and new **Direct Lake semantic models** for business groups BG1 and BG2. They replace the current ones to remove throttling on Fabric capacity **A**. **Existing reports are kept and rebound** to the new models. During the migration, the old and new stacks run side by side.

## 2. Architecture at a glance

```
 Capacity B (development)                   Capacity A (prod + non-prod; dev / test / prod)
 ┌─────────────────────────┐   ADO PR    ┌──────────────────────────────────────────────────┐
 │ Slalom branched         │ ──────────▶ │ elt      : master pipeline (old + new branches)  │
 │ workspaces (build,      │             │ transf   : silver ─▶ lh_gold (old)               │
 │ validation, load tests) │             │                    ─▶ lh_gold_v2 (new)           │
 │ reads prod silver via   │             │ cons-bg1 : sm_*_v2 (Direct Lake) ─▶ reports      │
 │ shortcuts (paid by B)   │             │ cons-bg2 : sm_*_v2 (Direct Lake) ─▶ reports      │
 └─────────────────────────┘             └──────────────────────────────────────────────────┘
          Azure DevOps Repos ─▶ dev ─▶ test ─▶ prod  (deployment pipelines started from ADO)
```

## 3. Key design principles

- **Prod stays on capacity A.** B is a development area, and nothing on A depends on it.
- **Efficiency over capacity.** Load drops because the gold layer and models cost less to run, not because more compute is added.
- **Report identity is preserved.** Reports are rebound, never recreated, so URLs, apps and subscriptions survive.
- **One source of truth.** All item definitions live in Azure DevOps. There are no UI edits outside development.
- **Explicit binding.** Environment values come from variable libraries and deployment rules, never hard-coded.
- **Security in the semantic layer.** RLS is defined and tested in the models.

## 4. Components

| Area | Design |
|---|---|
| **Capacities** | **A**: all prod and non-prod workspaces. **B**: development only. Same region. Surge protection on A; Spark autoscale billing is the first relief valve |
| **Workspaces** | Per environment: `elt` (orchestration), `transf` (silver → gold), `cons-bg1` / `cons-bg2` (models and reports) |
| **Gold layer** | `lh_gold_v2` (Delta, V-Order, scheduled OPTIMIZE, partitioning only where needed). Tables created by an idempotent setup notebook. Silver read through shortcuts set from variables |
| **Orchestration** | One master pipeline with independent **old and new branches**, switched per subject area (`run_old_<area>` / `run_new_<area>`). A failure in one branch doesn't stop the other |
| **Semantic models** | Direct Lake, one per business group and subject area (`sm_<bg>_<area>_v2`). Compatibility measures keep the field names existing reports use |
| **Reports** | Kept as-is; rebound to the new models one group (wave) at a time. Each report is on exactly one model at a time |

## 5. Running both stacks side by side

| Shared | Separate (old vs new) |
|---|---|
| Prod silver; master pipeline; report items | Gold lakehouse, gold notebooks, semantic models, orchestration branch |

- **Consistency:** an automated KPI cross-check compares the two stacks after every load.
- **Retiring old items:** a dependency map (from the admin scanner API) shows when an old item has nothing using it. It's then switched off, and removed later.
- **End state:** a single stack, with the old branch, `lh_gold`, the old models and the switch flags removed.

## 6. Environments and release

| Environment | Capacity | Changed by |
|---|---|---|
| Slalom development | B | Feature branches (one branched workspace per layer) |
| dev | A | Sync from Azure DevOps `dev` branch |
| test / prod | A | Fabric deployment pipelines, **started from Azure DevOps** with approvals and post-deploy checks |

- **Azure DevOps:** one folder per workspace; pull requests with a BI team reviewer; build validation that rejects hard-coded IDs and references to B; release tags for prod.
- **Variable libraries** (dev, test and prod values) hold workspace, lakehouse, notebook and model IDs, and shortcut targets.

**Fabric behaviours the design accounts for:**

| Behaviour | Response |
|---|---|
| References to items in another workspace never rebind | Variable-library IDs |
| Direct Lake models stay bound to the source environment's lakehouse | Data source or parameter rules, checked after every deploy |
| Deployments reset a lakehouse's shortcuts | No manual shortcuts in test or prod; review before deploying |
| Items are paired by name across stages | `_v2` naming; no renames |
| Tables and data are never deployed | Setup notebook per environment |

## 7. Security, cost and operations

- **Security:** RLS in the models; Slalom has no access to prod; service principal for Git and deployments; separation of duties (developers raise pull requests, the BI team approves and deploys).
- **Cost:** shortcut reads are charged to the reading workspace's capacity, so validation on B costs A nothing. A CU budget applies while both stacks run.
- **Observability:** Capacity Metrics app (CU), monitoring hub and Activator alerts (runs, refreshes), reconciliation and KPI cross-check notebooks (correctness), Azure DevOps history (audit).

## 8. Key decisions and open items

| Decided | Open |
|---|---|
| All workspaces stay on A; B is development only | Q-1 Direct Lake flavour per model (OneLake / SQL endpoint) |
| Rebind reports, don't replace them | Q-2 Promotion route: deployment pipelines from ADO, or `fabric-cicd` |
| Old and new stacks side by side, with flags by subject area | Q-3 Notebook default lakehouse mechanism |
| Azure DevOps as source of truth; deployment pipelines started from ADO | Q-4 Spark autoscale billing for gold notebooks |
| Variable libraries for environment values; `_v2` naming; RLS in models | Q-5 Future of capacity B |
