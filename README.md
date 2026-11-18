# DimCustAccount (Courtesy) – SCD Type 2 history fix

## Goal

Make the Snowflake ELT for the Courtesy customer-account dimension return the **same history as the legacy SQL Server warehouse**.

The test case is `SourceSiteUseID = 5494035` (Oracle account 7513, Delta Marriott Hunt Valley):

| | Rows | What they show |
|---|---|---|
| SQL Server (legacy) | **15** | Every change of Management Co, Brand and Buying Group, 2000-01-01 → today |
| Snowflake (today) | **1** | Only the current values |

## Full Goal/high-level:  

Description -- Recap: Goal: make DEV's Courtesy DimCustAccount keep full SCD2 history, matching SQL Server's 15 rows for site use 5494035. Current state: DEV has fresh ORG 83 data copied from PROD, and the README is written. Next action: your go-ahead to create the DEV views from scripts 02 and 03.
 
## Recap Detail: "what is wrong"

**Data flow (PROD):**

```
RAW_PROD.SHARED.VW_DIMCUSTACCOUNT_COURTESY        (Oracle EBS source view, ORG_ID 83)
   │  USP_ADMINMERGE, ADMINMERGE row 140  (config key: SourceSiteUseID, EffDate)
   ▼
EDW_PROD.DWSHARED.DIMCUSTACCOUNT                  (shared with Keefe; row 141)
   │  view EDW_PROD.DWSHARED_CP.DIMCUSTACCOUNT (ORG_ID = 83)
   │  ADMINMERGE row 31, key DimCustAccountKey (+ delete step)
   ▼
EDW_PROD.COURTESY.DIMCUSTACCOUNTBASETBL
   (also: DWSHARED → EDW_PROD.FINANCIALS.DIMCUSTACCOUNT, row 56)
```

**Root causes:**

1. **The source view only gives today's state.** `VW_DIMCUSTACCOUNT_COURTESY` takes Management Co, Brand and Buying Group from `STGACCOUNTDATEGRPBRNDMNG`. That staging table keeps only the relationships active today, one row per account. The view's `EffDate` / `ExpDate` are hard-coded `'1/1/2000'` / `'12/31/4712'`.
2. **The merge overwrites instead of adding versions.** ADMINMERGE row 140 says to merge on `SourceSiteUseID, EffDate`. But `MANUAL_STMT_FLAG = TRUE` makes it run a hand-written `SQLSTMT1` that merges on **`SourceSiteUseID` only**, so every change overwrites the single row.
3. **The history still exists in RAW.** `RAW_PROD.ORACLE.HZ_RELATIONSHIPS` holds every dated relationship (active and inactive). For account 7513 it has 12 rows back to 2011, which rebuild **exactly** the 15 SQL Server rows.

## Target state

- **One row per version.** `DWSHARED.DIMCUSTACCOUNT`, `COURTESY.DIMCUSTACCOUNTBASETBL` and `FINANCIALS.DIMCUSTACCOUNT` hold one row per version of each Courtesy site use, with real `EffDate` / `ExpDate`.
- **One current row.** The current version has `ExpDate = 4712-12-31`.
- **Some columns change over time, others don't.**
  - Versioned (Type 2): Management Co, Brand, Buying Group, Owner, InnCode.
  - Same value on every version, as in SQL Server (Type 1): all other attributes (address, email, phone, sales rep, `CurrentManagementCo`, `CurrentBrand`, etc.).
- **Daily loads keep adding versions automatically**, using the existing AdminMerge framework. No new procedure is needed.

## Major changes

