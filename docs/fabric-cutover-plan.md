# Cutover Plan: Moving Reports to the New Gold Layer and Semantic Models

_A simple, step-by-step plan for the cutover, grouped by team. Full detail is in the [migration runbook](fabric-capacity-migration-plan.md)._

| | |
|---|---|
| **Version / status** | 1.1 · Draft for review · 2026-11-23 |
| **Owner** | Data Services, Enterprise Technology |

## 1. What we're doing

We're moving the existing reports from the **old semantic models** to the **new semantic models**, one group of reports (a **wave**) at a time.

- The new gold layer is built on **capacity B** (`transf-v2-prod`).
- Orchestration and all semantic models stay on **capacity A**.
- Reports are **rebound**, not replaced. Users keep the same links, apps and subscriptions.
- The old gold and old models keep running until the last wave is done, so we can always go back.

## 2. Who does what

| Team | Owns |
|---|---|
| **Data Engineering (DE)** | `transf-v2` workspaces on B, new gold notebooks, `lh_gold_v2`, backfill, daily gold loads, elt orchestration, gold reconciliation, load alerts, run identity |
| **BI** | Semantic models (old and new), `lh_gold_v2_ref` shortcuts, report rebinding, report checks, RLS, KPI cross-check, UAT support, report alerts |
| **Capacity admins** | Capacities A and B: region, size, surge protection, monitoring |
| **Business owners (BG1, BG2)** | UAT, sign-off, user communication, Go/No-Go agreement |

**Handoffs between DE and BI**

| Handoff | From → To | Signal |
|---|---|---|
| H1 – Gold ready for models | DE → BI | Backfill reconciled; daily loads on time for 5 days |
| H2 – Daily gold ready | DE → BI | Today's new gold load succeeded and passed checks; BI can refresh the new models |
| H3 – Old gold no longer needed | BI → DE | No model reads an old gold table any more; DE can switch off its notebook |

## 3. Before the first cutover (do once)

**Data Engineering**

| # | Task | Done |
|---|---|---|
| DE-1 | Deploy `transf-v2-prod` on B through the Azure DevOps pipeline | ☐ |
| DE-2 | Set up the run identity with access to silver (A) and `lh_gold_v2` (B); test it end to end | ☐ |
| DE-3 | Complete the **backfill** (section 4) and reconcile it with the old gold | ☐ |
| DE-4 | Turn on the daily new-gold load in elt (`run_new_<area>` on); run it for at least **5 loads**, on time | ☐ |
| DE-5 | Set up alerts: notebook failure, new gold late | ☐ |
| DE-6 | Make sure a failure of the new branch doesn't stop the old branch (tested in test) | ☐ |
| DE-7 | Send **handoff H1** to BI | ☐ |

**BI**

| # | Task | Done |
|---|---|---|
| BI-1 | Deploy `lh_gold_v2_ref` (shortcuts to `lh_gold_v2`) in the prod cons workspaces; check they point at **prod** | ☐ |
| BI-2 | Deploy the new models to prod; check each reads **prod** `lh_gold_v2_ref` | ☐ |
| BI-3 | Turn off automatic refresh on the new models; set elt to refresh them only after handoff H2 | ☐ |
| BI-4 | Rebind this wave's reports in **dev**; deploy to **test**; check every page | ☐ |
| BI-5 | Test RLS with test users | ☐ |
| BI-6 | Support UAT in test; get sign-off from BG1 and BG2 | ☐ |
| BI-7 | Rehearse the rollback in test and record how long it takes | ☐ |
| BI-8 | Set up alerts: model refresh failure; KPI cross-check failure | ☐ |

**Capacity admins and business owners**

| # | Task | Done |
|---|---|---|
| CA-1 | Confirm A and B are in the **same region**; surge protection on both; B never paused | ☐ |
| BO-1 | Confirm the reports in each wave, and the acceptance tolerances | ☐ |

## 4. Backfill (Data Engineering)

The new gold starts empty. Before any report uses it, it needs **all the history** the old gold has.

| Step | Task | Done |
|---|---|---|
| 1 | **Decide per table where history comes from.** If silver holds the full history, **rebuild** from silver. If not (for example, record history kept only in gold), **copy** the history from the old gold once, then let the new notebooks take over. Write the decision in the inventory | ☐ |
| 2 | Run the table setup notebook in `transf-v2-prod` | ☐ |
| 3 | Load history **in batches** (for example, one year or one month at a time), oldest first, **outside the daily load window**. Pause dev/test jobs on B while it runs. Record time and CU per batch | ☐ |
| 4 | **Catch up:** run the normal daily load from the backfill's end date up to today | ☐ |
| 5 | **Reconcile** new gold with old gold for **current and historical** rows: row counts, key totals, distinct keys, history rows per key. Fix each difference, or get business sign-off for it | ☐ |
| 6 | Keep the new gold loading daily next to the old gold until cutover. Rerun any failed load the same day | ☐ |

