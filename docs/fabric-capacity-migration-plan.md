# Fabric Capacity Migration Plan: New Gold Layer and Semantic Models

_Last updated: 2026-11-12_

## 1. Current state

**Capacity A (existing)** holds **both non-prod and prod** workspaces, split by layer:

| Layer | Non-prod (on A) | Prod (on A) |
|---|---|---|
| ELT / orchestration | elt-dev (/test) | elt-prod |
| Transformation (silver → gold) | transf-dev (/test) | transf-prod |
| Consumption, business group 1 | cons-bg1-dev (/test) | cons-bg1-prod |
| Consumption, business group 2 | cons-bg2-dev (/test) | cons-bg2-prod |

- The end-to-end pipeline lives in the **elt** workspace. It runs notebooks in several workspaces to build the gold tables that the cons workspaces use.
- **Fabric deployment pipelines** promote content from non-prod to prod, all within A.
- **Problem:** data volume keeps growing, so the semantic models are getting slower and using more compute, and A gets **throttled**.

**Capacity B (new)** is a **development area only**. It was bought so the fixes can be built without adding load to A or disrupting production.

- The SI partner **Slalom** is working in **`trans-dev-2`** on B. It reads data from capacity A.
- Slalom is building a **new gold layer** (new silver-to-gold notebooks writing to a **new gold lakehouse**) and **new semantic models** for the **existing prod reports** on A.

## 2. Target state

**Everything stays on capacity A.** The workspace layout and the deployment pipelines don't change. What changes is the content inside them:

| Layer (non-prod and prod, both on A) | What changes |
|---|---|
| elt | Orchestration calls the new silver-to-gold notebooks; the old ones are retired |
| transf | New gold notebooks plus a **new gold lakehouse**; the old gold is retired after cutover |
| cons-bg1 / cons-bg2 | **New semantic models** on new gold; the **existing reports are rebound** to them (same report IDs and URLs) |

**Capacity B** stays a development area (`trans-dev-2`). Nothing in prod or in the existing non-prod lanes depends on it. Once Slalom's work is in A, B can be scaled down, paused between engagements, or kept for future development.

The way this relieves A is by making the **semantic models and gold layer cheaper to run**. It doesn't come from moving workloads off A. So the check in step 3, that the new models really do use fewer CUs, matters most.

## 3. Step 1: build and validate on B, before anything touches A

Do the heavy work where it costs A nothing:

- Slalom builds and iterates in `trans-dev-2` on B, reading A's silver data through **shortcuts**. Reads through a shortcut are charged to the capacity of the workspace doing the reading, so this runs on **B**, not A. Storage stays billed where the data lives.
- **Reconcile in B:** run the new notebooks against **prod silver** for several cycles and compare new gold with current prod gold: row counts, key totals, SCD behaviour.
- **Load-test the new semantic models in B** and check their CU cost in the Capacity Metrics app. That confirms the fix will reduce load on A before it gets there.
- **Compare field names** (see step 4) between old and new models while still in B, so field-name gaps are fixed before the merge.

## 4. Step 2: merge Slalom's work into the existing non-prod lanes on A

`trans-dev-2` isn't part of the deployment pipelines, so it should **not** become a separate route to prod. Bring the work into the existing non-prod workspaces on A through Git:

- new gold notebooks and lakehouse → **transf-dev**
- orchestration changes → **elt-dev**
- new semantic models → **cons-bg1-dev** / **cons-bg2-dev**

Process:

1. If the existing dev workspaces aren't connected to Git yet, connect them and **commit their current state** as a baseline.
2. Connect `trans-dev-2` to a **feature branch** in the same repo. Slalom commits there.
3. Slalom opens a **pull request**; the internal BI/data team reviews notebook diffs, gold schema changes and new items.
4. Merge, then **sync the dev workspaces on A from Git**.

**Permissions:** Slalom commits and opens pull requests. The internal team approves and runs prod deployments.

**Item names matter.** Deployment pipelines pair items across stages by **name and type**. New or renamed items are created as new items in the next stage.

**Clean up bindings to B before merging.** Notebooks built in `trans-dev-2` are pinned to its lakehouses and may contain hard-coded workspace/lakehouse GUIDs or ABFS paths that point at B. Replace them with **variable library** values or parameters, so each stage on A points at its own lakehouses. Check where the shortcuts point too.

If Git isn't an option, the fallbacks are copying items by hand (export/import notebooks, PBIP for reports) or using the open-source **`fabric-cicd`** Python library, which deploys from Git to each workspace and swaps IDs per environment through a parameter file. Copying by hand is the riskiest.

## 5. Step 3: update the pipelines and promote the new gold to prod

