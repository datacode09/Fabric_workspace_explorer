# Fabric Capacity Migration Plan: New Gold Layer and Semantic Models

_Last updated: 2026-11-12_

## Context

- **Capacity A (prod):** hosts the production workspaces: elt, transf, and two consumption (cons) workspaces, one per business group. The end-to-end orchestration lives in the elt workspace and runs notebooks across workspaces to build the gold tables that the cons workspaces use.
- **Capacity B (dev only):** bought so development work stops slowing down prod. Slalom (SI partner) is working in `trans-dev-2` on B, building a **new gold layer** and **new semantic models** for the **existing prod reports** on A.
- **Goal:** bring the new gold layer, the pipeline changes and the new semantic models into prod on A, and move the existing prod reports onto the new models without breaking report links for users.

## Target layout

| Capacity | Workspaces | Purpose |
|---|---|---|
| **A** | elt-prod, transf-prod, cons-prod (BG1), cons-prod (BG2) | Production only |
| **B** | elt-dev/test, transf-dev/test, cons-dev/test, trans-dev-2 (sandbox) | All development and testing, including Slalom's work and heavy validation |

Moving all non-prod workspaces to B frees A's CUs for prod alone. Deployment pipeline stages can sit on different capacities. **Before you start, confirm that B is in the same region as A.** Within one region, moving a workspace is an in-place reassignment. Across regions, lakehouses, notebooks and pipelines can't be moved and would have to be rebuilt.

## 1. Get Slalom's work into your normal dev workspaces first

Connect `trans-dev-2` to Git and merge Slalom's work into the branches behind your existing dev workspaces:

- new gold notebooks and lakehouse → **transf-dev**
- orchestration changes → **elt-dev**
- new semantic models → each **cons-dev**

Those dev workspaces sit on B, and the existing deployment pipelines already promote from them to prod on A. `trans-dev-2` then becomes a sandbox only, not a route to prod. Slalom commits and opens pull requests; the internal team reviews, merges and runs the prod deployments.

## 2. Update the prod pipelines that build the new gold

- In **elt-dev**, update the orchestration pipeline to call the new silver-to-gold notebooks. Leave the old ones in place but turn them off later; don't delete them yet.
- **Watch for references to other workspaces.** Deployment pipelines remap references between items in the same workspace, but the elt pipeline calls notebooks in transf, which is a different workspace. Those references can still point at the dev notebooks after you deploy. Set the workspace IDs and notebook IDs through a **variable library** or pipeline parameters for each stage, and check them after the first deploy.
- Deploy transf, then elt, from dev to test to prod. The **new gold lakehouse arrives in prod empty**, because deployment copies item definitions, not data. Set deployment rules or variable-library values so the prod notebooks **read prod silver** and **write to prod new-gold**. After deploying, open the prod lakehouse and confirm where the shortcuts point.
- Do the heavy validation in **test on B**: point it at prod silver through shortcuts and reconcile new gold against current prod gold (row counts, key totals, SCD behaviour). Reads through a shortcut are charged to the capacity of the workspace doing the reading, so this runs on B, not A.
- Run the new gold build in prod once, at off-peak hours, before any report switches over.

## 3. Bring in the new semantic models and move the reports onto them

### Recommended: rebind in dev and let the pipeline carry it to prod

1. In each **cons-dev** workspace, rebind the existing report (the dev copy of the prod report) to Slalom's new semantic model.
2. Deploy the model and the report **together**, dev to test to prod. When a report's semantic model is deployed alongside it, the pipeline binds the prod report to the prod copy of the new model. The prod report stays the same item: same ID, URL, app placement, bookmarks and subscriptions.
3. Add a deployment rule so the prod model points at **prod new-gold**.

### Check field names before you rebind

A report only works on a new model if the names its visuals use still exist there.

- With **semantic-link-labs** in a notebook, list the tables, columns and measures in the old and new models and compare them. Also list the fields each report uses.
- For each gap, either ask Slalom to keep the old names or add compatibility measures to the new model, or change the report to use the new names.
- After rebinding in dev and test, open every page and look for broken visuals. Have each business group do UAT in test.

### If you can't use the pipeline

In prod, rebind directly with the Power BI **Rebind Report In Group** REST API, or with semantic-link-labs `report_rebind`. That works, but dev and prod then disagree until you make the same change in dev.

### Shortcut to avoid

Giving the new model the same name as the old one so the deployment overwrites it in place. You'd skip the rebind, but you'd lose your rollback, and anything else connected to that model, such as Excel files or other reports, breaks immediately.

## 4. Cutover order on A

1. Deploy the new gold pipeline, run it, and reconcile new gold against old gold in prod, using what you already checked in test.
2. Deploy the new semantic models and refresh them.
3. Deploy the rebound reports.
4. **Turn off the old gold notebooks and pause refresh on the old models right away**, so A isn't running both versions.
5. Keep the old items for one reporting cycle. **Rollback:** rebind the reports to the old model and turn its refresh back on.
6. Delete the old items, then run a deployment so dev, test and prod match again.

## 5. Protect capacity A afterward

- Turn on **surge protection** on A.
- Watch the **Capacity Metrics app** for two to four weeks.
- If Spark is still the biggest cost, look at **Spark autoscale billing** before buying more capacity.
- Keep tuning the semantic models (V-Order and OPTIMIZE on gold tables, incremental refresh, removing unused or high-cardinality columns, aggregations, and staying within the Direct Lake guardrails for your SKU). More capacity alone only puts off the throttling.

## Sources

- [Capacity reassignment restrictions and common issues](https://learn.microsoft.com/fabric/admin/portal-workspace-capacity-reassignment)
- [Fabric capacity and OneLake consumption: shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Cross-region migration handling](https://learn.microsoft.com/power-bi/support/premium-migration-how-to#cross-region-migrations-special-handling)
