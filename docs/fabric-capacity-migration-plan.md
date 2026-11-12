# Fabric Capacity Migration Plan: New Gold Layer and Semantic Models

_Last updated: 2026-11-12_

## 1. Current state

**Capacity A (existing)** holds **both non-prod and prod** workspaces, split by layer:

| Layer | Non-prod (on A today) | Prod (on A) |
|---|---|---|
| ELT / orchestration | elt-dev (/test) | elt-prod |
| Transformation (silver → gold) | transf-dev (/test) | transf-prod |
| Consumption, business group 1 | cons-bg1-dev (/test) | cons-bg1-prod |
| Consumption, business group 2 | cons-bg2-dev (/test) | cons-bg2-prod |

- The end-to-end pipeline lives in the **elt** workspace. It runs notebooks in several workspaces to build the gold tables that the cons workspaces use.
- **Fabric deployment pipelines** promote content from non-prod to prod.
- **Problem:** data volume keeps growing, so the semantic models are getting slower and using more compute. Prod and non-prod share A's CUs, so dev/test refreshes and Spark runs add to the load and A gets **throttled**.

**Capacity B (new)** was bought for **development only**, so the fixes can be built without disrupting production.

- The SI partner **Slalom** is working in **`trans-dev-2`** on B. It reads data from capacity A.
- Slalom is building a **new gold layer** (new silver-to-gold notebooks writing to a **new gold lakehouse**) and **new semantic models** for the **existing prod reports** on A.
- Once their work is done, the changes need to reach prod on A: the pipelines updated, the new gold built, and the existing reports **moved onto the new semantic models**.

## 2. Target state

| Capacity | Workspaces | Purpose |
|---|---|---|
| **A** | elt-prod, transf-prod, cons-bg1-prod, cons-bg2-prod | **Prod only**: all of A's CUs serve production |
| **B** | elt/transf/cons dev and test, plus `trans-dev-2` as Slalom's sandbox | **All non-prod**: development, testing, heavy validation |

Prod stays on A. The existing prod reports keep their IDs and URLs. They are rebound to the new semantic models, which read from the new gold lakehouse.

### Do we need both capacities?

Not strictly: scaling A up (for example F64 → F128) would also stop the throttling, and one large pool absorbs spikes better than two smaller ones. Since B is already bought, its value is **isolation**. Non-prod work, including Slalom's development and the validation runs, stops using prod's CUs. If, once the models are tuned, use is steady and B sits mostly idle, consider moving back to a single larger capacity.

## 3. Step 0: move non-prod workspaces from A to B (recommended)

- Reassign elt, transf and cons **dev/test** workspaces from A to B. Within the same region, this is an in-place reassignment: items keep their IDs, and the **deployment pipelines keep working**, because each stage can sit on a different capacity.
- Do it in a quiet window. **Reassigning a workspace cancels its running jobs.**
- Afterwards, look in workspace settings for a banner saying some items stayed on A. Don't pause or delete anything on A while an item is still attached to it.
- **Check the region first.** If B is in a different region from A, lakehouses, notebooks, pipelines and large-format semantic models can't be moved, and would have to be rebuilt.

This is the quickest throttling relief available: dev/test refreshes and Spark runs stop using prod's CUs.

**If non-prod stays on A instead:** the rest of this plan still applies. The difference is that the validation in step 2 runs on A's CUs, so schedule those runs off-peak and keep them short.

## 4. Step 1: get Slalom's work into your normal dev workspaces

`trans-dev-2` isn't part of the deployment pipelines, so it should **not** become a second route to prod. Connect it to Git and merge Slalom's work into the branches behind your existing dev workspaces:

- new gold notebooks and lakehouse → **transf-dev**
- orchestration changes → **elt-dev**
- new semantic models → **cons-bg1-dev** / **cons-bg2-dev**

Process:

1. If the existing dev workspaces aren't connected to Git yet, connect them and **commit their current state** as a baseline.
2. Connect `trans-dev-2` to a **feature branch** in the same repo. Slalom commits there.
3. Slalom opens a **pull request**; the internal BI/data team reviews notebook diffs, gold schema changes and new items.
4. Merge, then **sync the dev workspaces from Git**.
5. Keep `trans-dev-2` as a sandbox or retire it.

**Permissions:** Slalom commits and opens pull requests. The internal team approves and runs prod deployments.

**Item names matter.** Deployment pipelines pair items across stages by **name and type**. New or renamed items are created as new items in the next stage.