**BI's part:** review the reconciliation for the tables its models use, and confirm KPIs match before accepting handoff H1.

**Later waves:** if a wave adds new gold tables, DE repeats steps 1–6 for those tables before that wave.

## 5. Cutover timeline (repeat for each wave)

### 5 business days before (T-5)

| Team | Task |
|---|---|
| DE | Confirm the gold tables this wave needs are loaded and reconciled |
| BI | Confirm the list of reports in this wave, and that each one is checked in test |
| Business owners | Tell users: the date, what to expect, any planned data corrections |
| All | Book the cutover window (outside business hours, after the daily load) |

### 1 business day before (T-1)

| Team | Task |
|---|---|
| DE | Report the last 5 new-gold loads: on time, reconciled, any incidents |
| BI | Report the test results: report checks, RLS, UAT sign-off, rollback rehearsal |
| All | **Go/No-Go meeting** (section 6). Confirm on-call people from DE and BI, and capacity admins |

### Cutover day (T-0)

| Step | Team | Task |
|---|---|---|
| 1 | DE | Confirm today's **new gold load finished** and passed checks; send **handoff H2** |
| 2 | BI | Refresh the new models; confirm they succeed |
| 3 | BI | Deploy this wave's **rebound reports** to prod through the Azure DevOps pipeline (only this wave's reports) |
| 4 | BI | Run the post-deploy checks: each report on the right model; each model reads prod `lh_gold_v2_ref` |
| 5 | BI + business testers | Open every page of every report in the wave; compare key numbers with the old reports |
| 6 | BI | Check that reports **not** in this wave are still on the old models |
| 7 | Business owners | Tell users the cutover is done |

### After cutover (hypercare: one full reporting cycle)

| Team | Every day |
|---|---|
| DE | New gold loaded on time; old branch still running; no notebook failures on B; send handoff H2 |
| BI | New models refreshed after H2; KPI cross-check passed; no report or RLS issues from users |
| Capacity admins | No throttling on A; no dev/test contention on B |

**At the end of hypercare**

| Team | Task |
|---|---|
| Business owners | Sign off the wave |
| BI | Pause refresh on old models that **no report uses any more**; send **handoff H3** to DE for the old gold tables they read |
| DE | Switch off the old gold notebooks covered by H3. Keep them, switched off, for one more reporting cycle, then delete them |

## 6. Go/No-Go (T-1)

All must be **Yes**. A single **No** means we don't cut over.

| Team | Check | Yes / No |
|---|---|---|
| DE | Backfill complete and reconciled (current and history) | |
| DE | Last 5 new-gold loads on time and reconciled | |
| DE | Alerts working; on-call person available | |
| BI | Every report in this wave checked in test, no broken visuals | |
| BI | RLS tested with test users | |
| BI | Rollback rehearsed; old models still refreshing | |
| BI | On-call person available | |
| Business | UAT signed off; users notified | |
| Capacity admins | A and B healthy; no throttling | |

**Decision:** Go / No-Go · **By:** ______ · **Date:** ______

## 7. Rollback (if something goes wrong)

**Roll back if:** key numbers are wrong, report pages are broken, RLS shows the wrong data (**roll back immediately**), or the new gold is late and can't be fixed the same day.

| Step | Team | Task |
|---|---|---|
| 1 | DE | Confirm the old gold branch is running and current. It's on A, so it doesn't depend on B |
| 2 | BI | Confirm the old models are refreshed and current |
| 3 | BI | **Rebind this wave's reports back to the old models.** Links and apps stay the same |
| 4 | BI | Make the same change in dev and commit it in Azure DevOps; pause cons deployments until it's merged |
| 5 | BI + business | Check the key numbers; tell users |
| 6 | DE + BI | Find and fix the cause, then plan the wave again |

**Rollback time:** about ______ minutes (from the rehearsal).

## 8. After the last wave (close-out)

| Team | Task |
|---|---|
| BI | Confirm all reports are on the new models; retire the KPI cross-check; delete the old models through Azure DevOps and the deployment pipelines |
| DE | Delete the old gold notebooks, old gold lakehouse and the old/new switches in elt, through Azure DevOps and the deployment pipelines |
| DE + BI | Remove Slalom's access and leftover development workspaces |
| Capacity admins | Review A's and B's sizes now that gold processing runs only on B |

## 9. Contacts

| Role | Name | Backup |
|---|---|---|
| Cutover lead (Data Services) | ______ | ______ |
| Data Engineering lead / on-call | ______ | ______ |
| BI lead / on-call | ______ | ______ |
| Capacity admin A | ______ | ______ |
| Capacity admin B | ______ | ______ |
| Business owner BG1 | ______ | ______ |
| Business owner BG2 | ______ | ______ |
