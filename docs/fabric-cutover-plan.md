# Cutover Plan: Moving Reports to the New Gold Layer and Semantic Models

_A simple, step-by-step plan for the cutover. Full detail is in the [migration runbook](fabric-capacity-migration-plan.md)._

| | |
|---|---|
| **Version / status** | 1.0 · Draft for review · 2026-11-23 |
| **Owner** | Data Services, Enterprise Technology |

## 1. What we're doing

We're moving the existing reports from the **old semantic models** to the **new semantic models**, one group of reports (a **wave**) at a time.

- The new gold layer is built on **capacity B** (`transf-v2-prod`).
- Orchestration and all semantic models stay on **capacity A**.
- Reports are **rebound**, not replaced. Users keep the same links, apps and subscriptions.
- The old gold and old models keep running until the last wave is done, so we can always go back.

## 2. Before the first cutover (do once)

| # | Task | Owner | Done |
|---|---|---|---|
| 1 | A and B confirmed in the **same region** | Capacity admins | ☐ |
| 2 | `transf-v2-prod` deployed on B; run identity has access on A and B | BI team | ☐ |
| 3 | **Backfill complete and reconciled** (section 3) | BI team + Slalom | ☐ |
| 4 | New gold running daily in prod for at least **5 loads**, on time, with **no reports reading it** | BI team | ☐ |
| 5 | New gold reconciles with old gold (row counts, totals, keys, history) | BI team | ☐ |
| 6 | New models deployed to prod; each one reads **prod** `lh_gold_v2_ref` (checked) | BI team | ☐ |
| 7 | New models **don't refresh automatically**; elt refreshes them only after the gold load succeeds | BI team | ☐ |
| 8 | UAT signed off by BG1 and BG2 in test | Business owners | ☐ |
| 9 | Rollback rehearsed in test, and how long it took recorded | BI team | ☐ |
| 10 | Alerts working: notebook failure, new gold late, model refresh failure | BI team | ☐ |

## 3. Backfill (load the history into the new gold)

The new gold starts empty. Before any report uses it, it needs **all the history** the old gold has.

**Step 1: Decide, per table, where history comes from**

| If… | Then… |
|---|---|
| Silver still holds the full history | **Rebuild** the table from silver with the new notebooks |
| Silver doesn't hold the full history (for example, record history kept only in gold) | **Copy** the history from the old gold into the new gold once, then let the new notebooks take over |

Write the decision for each table in the inventory.

**Step 2: Run the backfill on B**
1. Run the table setup notebook in `transf-v2-prod`.
2. Load history **in batches** (for example, one year or one month at a time), oldest first.
3. Run backfills **outside the daily load window** and pause dev/test jobs on B while they run.
4. Record how long each batch takes and its CU use on B.

**Step 3: Catch up**
- Run the normal daily load from the backfill's end date up to today, so the new gold is current.
- Turn on the daily new-gold load in elt (`run_new_<area>` on).

**Step 4: Reconcile**
- Compare new gold with old gold for **current and historical** rows: row counts, key totals, distinct keys, and history rows per key.
- Every difference is either fixed, or explained and signed off by the business.

**Step 5: Keep it fresh until cutover**
- From now on, the new gold loads daily next to the old gold. If a load fails, rerun it the same day. Notebooks can be safely re-run.

**Backfill later on:** if a wave adds new gold tables, repeat steps 1–5 for those tables before that wave.

## 4. Cutover timeline (repeat for each wave)

**5 business days before (T-5)**
- Confirm the list of reports in this wave.
- Tell the users of those reports: the date, what to expect, and any planned data corrections.
- Book the cutover window (outside business hours, after the daily load).

**1 business day before (T-1)**
- Go/No-Go meeting (section 5).
- Check the last 5 new-gold loads were on time and reconciled.
- Confirm the BI team on-call person and the capacity admins are available.

**Cutover day (T-0)**

| Step | Action | Owner |
|---|---|---|
| 1 | Confirm today's **new gold load finished** and reconciled | BI team |
| 2 | Refresh the new models; confirm they succeed | BI team |
| 3 | Deploy this wave's **rebound reports** to prod through the Azure DevOps pipeline (only this wave's reports) | BI team |
| 4 | Run the post-deploy checks: each report is on the right model; each model reads prod `lh_gold_v2_ref` | BI team |
| 5 | Open every page of every report in the wave; compare key numbers with the old reports | BI team + business testers |
| 6 | Check that reports **not** in this wave are still on the old models | BI team |
| 7 | Tell users the cutover is done | Business owners |

**After cutover (T+1 to the end of hypercare: one full reporting cycle)**
- Every day: check the new gold loaded on time, the models refreshed, the KPI cross-check passed, and there are no user issues.
- At the end: business owner signs off the wave.
- Switch off old models and old gold notebooks that **nothing uses any more**. Keep them, switched off, for one more reporting cycle, then delete them.

## 5. Go/No-Go (T-1)

All must be **Yes**. A single **No** means we don't cut over.

| Check | Yes / No |
|---|---|
| Backfill complete and reconciled (current and history) | |
| Last 5 new-gold loads on time and reconciled | |
| UAT signed off for this wave's reports | |
| Every report in this wave checked in test, no broken visuals | |
| RLS tested with test users | |
| Rollback rehearsed; old models still refreshing | |
| People available: BI on-call, capacity admins, business testers | |
| Users notified | |

**Decision:** Go / No-Go · **By:** ______ · **Date:** ______

## 6. Rollback (if something goes wrong)

**Roll back if:** key numbers are wrong, report pages are broken, RLS shows the wrong data (**roll back immediately**), or the new gold is late and can't be fixed the same day.

1. Confirm the old models are refreshed and current. They read the old gold on A, so they don't depend on B.
2. **Rebind this wave's reports back to the old models.** Links and apps stay the same.
3. Make the same change in dev and commit it in Azure DevOps, so the next deployment doesn't undo it. Pause cons deployments until then.
4. Check the key numbers and tell users.
5. Find and fix the cause, then plan the wave again.

**Rollback time:** about ______ minutes (from the rehearsal).

## 7. After the last wave (close-out)

- All reports are on the new models; the KPI cross-check is retired.
- Delete the old gold, old notebooks, old models and the old/new switches, through Azure DevOps and the deployment pipelines.
- Remove Slalom's access and leftover development workspaces.
- Review A's and B's sizes now that gold processing runs only on B.

## 8. Contacts

| Role | Name | Backup |
|---|---|---|
| Cutover lead (Data Services) | ______ | ______ |
| BI team on-call | ______ | ______ |
| Capacity admin A | ______ | ______ |
| Capacity admin B | ______ | ______ |
| Business owner BG1 | ______ | ______ |
| Business owner BG2 | ______ | ______ |