| # | Change | Object | Type |
|---|---|---|---|
| 1 | **Courtesy source view emits versions.** Adds SCD2 logic inside the view (CTEs) that builds dated versions of Management Co, Brand, Buying Group and InnCode per account from `HZ_RELATIONSHIPS`, replacing the current-only staging join. `Owner` follows the version's Management Co. `EffDate` / `ExpDate` become the version dates (DATE), not hard-coded strings. No new object. | `RAW_*.SHARED.VW_DIMCUSTACCOUNT_COURTESY` | View change (`COPY GRANTS`) |
| 2 | **Merge on the real key.** `SQLSTMT1` merges on `SourceSiteUseID AND EffDate` (matching the configured key and the Keefe merge). Existing versions are updated only when their hash changes (e.g. `ExpDate` closed); new versions are inserted with new keys. | `EDW_*.ADMINBI.ADMINMERGE` row 140 | Config (data) change |
| 3 | **One-time history rebuild.** Gives each existing row its current version's `EffDate`, so it keeps its `DimCustAccountKey`, then loads all earlier versions as new keys, with every ORG 83 account in scope. Then re-sorts and runs the Courtesy and Financials downstream merges. | `DWSHARED` / `COURTESY` / `FINANCIALS` DimCustAccount tables | One-time data load |
| 4 | **Validation.** Compare against the SQL Server output: row counts, dates, no gaps or overlaps, exactly one current row per site use. | – | Test |
| 5 | **Downstream consumer fixes** (see risk 1): current-row filter or date-aware ranking in 4 objects. | `COURTESY.FACTPRICELIST_ANYACCOUNT`, `COURTESY_SSAS.DIMCUSTACCOUNTCOURTESYORDER_SERVERS`, `COURTESY_SSAS.FACTORDERCOURTESY`, `RAW_*.CP.VW_FACTTASK_LINKED` | View / dynamic table changes (`COPY GRANTS`) |

No stored procedure changes: `USP_ADMINMERGE` is unchanged. The downstream merges (rows 31 and 56) already match on `DimCustAccountKey`, so they carry the new version rows through as-is.

### Version rules

These were inferred from the SQL Server output and verified 15/15 on account 7513:

- **New version:** starts on each relationship start date and on the day after each relationship end date.
- **Overlaps:** if a new relationship of the same type starts on or before the day the previous one ends, the new value takes effect the **day after** the previous one ends.
- **Gaps:** no active relationship of a type → `NA`.
- **First and last version:** the first version starts **2000-01-01**; the last ends **4712-12-31**.
- **No-change versions:** adjacent versions with identical values are merged into one.

## Environment preparation (done in DEV on 2026-11-17)

- **Utility database:** `UTIL_DB_DEV` created as SYSADMIN from `UtilityDbConfiguration/UTIL_DB_DEV_setup.sql`. Backups go in `UTIL_DB_DEV.BACKUPS_BI`.
- **Backups:** every DEV object changed has a backup clone or `DDL_HISTORY` row in `BACKUPS_BI`.
- **Data refresh:** DEV refreshed from PROD for ORG 83 only (`00_copy_prod_to_dev_ORG83.sql`). EDW_DEV tables now carry PROD's keys, and the DEV key sequence was moved past PROD's highest key.

## Scripts and status

| Script | Purpose | DEV | PROD |
|---|---|---|---|
| `00_copy_prod_to_dev_ORG83.sql` | One-time PROD → DEV data copy (ORG 83) | Done | n/a |
| `00_preflight_prod.sql` | Read-only go / no-go checks at the start of the window | n/a | **Ready** (dry-run 2026-11-18: all GO except UTIL_DB not yet deployed) |
| `01b_backup_dev.sql` | DEV backups | Done | n/a |
| `01_backup_prod.sql` | PROD backups into `UTIL_DB.BACKUPS_BI` (6 DDL captures + 5 dated clones) | n/a | **Ready**; all 6 `GET_DDL` captures checked against PROD |
| `02_create_history_view_dev.sql` | Superseded: logic folded into 03 (kept for reference) | Not deployed | Not deployed |
| `03_alter_courtesy_view_*.sql` | Change 1 (DEV keeps its `SET2` lookup filter; PROD version omits it) | **Done** – 5494035 returns the 15 SQL Server versions | **Ready**; body run read-only on RAW_PROD: 0 duplicates / overlaps / gaps, 5494035 = 15 SQL Server versions |
| `04_update_adminmerge_*.sql` | Change 2 (DEV also adds the missing `HASHFIELD` update) | **Done** | **Ready**; ON clause found exactly once; PROD already sets the hash, NULL-safe |
| `05_rebuild_history_*.sql` | Change 3 | **Done** – ORG 83: 115,680 → 202,799 rows; 1 current row per site use | **Ready**; stops if the guard fails; dated `_PREREBUILD` backups; re-sort compiles |
| `06_validate_*.sql` | Change 4 | **Done** – all checks pass (see below) | **Ready**; checks run read-only on today's PROD data |
| `07_fix_downstream_consumers_*.sql` | Change 5 (3 objects in DEV; 4 in PROD incl. the PROD-only `_SERVERS` view) | **Done** – grants and owner unchanged | **Ready**; each pattern matches once; current output unchanged by all 4 fixes (read-only test) |