- In **elt-dev**, update the orchestration pipeline to call the new silver-to-gold notebooks. Leave the old ones in place but turn them off later; don't delete them yet.
- **Watch for references to other workspaces.** Deployment pipelines remap references between items in the same workspace, but the elt pipeline calls notebooks in transf, which is a different workspace. Those references can still point at the dev notebooks after deployment. Set workspace IDs and notebook IDs through a **variable library** or pipeline parameters for each stage, and check them after each deploy.
- Deploy **transf, then elt**, dev to test to prod. The **new gold lakehouse arrives empty** in each stage, because deployment copies item definitions, not data. Set deployment rules or variable-library values so the prod notebooks **read prod silver** and **write to prod new-gold**. After deploying, open the prod lakehouse and confirm where the shortcuts point.
- **Keep test on A light.** The full reconciliation already ran on B (step 1). In test on A, run only a short smoke test, at **off-peak hours**.
- **Protect the gold contract.** Use lineage and impact analysis to see which semantic models and reports use each gold table that changed.
- In prod, run the new gold build once, at off-peak hours, before any report switches over.

## 6. Step 4: bring in the new semantic models and move the reports onto them

### Recommended: rebind in dev and let the pipeline carry it to prod

1. In each **cons-dev** workspace on A, rebind the existing report (the dev copy of the prod report) to Slalom's new semantic model.
2. Deploy the **model and the report together**, dev to test to prod. When a report's semantic model is deployed alongside it, the pipeline binds the prod report to the prod copy of the new model. The prod report stays the same item: same ID, URL, app placement, bookmarks and subscriptions.
3. Add a deployment rule so the prod model points at **prod new-gold**.

### Check field names before you rebind

A report only works on a new model if the names its visuals use still exist there.

- With **semantic-link-labs** in a notebook, list the tables, columns and measures in the old and new models and compare them. Also list the fields each report uses.
- For each gap, either ask Slalom to keep the old names or add compatibility measures to the new model, or change the report to use the new names.
- After rebinding in dev and test, open every page and look for broken visuals. Each business group does **UAT in test**.

### If you can't use the pipeline

In prod, rebind directly with the Power BI **Rebind Report In Group** REST API, or with semantic-link-labs `report_rebind`. That works, but dev and prod then disagree until you make the same change in dev.

### Shortcut to avoid

Giving the new model the same name as the old one so the deployment overwrites it in place. You'd skip the rebind, but you'd lose your rollback, and anything else connected to that model, such as Excel files or other reports, breaks immediately.

## 7. Step 5: cutover order on A

1. Deploy the new gold pipeline to prod, run it, and spot-check new gold against old gold, using what you already reconciled on B.
2. Deploy the new semantic models and refresh them.
3. Deploy the rebound reports. If any reports were replaced rather than rebound, update the workspace app. Repoint subscriptions and Excel files connected to the old models, and tell each business group the cutover date.
4. **Turn off the old gold notebooks and pause refresh on the old models right away**, so A isn't running both versions.
5. Keep the old items for one reporting cycle. **Rollback:** rebind the reports to the old model and turn its refresh back on.
6. Delete the old items, in non-prod and prod, then run a deployment so dev, test and prod match again.

## 8. Step 6: protect capacity A afterward

- Turn on **surge protection** on A, so background spikes, including non-prod jobs on A, can't starve prod.
- Watch the **Capacity Metrics app** for two to four weeks and compare CU use with the period before cutover.
- Schedule non-prod refreshes and Spark jobs on A **away from prod peak hours**.
- If Spark is still the biggest cost, look at **Spark autoscale billing** before buying more capacity.
- Keep tuning the semantic models: V-Order and OPTIMIZE on gold tables, partitioning and incremental refresh, removing unused or high-cardinality columns, aggregations, choosing Import or Direct Lake for each model, and staying within the Direct Lake guardrails for your SKU.
- **Decide what to do with B:** keep it for future development, scale it down, or pause it between engagements. A paused capacity still pays for its stored data, so clean up what you don't need.

## Summary checklist

- [ ] Slalom builds in `trans-dev-2` on B, reading A's silver through shortcuts
- [ ] Reconcile new gold against prod gold, and load-test the new models, **on B**
- [ ] Compare field names between old and new models; fix gaps on B
- [ ] Connect the existing dev workspaces on A and `trans-dev-2` to Git; set a baseline commit
- [ ] Merge Slalom's work into elt-dev / transf-dev / cons-dev on A through pull requests
- [ ] Replace bindings to B with variable libraries and deployment rules (cross-workspace IDs, default lakehouses, shortcuts)
- [ ] Smoke test and UAT in test on A, at off-peak hours
- [ ] Rebind reports in dev; deploy model and report together to prod
- [ ] Cut over; turn off old gold and old model refresh immediately
- [ ] Keep old items for one reporting cycle for rollback, then delete them
- [ ] Turn on surge protection on A; monitor the Capacity Metrics app
- [ ] Decide B's future (keep, scale down, or pause)

## Sources

- [Fabric capacity and OneLake consumption: shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Capacity reassignment restrictions and common issues](https://learn.microsoft.com/fabric/admin/portal-workspace-capacity-reassignment)