If Git isn't an option, the fallbacks are: swap `trans-dev-2` in as the pipeline's dev stage (anything in the old dev workspace that isn't also in `trans-dev-2` drops out of the promotion path), or copy items by hand (export/import notebooks, PBIP for reports). Both are riskier. The open-source **`fabric-cicd`** Python library can also deploy from Git to each workspace, swapping IDs per environment through a parameter file.

## 5. Step 2: update the prod pipelines that build the new gold

- In **elt-dev**, update the orchestration pipeline to call the new silver-to-gold notebooks. Leave the old ones in place but turn them off later; don't delete them yet.
- **Watch for references to other workspaces.** Deployment pipelines remap references between items in the same workspace, but the elt pipeline calls notebooks in transf, which is a different workspace. Those references can still point at the dev notebooks after deployment. Set workspace IDs and notebook IDs through a **variable library** or pipeline parameters for each stage, and check them after the first deploy.
- **Clean up bindings to dev.** Notebooks built in `trans-dev-2` are probably pinned to its default lakehouse, and may contain hard-coded workspace/lakehouse GUIDs or ABFS paths. Slalom reads from capacity A in dev, so check where its shortcuts point too.
- Deploy **transf, then elt**, dev to test to prod. The **new gold lakehouse arrives in prod empty**, because deployment copies item definitions, not data. Set deployment rules or variable-library values so the prod notebooks **read prod silver** and **write to prod new-gold**. After deploying, open the prod lakehouse and confirm where the shortcuts point.
- **Do the heavy validation in test** (on B, once step 0 is done). Point test at **prod silver** through shortcuts, run the new notebooks for several cycles, and reconcile new gold against current prod gold: row counts, key totals, SCD behaviour. Reads through a shortcut are charged to the capacity of the workspace doing the reading, so this runs on B, not A.
- **Protect the gold contract.** Use lineage and impact analysis to see which semantic models and reports use each gold table that changed.
- In prod, run the new gold build once, at off-peak hours, before any report switches over.

## 6. Step 3: bring in the new semantic models and move the reports onto them

### Recommended: rebind in dev and let the pipeline carry it to prod

1. In each **cons-dev** workspace, rebind the existing report (the dev copy of the prod report) to Slalom's new semantic model.
2. Deploy the **model and the report together**, dev to test to prod. When a report's semantic model is deployed alongside it, the pipeline binds the prod report to the prod copy of the new model. The prod report stays the same item: same ID, URL, app placement, bookmarks and subscriptions.
3. Add a deployment rule so the prod model points at **prod new-gold**.

### Check field names before you rebind

A report only works on a new model if the names its visuals use still exist there.

- With **semantic-link-labs** in a notebook, list the tables, columns and measures in the old and new models and compare them. Also list the fields each report uses.
- For each gap, either ask Slalom to keep the old names or add compatibility measures to the new model, or change the report to use the new names.
- After rebinding in dev and test, open every page and look for broken visuals. Each business group does **UAT in test** against prod data.
- **Load-test** the new models in test and check their CU cost in the Capacity Metrics app, to confirm they will actually reduce the load on A.

### If you can't use the pipeline

In prod, rebind directly with the Power BI **Rebind Report In Group** REST API, or with semantic-link-labs `report_rebind`. That works, but dev and prod then disagree until you make the same change in dev.

### Shortcut to avoid

Giving the new model the same name as the old one so the deployment overwrites it in place. You'd skip the rebind, but you'd lose your rollback, and anything else connected to that model, such as Excel files or other reports, breaks immediately.

## 7. Step 4: cutover order on A

1. Deploy the new gold pipeline, run it, and spot-check new gold against old gold in prod, using what you already checked in test.
2. Deploy the new semantic models and refresh them.
3. Deploy the rebound reports. If any reports were replaced rather than rebound, update the workspace app. Repoint subscriptions and Excel files connected to the old models, and tell each business group the cutover date.
4. **Turn off the old gold notebooks and pause refresh on the old models right away**, so A isn't running both versions.
5. Keep the old items for one reporting cycle. **Rollback:** rebind the reports to the old model and turn its refresh back on.
6. Delete the old items, then run a deployment so dev, test and prod match again.

## 8. Step 5: protect capacity A afterward

- Turn on **surge protection** on A.
- Watch the **Capacity Metrics app** on both capacities for two to four weeks.
- If Spark is still the biggest cost, look at **Spark autoscale billing** before buying more capacity.
- Keep tuning the semantic models: V-Order and OPTIMIZE on gold tables, partitioning and incremental refresh, removing unused or high-cardinality columns, aggregations, choosing Import or Direct Lake for each model, and staying within the Direct Lake guardrails for your SKU. **More capacity alone only puts off the throttling.**

## Summary checklist

- [ ] Confirm that A and B are in the same region
- [ ] Move non-prod workspaces from A to B (or decide to keep them on A and schedule validation off-peak)
- [ ] Connect the dev workspaces and `trans-dev-2` to Git; set a baseline commit
- [ ] Merge Slalom's work into elt-dev / transf-dev / cons-dev through pull requests
- [ ] Set variable libraries and deployment rules for cross-workspace IDs, default lakehouses and shortcuts
- [ ] Validate new gold and the new models in test (reconciliation, UAT, load test)
- [ ] Compare field names between old and new models; fix gaps
- [ ] Rebind reports in dev; deploy model and report together to prod
- [ ] Cut over; turn off old gold and old model refresh
- [ ] Keep old items for one reporting cycle for rollback, then delete them
- [ ] Turn on surge protection; monitor the Capacity Metrics app

## Sources

- [Capacity reassignment restrictions and common issues](https://learn.microsoft.com/fabric/admin/portal-workspace-capacity-reassignment)
- [Fabric capacity and OneLake consumption: shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-capacity-consumption#shortcuts-capacity-usage)
- [OneLake compute and storage consumption](https://learn.microsoft.com/fabric/onelake/onelake-consumption)
- [Direct Lake overview](https://learn.microsoft.com/fabric/fundamentals/direct-lake-overview)
- [Cross-region migration handling](https://learn.microsoft.com/power-bi/support/premium-migration-how-to#cross-region-migrations-special-handling)