### DEV validation results (2026-11-17)

| Check | Result |
|---|---|
| Site use 5494035 vs SQL Server | 15/15 rows identical (values, dates, Owner); current row kept PROD key `7142169` |
| Exactly one current row per site use | 0 exceptions |
| Overlapping / gapped / inverted date ranges | 0 / 0 / 0 |
| Duplicate `SourceSiteUseID + EffDate` | 0 |
| Pre-rebuild current keys still current | 0 changed |
| Other orgs (Keefe etc.) in DWSHARED | 0 rows changed |
| DWSHARED_CP view = Courtesy base table; DWSHARED ORG 83 = FINANCIALS ORG 83 | 202,800 = 202,800; 202,799 = 202,799 |
| Daily re-run of the row 140 merge with nothing changed | 0 inserted, 0 updated |

### PROD deployment runbook

All scripts run as **DEVELOPER** (owner of every changed object) except `UTIL_DB_setup.sql` (SYSADMIN). No step changes a task or schedule.

| Step | When | Script | Go / stop rule |
|---|---|---|---|
| 0 | Any time before the window | `UtilityDbConfiguration/UTIL_DB_setup.sql` (SYSADMIN) | **Done 2026-11-18.** Verified: `UTIL_DB` owned by SYSADMIN, 7 schemas, `BACKUPS_BI` (managed access, 1-day retention) with `DDL_HISTORY`, `UTIL_WH` (XS, suspended), DEVELOPER holds `BACKUPS_BI_RW` (CREATE TABLE on the schema; INSERT/SELECT on `DDL_HISTORY`) |
| 1 | Window start (~22:00 CT) | `00_preflight_prod.sql` | Every check GO / TRUE / 1, else stop |
| 2 | | `01_backup_prod.sql` | Every `src = bkp`, `DDL_ROWS = 6`, else stop |
| 3 | | `03_alter_courtesy_view_prod.sql` | `SHOW GRANTS` unchanged, owner DEVELOPER |
| 4 | | `04_update_adminmerge_prod.sql` | Verify row: all TRUE |
| 5 | | `05_rebuild_history_prod.sql` | Guard passes (else the script stops; undo step 1 of 05 as its message says) |
| 6 | | `07_fix_downstream_consumers_prod.sql` | Block returns `1 ok; 2 ok; 3 ok; 4 ok`, verify row all TRUE |
| 7 | | `06_validate_prod.sql` | All checks pass, else roll back (end of 05, then 07 / 04 / 03 per `01_backup_prod.sql`) |
| 8 | After the next `HOURLY_LOAD` (04:00 CT) | `06_validate_prod.sql` checks 2–6d again | Loads succeeded; checks still pass (check F) |

**Window:** the AdminMerge loads are started by root tasks `EDW_PROD.ADMINBI.HOURLY_LOAD` (04:00, 08:00, 12:00, 16:00 CT) and `NONHOURLY_LOAD` (21:00 CT); observed runs span 04:10–21:42 CT. Deploy between ~22:00 and 03:45 CT. Optionally the team can also suspend the two root tasks for the window (a PROD change, not included in these scripts).

Rollback steps are in `01_backup_prod.sql` and at the end of each script.

## Changed objects and backup locations

Each object that changes, and where its backup lives:

