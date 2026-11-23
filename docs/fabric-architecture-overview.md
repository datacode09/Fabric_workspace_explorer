# Solution Architecture Overview: Gold Layer and Semantic Models

_A summary of the design. Full detail is in the [solution architecture](fabric-solution-architecture.md)._

| | |
|---|---|
| **Version / status** | 2.0 · Final draft for review · 2026-11-23 |
| **Owner** | Data Services, Enterprise Technology |

## 1. What we're building

A more efficient **gold layer** and new **Direct Lake semantic models** for business groups BG1 and BG2, to remove throttling on Fabric capacity **A**. The **new silver-to-gold processing runs on capacity B**. **Orchestration and all semantic models stay on A.** **Existing reports are kept and rebound** to the new models. During the migration, the old and new stacks run side by side.

## 2. Architecture at a glance

```
 Capacity A  (orchestration + semantic layer)            Capacity B  (new gold processing)
 ┌──────────────────────────────────────────────┐        ┌─────────────────────────────────┐
 │ elt: master pipeline ── new branch ──────────┼──────▶ │ transf-v2: new gold notebooks   │
 │  └─ old branch ─▶ transf (old gold, retiring)│        │   reads silver (shortcut to A)  │
 │ silver ◀──────────────── shortcut ───────────┼─────── │   writes lh_gold_v2             │
 │ cons-bg1/bg2: lh_gold_v2_ref (shortcuts) ◀───┼─────── │                                 │
 │   new models (Direct Lake) + old models      │        │ dev / test / prod all on B      │
 │   reports (rebound in waves)                 │        │ (dev/test Spark capped)         │
 └──────────────────────────────────────────────┘        └─────────────────────────────────┘
       Azure DevOps Repos ─▶ dev ─▶ test ─▶ prod  (deployment pipelines started from ADO)
```

## 3. Key design principles

- **Each workload on the right capacity.** Heavy Spark transformation on B; orchestration and the semantic layer on A.
- **Reports keep working if B has a problem.** New models read new gold through **shortcuts on A** (`lh_gold_v2_ref`). If B is paused or throttled, reports keep the last loaded data.
- **Prod on B is production-grade.** B is never paused; it has named admins, monitoring, alerts and on-call.
- **Report identity is preserved.** Reports are rebound, never recreated, so URLs, apps and subscriptions survive.
- **One source of truth, explicit binding.** All definitions in Azure DevOps; environment values in variable libraries and deployment rules.
- **Security in the semantic layer.** RLS is defined and tested in the models on A.

## 4. Components

| Area | Design |
|---|---|
| **Capacity A** | `elt` (orchestration), `transf` (old gold, retiring), `cons-bg1` / `cons-bg2` (old and new models, reports, `lh_gold_v2_ref`). Surge protection on |
| **Capacity B** | `transf-v2` dev / test / prod: new gold notebooks, `lh_gold_v2`, setup notebook. Never paused. Surge protection; dev/test Spark pools capped |
| **Region** | A and B **must be in the same region** (required for Direct Lake and the shortcuts between them) |
| **Gold layer** | `lh_gold_v2` on B (Delta, V-Order, OPTIMIZE). Tables created by an idempotent setup notebook. Reads silver on A through shortcuts set from variables |
| **Orchestration** | elt on A starts the `transf-v2` notebooks on B (IDs from the variable library). Independent old and new branches, switched per subject area. A failure on B doesn't stop the old branch |
| **Semantic models** | All on A. New models: Direct Lake on `lh_gold_v2_ref`, one per business group and subject area (`sm_<bg>_<area>_v2`). Compatibility measures keep the field names existing reports use |
| **Reports** | Kept; rebound to the new models one group (wave) at a time |
| **Run identity** | Service principal or workspace identity, with access to silver (A), `lh_gold_v2` (B) and `lh_gold_v2_ref` (A) |

## 5. Who pays for what

| Work | Capacity |
|---|---|
| New gold notebooks; reading silver through shortcuts | **B** |
| Orchestration; old gold (until retired); refreshes and report queries for all models; reads through `lh_gold_v2_ref` | **A** |
| Storage of `lh_gold_v2` | B's storage bill (no CUs) |

**Running both stacks side by side:** they share silver, the master pipeline and the report items. An automated KPI cross-check compares the two after every load. Old items are switched off as soon as nothing uses them, so A's load falls with each wave. The end state is gold processing only on B, with A holding orchestration and the semantic layer.

## 6. Environments and release

| Layer | dev | test | prod |
|---|---|---|---|
| elt, transf (old), cons-bg1, cons-bg2 | A | A | A |
| **transf-v2 (new gold)** | **B** | **B** | **B** |

- **Azure DevOps:** one repo, one folder per workspace (including `/workspaces/transf-v2`). Pull requests need a BI team reviewer; build validation rejects hard-coded IDs; prod releases are tagged.
- **Promotion:** one deployment pipeline per layer, all with the same number of stages, started from ADO with approvals. Release order: transf-v2 → elt → cons. Post-deploy checks cover references across capacities, the Direct Lake source and shortcut targets.
- **Access to prod on B:** BI team, deployment service principal and run identity only. No Slalom.

**Fabric behaviours the design accounts for:**

| Behaviour | Response |
|---|---|
| References to items in another workspace or capacity never rebind | Variable-library IDs and item references |
| Direct Lake models stay bound to the source environment's lakehouse | Data source or parameter rules, checked after every deploy |
| Deployments reset a lakehouse's shortcuts | No manual shortcuts in test or prod; review before deploying |
| Items paired by name; tables never deployed | `_v2` naming, no renames; setup notebook per environment |

## 7. Operations

- **Monitoring:** Capacity Metrics app for **both A and B**; monitoring hub; Activator alerts per branch; a **new gold freshness alert**.
- **Failure modes:** if B is unavailable, new-stack reports show the last loaded data, the old branch keeps running, and the freshness alert fires. If A is throttled, surge protection and the CU budget apply.

## 8. Key decisions and open items

| Decided | Open |
|---|---|
| New gold processing (dev/test/prod) on B; orchestration and all models on A | Q-1 Direct Lake flavour per model |
| New models read new gold through shortcuts on A | Q-2 Promotion route (deployment pipelines from ADO, or `fabric-cicd`) |
| Rebind reports; old and new stacks side by side | Q-3 Notebook default lakehouse mechanism |
| ADO as source of truth; deployment pipelines per layer, started from ADO | Q-4/Q-5 Spark autoscale billing on B; B's SKU size |
| Variable libraries; `_v2` naming; RLS in models; service principal / workspace identity for runs | Q-6 Run identity type |