```
 PROD object (change #)                                   PROD backup (UTIL_DB.BACKUPS_BI)
 ───────────────────────────────────────────────────      ───────────────────────────────────────────────────────
 RAW_PROD.SHARED.VW_DIMCUSTACCOUNT_COURTESY      (1) ───► DDL_HISTORY  (object_fqn = same name)
 EDW_PROD.ADMINBI.ADMINMERGE  row 140            (2) ───► EDW_PROD__ADMINBI__ADMINMERGE__<date>
 EDW_PROD.DWSHARED.DIMCUSTACCOUNT                (3) ───► EDW_PROD__DWSHARED__DIMCUSTACCOUNT__<date>
 EDW_PROD.COURTESY.DIMCUSTACCOUNTBASETBL         (3) ───► EDW_PROD__COURTESY__DIMCUSTACCOUNTBASETBL__<date>
 EDW_PROD.FINANCIALS.DIMCUSTACCOUNT              (3) ───► EDW_PROD__FINANCIALS__DIMCUSTACCOUNT__<date>
 RAW_PROD.ORACLE.STGCHGDETECTDIMCUSTACCOUNT      (3) ───► RAW_PROD__ORACLE__STGCHGDETECTDIMCUSTACCOUNT__<date>  (temp scope add/remove)
 EDW_PROD.COURTESY.FACTPRICELIST_ANYACCOUNT      (5) ───► DDL_HISTORY
 EDW_PROD.COURTESY_SSAS.FACTORDERCOURTESY (DT)   (5) ───► DDL_HISTORY
 EDW_PROD.COURTESY_SSAS.DIMCUSTACCOUNTCOURTESYORDER_SERVERS (5) ► DDL_HISTORY
 RAW_PROD.CP.VW_FACTTASK_LINKED                  (5) ───► DDL_HISTORY
 EDW_PROD.ADMINBI.USP_ADMINMERGE                 (not changed) ► DDL_HISTORY (reference copy)
```

| # | Object | Kind | DEV backup (`UTIL_DB_DEV.BACKUPS_BI`) | PROD backup (`UTIL_DB.BACKUPS_BI`) |
|---|---|---|---|---|
| 1 | `RAW_*.SHARED.VW_DIMCUSTACCOUNT_COURTESY` | View definition | `DDL_HISTORY` | `DDL_HISTORY` |
| 2 | `EDW_*.ADMINBI.ADMINMERGE` (row 140) | Config data | `EDW_DEV__ADMINBI__ADMINMERGE__20261117` | `EDW_PROD__ADMINBI__ADMINMERGE__<date>` |
| 3 | `EDW_*.DWSHARED.DIMCUSTACCOUNT` | Table data | `EDW_DEV__DWSHARED__DIMCUSTACCOUNT__20261117` (original DEV) · `..._20261117_PREREBUILD` (after PROD copy, before rebuild) | `EDW_PROD__DWSHARED__DIMCUSTACCOUNT__<date>` (01) · `..._<date>_PREREBUILD` (05) |
| 3 | `EDW_*.COURTESY.DIMCUSTACCOUNTBASETBL` | Table data | `EDW_DEV__COURTESY__DIMCUSTACCOUNTBASETBL__20261117` · `..._PREREBUILD` | `EDW_PROD__COURTESY__DIMCUSTACCOUNTBASETBL__<date>` (01) · `..._PREREBUILD` (05) |
| 3 | `EDW_*.FINANCIALS.DIMCUSTACCOUNT` | Table data | `EDW_DEV__FINANCIALS__DIMCUSTACCOUNT__20261117` · `..._PREREBUILD` | `EDW_PROD__FINANCIALS__DIMCUSTACCOUNT__<date>` (01) · `..._PREREBUILD` (05) |
| 3 | `RAW_*.ORACLE.STGCHGDETECTDIMCUSTACCOUNT` | Staging data (rows added, then removed) | `RAW_DEV__ORACLE__STGCHGDETECTDIMCUSTACCOUNT__20261117` · `..._PREREBUILD` | `RAW_PROD__ORACLE__STGCHGDETECTDIMCUSTACCOUNT__<date>` (01) · `..._PREREBUILD` (05) |
| 5 | `EDW_*.COURTESY.FACTPRICELIST_ANYACCOUNT` | View definition | `DDL_HISTORY` | `DDL_HISTORY` (01) |
| 5 | `EDW_*.COURTESY_SSAS.FACTORDERCOURTESY` | Dynamic table definition | `DDL_HISTORY` | `DDL_HISTORY` (01) |
| 5 | `EDW_PROD.COURTESY_SSAS.DIMCUSTACCOUNTCOURTESYORDER_SERVERS` | View definition (PROD only) | n/a | `DDL_HISTORY` (01) |
| 5 | `RAW_*.CP.VW_FACTTASK_LINKED` | View definition | `DDL_HISTORY` | `DDL_HISTORY` (01) |
| – | `RAW_DEV` source tables (20) and `EDW_DEV` targets touched by the one-time PROD → DEV copy | DEV data only | `RAW_DEV__<SCHEMA>__<TABLE>__20261117` (Time Travel clones as of 16:57:00, pre-copy) | n/a |
| – | `EDW_*.ADMINBI.USP_ADMINMERGE` | Procedure (not changed) | `DDL_HISTORY` (reference) | `DDL_HISTORY` (reference) |

Restoring: views and dynamic tables are restored by running their saved `ddl_text` with `COPY GRANTS`; table data with `INSERT OVERWRITE INTO <table> SELECT * FROM <clone>`, which keeps the live table's grants and sequence defaults.

## Risks / questions for the team

1. **Consumers expecting one row per site use.** Reports, cubes and joins on `COURTESY.DIMCUSTACCOUNTBASETBL` or `FINANCIALS.DIMCUSTACCOUNT` that assume one row per `SourceSiteUseID` will start seeing multiple rows. Each needs a current-row filter (`ExpDate = '4712-12-31'`) or must join on `DimCustAccountKey`.
   - *Dependency check 2026-11-17 (PROD object dependencies) on `COURTESY.DIMCUSTACCOUNTBASETBL` (14 objects, via view `COURTESY.DIMCUSTACCOUNT`):*

     | Consumer | Join / usage | Impact | Action |
     |---|---|---|---|
     | `RAW_PROD.CP` / `PUBLIC` `VW_FACTORDERHEADER`, `VW_FACTFREIGHTCHARGES`, `VW_FACTFREIGHTWEIGHTS`, `CP.VW_COURTESYFACTORDER_CHGS` | Site use **and** order/creation date `BETWEEN EffDate AND ExpDate` | Correct; designed for SCD2. Today, orders dated before a site use's single `EffDate` find no dimension row | None |
     | `COURTESY.DIMCUSTACCOUNT`, `COURTESY.VW_DIMCUSTACCOUNT`, `COURTESY_SSAS.DIMCUSTACCOUNTCOURTESYORDER` (DT) | Pass-through of the dimension | More rows (expected); consumers join on `DimCustAccountKey` | None |
     | `COURTESY.FACTPRICELIST_ANYACCOUNT` | `dca2` joined on `SourceCustAccoutID`, no date | **Duplicate rows** (× versions) | Add `dca2.ExpDate = '4712-12-31'` |
     | `COURTESY_SSAS.DIMCUSTACCOUNTCOURTESYORDER_SERVERS` | `SUM(workdayscount)`, `SUM(fullworkterm)` by account | **Inflated totals** (× versions) | Aggregate current rows only |
     | `COURTESY_SSAS.FACTORDERCOURTESY` (DT), placeholder branch | Base table joined on `STRNumber`, no date | Duplicate zero-value placeholder rows | Add `dcabt.ExpDate = '4712-12-31'` |
     | `RAW_PROD.CP.VW_FACTTASK_LINKED` | `CustSrt = 1` ranked across all rows, then date range | Tasks before the current version's `EffDate` lose their customer match | Rank within the date-matched rows |

   - *`EDW_PROD.FINANCIALS.DIMCUSTACCOUNT` (2 views): **not affected**:*
     - `EDW_PROD.FINANCIALS_QLIK.CUSTOMERS`: uses `SELECT DISTINCT OracleAccount, BusinessLine`. BusinessLine is the same on every version, so the extra versions collapse to the same rows.
     - `EDW_PROD.FINANCIALS_SSAS.ACGLINTERFACE_CUSTOMER`: filters `SingleAccountSort = 1` (one row per account: lowest site use, latest `EffDate`, i.e. the current version) and only keys present in `FactAccessCorrectionsGL`. Still one row per account, and all its columns are the same on every version.
   - *Still to check: references from outside Snowflake's dependency tracking (Qlik/SSAS models, ad-hoc SQL, stored procedures with dynamic SQL).* See remaining checks below.
2. **How fact loads pick `DimCustAccountKey`: verified, favorable.** The Courtesy fact views already join on site use and order date between `EffDate` and `ExpDate`. Once history exists, they will assign the version that was in effect on the order date, without any code change. *Still to confirm: how the fact tables reload (see remaining checks below).*
3. **Which key existing facts keep.** The rebuild keeps each existing key on the **current** version (verified in DEV: 0 current keys changed), so facts already loaded don't change until they're reloaded. Because the fact views are date-aware, a fact reload would then attach historical orders to historical versions, matching SQL Server.
4. **Rules validated on one account only.** They should be checked against a larger SQL Server extract (ideally the full legacy table for ORG 83) before PROD.
5. **Change detection.** The view only processes parties listed in `STGCHGDETECTDIMCUSTACCOUNT`. Confirm the upstream change detection includes parties whose `HZ_RELATIONSHIPS` rows changed. Otherwise new versions won't be picked up daily.
6. **Back-dated edits in Oracle.** If a relationship's dates are edited after the fact, earlier versions can be left behind with stale dates. Today's logic doesn't delete versions (deleting could orphan fact keys). A cleanup rule is to be decided.
7. **Which columns are versioned.** Only Management Co, Brand, Buying Group, Owner and InnCode are versioned. This matches the SQL Server sample, but the business should confirm it.
8. **DEV vs PROD differences.** The DEV Oracle lookup (`FND_LOOKUP_VALUES`) has two editions, so the DEV view keeps `zd_edition_name = 'SET2'`. The PROD scripts must not include that line. DEV's row 140 was also missing the `HASHFIELD` update that PROD has (fixed in DEV by script 04).
9. **Site uses not produced by the view.** 2 existing ORG 83 rows in DWSHARED have no match in the view (likely site uses that no longer meet the view's joins in Oracle). The rebuild leaves them untouched as current rows, matching today's behaviour.
10. **AdminMerge write-back.** `USP_ADMINMERGE` writes a regenerated `SQLSTMT1` back only when the column is NULL, so the row 140 change is not overwritten by daily runs. Rows 140 and 141 share `PACKAGEFILTER` + `TARGETTABLE`, and the write-back matches on those two columns: **never blank `SQLSTMT1` on either row**, or the regenerated statement would be written to both.
11. **FINANCIALS lags DWSHARED between loads.** On 2026-11-18, 4 ORG 83 rows added by the 08:15 load were not yet in FINANCIALS (row 56 runs later). Count comparisons in 06 are taken right after 05, which runs row 56 itself.
12. **DEV and PROD `USP_ADMINMERGE` / `ADMINMERGE` have drifted** (DEV procedure last changed 2026-01-05, PROD 2026-06-10; 125 of 127 rows' stored statements differ, mostly in hash handling). Doesn't affect this change, but DEV is not a perfect rehearsal of PROD for other tables.

### Remaining checks / re-checks

| # | Check | Owner | Why it matters |
|---|---|---|---|
| A | **Fact reload pattern.** Do the Courtesy fact tables (`FactOrder`, `FactFreight*`, `FactTask*`) reload from the `VW_FACT*` views in full, or only a rolling window / changed rows? | ELT team | Decides whether historical orders pick up their historical versions on their own, or need a one-time fact reload after the rebuild |
| B | **References outside Snowflake's dependency tracking.** Qlik apps on `FINANCIALS_QLIK`, SSAS models on `*_SSAS`, ad-hoc reports, procedures with dynamic SQL. Do they filter to current rows or join on `DimCustAccountKey`? | BI / model owners | Anything that joins the dimension on site use or account alone will now see multiple rows |
| C | **SQL Server comparison at scale.** Legacy `CourtesyOracleDataMart.dbo.DimCustAccountBaseTbl` exported 2026-11-18 (203,868 rows) and diffed against the DEV rebuild (`08_compare_legacy_dev.sql`). | Team + this project | **Done 2026-11-18** – see *Legacy comparison results* below |
| D | **Change detection covers relationship changes** (risk 5). | ELT team | Daily runs must pick up new Management Co / Brand / Buying Group versions |
| E | **`VW_FACTTASK_LINKED` result review.** The fix ranks within each version period; compare task counts with a matched customer before/after in DEV. | This project | Confirms no task gains or loses a customer unexpectedly |
| F | **Re-check after PROD deploy.** Re-run `06_validate` against PROD, plus row counts of the 4 fixed consumers before/after. | This project | Confirms PROD matches DEV |

### Legacy comparison results (check C, 2026-11-18)

Legacy SQL Server export vs DEV rebuild (PROD data as of 2026-11-17):

| Measure | Result |
|---|---|
| Site uses with **identical version dates** | **114,703 of 115,693 (99.14%)**; 197,531 rows identical in both |
| Site uses with a different number of versions | 968 (legacy 6,301 rows vs DEV 5,246) |
| Same number of versions, different dates | 9 |
| Site uses only in legacy / only in DEV | 13 / 0 |
| Versioned values on the 202,788 rows that match on site use + `EffDate` | ManagementCo 18, Brand 65, BuyingGroup 31, **Owner 0** differences (≤ 0.03%) |
| BusinessLine / AccountName / ZIP / Status on matched rows | 0 / 0 / 0 / 1 differences |

**Why the 968 site uses have more versions in legacy:** of the 1,066 version starts only legacy has,
- **922** begin the day after an Oracle relationship ended, with **no value change**: SQL Server keeps a separate version at that boundary, while the new logic merges adjacent identical versions;
- **108** are at a relationship boundary with a value change that the new logic resolves differently (overlapping relationships of the same type);
- **36** have no matching boundary in current Oracle data (most likely relationship dates edited in Oracle after SQL Server captured them).

The legacy history looks like it was built from daily snapshots of Oracle, so it keeps boundaries that are no longer visible, or no longer meaningful, in today's relationship data. The new logic rebuilds history from the current Oracle dates. The 922 extra legacy versions carry the same Management Co, Brand, Buying Group and Owner as the version before, so reports by those attributes give the same results either way; only the version count differs.

**Differences that are not caused by this change:** InnCode, EMailAddresses, SalesRep, SalesRegion and CurrentManagementCo differ between legacy and Snowflake on current rows by the same amounts **today, before the change** (legacy vs PROD: 8,407 / 53,812 / 2,841 / 530 / 83; legacy vs DEV rebuild: 8,389 / 53,823 / 2,840 / 529 / 82). These come from the existing (unchanged) staging logic and are outside this project's scope.

**Decision for the team:** accept the merged-version behaviour (recommended: fewer, meaningful versions; same values on every date). Matching SQL Server's version count exactly is not a simple rule change: in a read-only simulation, removing the merge step made only 102 of the 968 site uses match, and splitting at every raw relationship date overshot (860 site uses with more versions than legacy). The legacy splits appear to depend on when SQL Server observed changes, which can't be rebuilt from current Oracle data.

## Open items

- **Reusable PROD → DEV copy procedure** to replace the one-time `00_copy_prod_to_dev_ORG83.sql`.
- **Remaining checks A–F** (above).
- **Schedule the deployment window** (target Friday 2026-11-20, ~22:00–03:45 CT).


## Effective Data Changes (result of work)

They change in data only:
  These are cloned because the data changes. The only structural changes in the deployment are the five object definitions recreated with COPY GRANTS: the Courtesy source view, three consumer views, and the FACTORDERCOURTESY dynamic table.