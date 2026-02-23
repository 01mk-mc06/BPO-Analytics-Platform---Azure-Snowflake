# Snowflake + Azure BPO Analytics Platform
## Troubleshooting Notes, Pain Points & Future Improvements

---

## Project Overview

End-to-end BPO analytics pipeline built over 7 days covering:
- Azure Blob + Snowpipe auto-ingestion
- Bronze → Silver → Gold medallion architecture
- Self-healing Bronze layer with quarantine (architecture in place, data flow currently being debugged)
- Streams & Tasks for incremental processing
- Dynamic Tables for Gold KPI layer
- Snowpark Python for ML scoring and anomaly detection
- Role-based access control with masking and row access policies
- Power BI Desktop dashboard connected to Snowflake Gold layer (planned migration to Looker Studio)

**Current Status:** Pipeline architecture complete. Actively troubleshooting data flow between Bronze → Silver → Gold layers. Quarantine table and retry mechanism are designed and in place but not yet fully operational — flagged as a future feature addition.

---

## Pain Points & Walls We Hit

### 1. Azure Stage URL — Took Multiple Attempts

**What happened:**
Getting the correct Azure stage URL took several iterations. The storage account name, container name, and resource group were confused with each other. The initial URL used the resource group name as the container which caused `ContainerNotFound` errors repeatedly.

**What we learned:**
The Azure Blob URL format is strict:
```
azure://<storage_account_name>.blob.core.windows.net/<container_name>/
```
- Storage account = `<your_storage_account>`
- Container = `raw` (found by clicking the container in Azure Portal and reading the URL)
- Resource group = `bpo-snowflake` — this is NOT part of the URL

**Time lost:** ~30 minutes of back and forth recreating the stage and integration.

---

### 2. Snowpipe + Event Grid Setup — Most Complex Part

**What happened:**
The Snowpipe AUTO_INGEST setup required multiple components working together in the right order. Missing any one step silently broke the pipeline without clear error messages.

**Components required in order:**
1. Storage Integration (Snowflake reads Azure Blob)
2. Consent URL accepted in browser
3. Azure IAM role assignments (Storage Blob Data Contributor)
4. Storage Queue created in Azure
5. Notification Integration (Snowpipe reads queue events)
6. Second consent URL accepted
7. Second IAM role assignments (Storage Queue Data Contributor + Message Processor)
8. Event Grid System Topic created from Storage Account Events tab
9. Event Subscription (BlobCreated → Storage Queue)
10. Snowpipe created with `AUTO_INGEST = TRUE`
11. Target tables must exist before pipe runs
12. `ALTER PIPE REFRESH` for existing files

**Biggest wall:** The Event Grid subscription endpoint type — had to select `Storage Queue` from the dropdown which wasn't immediately obvious. Also the pipe was pointing to a specific file path instead of a folder, so new uploads didn't auto-trigger.

**Time lost:** This was the single most time-consuming part of the entire project.

---

### 3. Stream Timing — Created After Data Was Loaded

**What happened:**
Streams were created after Snowpipe had already loaded data into Bronze tables. Since streams only track changes from the point of creation, the initial load was invisible to the stream. `SYSTEM$STREAM_HAS_DATA()` returned `FALSE` even though Bronze tables had data.

**Cascade effect:**
- Had to truncate Bronze tables
- Truncate generated DELETE events in the stream
- Had to consume the delete events with a dummy `BEGIN...SELECT...COMMIT`
- Then reload files from Azure to generate fresh INSERT events
- This cycle took multiple rounds to get right

**Time lost:** ~45 minutes across all 4 streams.

---

### 4. Quarantine Not Capturing Bad Records

**What happened:**
The task uses `BEGIN...END` with two INSERT statements — one for valid records into Silver, one for bad records into quarantine. Both read from the same stream. Once the first INSERT commits, the stream offset moves forward and the second INSERT sees an empty stream.

**Result:** Valid records landed in Silver correctly but bad records were silently ignored — not quarantined, not flagged, just dropped.

**Status:** Known issue, marked as future improvement. Fix requires staging stream data into a temp table first before splitting into valid/invalid paths.

---

### 5. Row Access Policy — Case Sensitivity Trap

**What happened:**
The `role_access_map` table was populated with team values in title case (`Alpha`, `Beta`, `Gamma`) but the actual column values in `silver.agents` were uppercase (`ALPHA`, `BETA`, `GAMMA`) because the task used `UPPER(TRIM(team))`. The row access policy does an exact string match so every row was filtered out for all roles.

**Cascade effect:**
- All custom roles saw zero data
- Thought it was a grants issue — spent time adding/verifying grants
- Turns out grants were fine, the problem was purely the case mismatch

**Time lost:** ~30 minutes diagnosing before isolating the case issue.

---

### 6. Snowpark Ambiguous Column Names After Join

**What happened:**
When joining `df_perf` and `df_agents` on `AGENT_ID`, both DataFrames had an `AGENT_ID` column. Snowpark auto-generated internal names like `l_0006_AGENT_ID` instead of the expected suffix. Multiple approaches were tried:
- `lsuffix/rsuffix` → generated `AGENT_IDPERF` (no underscore)
- Referencing `col("AGENT_ID_PERF")` → `invalid identifier` error
- Had to use quoted identifiers `col('"l_0006_AGENT_ID"')` to reference the auto-generated name

**Root cause:** Snowpark's column disambiguation is not as intuitive as PySpark. The safest pattern is to rename the duplicate column in one DataFrame **before** joining.

**Time lost:** ~20 minutes across multiple error iterations.

---

### 7. One Row Access Policy Per Table Limitation

**What happened:**
Tried to apply both `team_access_policy` (team-level filtering) and `agent_self_policy` (agent-level filtering) to `gold.agent_scorecard`. Snowflake only allows one row access policy per table, so the second apply failed.

**Decision:** Kept `team_access_policy` and documented `agent_self_policy` as a future improvement. The proper fix is to combine both into a single multi-column policy.

---

### 8. Power BI Authentication + Desktop Limitations

**What happened:**
Power BI Desktop threw a PEM private key error on the first connection attempt because it defaulted to key-pair authentication instead of username/password. Had to clear cached permissions and explicitly select `Database` authentication mode.

**Power BI Desktop limitations discovered:**
- Requires manual refresh — no live connection without Power BI Gateway or Pro license
- DirectQuery mode with Snowflake Dynamic Tables requires additional setup
- Publishing to Power BI Service requires a Pro license to share dashboards
- Not ideal for demonstrating real-time pipeline capabilities

**Planned migration:**
The dashboard will transition to **Looker Studio** which offers:
- Free native Snowflake connector
- No license required for sharing
- Live data connection without gateway setup
- Better suited for demonstrating Dynamic Table auto-refresh capabilities

---

## Troubleshooting Issues & Fixes

---

### Issue 1: Row Access Policy Blocking All Roles

**Symptom:**
Data visible to `ACCOUNTADMIN` but not to `bpo_admin`, `bpo_analyst`, `bpo_operations`, or `bpo_agent`.

**Root Cause:**
Case mismatch between `role_access_map` values and actual column values in `silver.agents`.

**How to Diagnose:**
```sql
select distinct team, length(team) from bpo_db.silver.agents;
select * from bpo_db.silver.role_access_map;

alter table bpo_db.silver.agents
  drop row access policy bpo_db.silver.team_access_policy;

select team, count(*) from bpo_db.silver.agents group by team;
```

**Fix:**
```sql
delete from bpo_db.silver.role_access_map;

insert into bpo_db.silver.role_access_map values
  ('ACCOUNTADMIN',   'ALPHA'),
  ('ACCOUNTADMIN',   'BETA'),
  ('ACCOUNTADMIN',   'GAMMA'),
  ('BPO_ADMIN',      'ALPHA'),
  ('BPO_ADMIN',      'BETA'),
  ('BPO_ADMIN',      'GAMMA'),
  ('BPO_ANALYST',    'ALPHA'),
  ('BPO_ANALYST',    'BETA'),
  ('BPO_OPERATIONS', 'ALPHA'),
  ('BPO_OPERATIONS', 'BETA'),
  ('BPO_OPERATIONS', 'GAMMA');

alter table bpo_db.silver.agents
  add row access policy bpo_db.silver.team_access_policy on (team);
```

**Key Lesson:** `CURRENT_ROLE()` always returns uppercase. Always verify exact column values with `select distinct` before populating access map tables.

---

### Issue 2: Row Access Policy Cannot Be Dropped

**Symptom:**
```
Policy TEAM_ACCESS_POLICY cannot be dropped/replaced as it is associated with one or more entities.
```

**Fix:**
```sql
alter table bpo_db.silver.agents
  drop row access policy bpo_db.silver.team_access_policy;

alter table bpo_db.gold.agent_scorecard
  drop row access policy bpo_db.silver.team_access_policy;

alter table bpo_db.gold.team_performance
  drop row access policy bpo_db.silver.team_access_policy;

drop row access policy bpo_db.silver.team_access_policy;
```

**Key Lesson:** Always remove a policy from all associated tables before dropping it.

---

### Issue 3: Duplicate Row Access Policy in Wrong Schema

**Symptom:**
Two `TEAM_ACCESS_POLICY` entries — one in `PUBLIC`, one in `SILVER`.

**How to Check:**
```sql
show row access policies in database bpo_db;
```

**Fix:**
```sql
drop row access policy bpo_db.public.team_access_policy;
```

**Key Lesson:** Always `use schema` explicitly before creating policies.

---

### Issue 4: Roles See No Data — Missing Grants

**Symptom:**
Custom roles return no data even after row access policy is correctly configured.

**Fix:**
```sql
use role accountadmin;

grant usage on warehouse bpo_transform_wh to role bpo_admin;
grant usage on warehouse bpo_transform_wh to role bpo_analyst;
grant usage on warehouse bpo_transform_wh to role bpo_operations;
grant usage on warehouse bpo_transform_wh to role bpo_agent;

grant usage on database bpo_db to role bpo_admin;
grant usage on database bpo_db to role bpo_analyst;
grant usage on database bpo_db to role bpo_operations;
grant usage on database bpo_db to role bpo_agent;

grant usage on schema bpo_db.silver to role bpo_admin;
grant usage on schema bpo_db.silver to role bpo_analyst;
grant usage on schema bpo_db.gold   to role bpo_analyst;
grant usage on schema bpo_db.gold   to role bpo_operations;
grant usage on schema bpo_db.gold   to role bpo_agent;

grant select on all tables in schema bpo_db.silver to role bpo_admin;
grant select on all tables in schema bpo_db.silver to role bpo_analyst;
grant select on all tables in schema bpo_db.gold   to role bpo_analyst;
grant select on all tables in schema bpo_db.gold   to role bpo_operations;
grant select on table bpo_db.gold.agent_scorecard  to role bpo_agent;

grant select on all dynamic tables in schema bpo_db.gold to role bpo_analyst;
grant select on all dynamic tables in schema bpo_db.gold to role bpo_operations;
grant select on all dynamic tables in schema bpo_db.gold to role bpo_agent;
```

**Key Lesson:** Privileges in Snowflake are not inherited automatically. Grant at every level: warehouse → database → schema → table. Dynamic tables need separate grants.

---

### Issue 5: Stream Has No Data After Initial Load

**Symptom:**
`SYSTEM$STREAM_HAS_DATA()` returns `FALSE` even though Bronze table has data.

**Root Cause:**
Stream was created after data was already loaded.

**Fix:**
```sql
drop stream bpo_db.bronze.stream_agents;
create or replace stream bpo_db.bronze.stream_agents on table bpo_db.bronze.raw_agents;

truncate table bpo_db.bronze.raw_agents;

insert into bpo_db.bronze.raw_agents
select $1, $2, $3, $4, $5, $6, $7, $8, metadata$filename, current_timestamp()
from @bpo_db.bronze.bpo_azure_stage/raw/agents.csv
file_format = bpo_db.bronze.csv_format;

select system$stream_has_data('bpo_db.bronze.stream_agents');
```

**Key Lesson:** Always create streams before loading data.

---

### Issue 6: Stream Shows DELETE Events After TRUNCATE

**Symptom:**
Stream has data but all records show `METADATA$ACTION = 'DELETE'`.

**Fix:**
```sql
begin;
  select * from bpo_db.bronze.stream_agents;
commit;

select system$stream_has_data('bpo_db.bronze.stream_agents');

insert into bpo_db.bronze.raw_agents ...
```

**Key Lesson:** `TRUNCATE` generates stream events. Consume the stream after truncating before reloading.

---

### Issue 7: Quarantine Not Capturing Bad Records

**Symptom:**
Bad records not appearing in `quarantine_records` even though task runs successfully.

**Root Cause:**
Both INSERT statements in `BEGIN...END` read from the same stream. First INSERT consumes the stream, second INSERT sees nothing.

**Workaround (Pending Fix):**
```sql
create or replace temporary table temp_tickets as
select * from bpo_db.bronze.stream_tickets
where metadata$action = 'INSERT';

insert into bpo_db.silver.tickets
select ... from temp_tickets where <valid conditions>;

insert into bpo_db.bronze.quarantine_records
select ... from temp_tickets where <invalid conditions>;
```

**Key Lesson:** Streams can only be consumed once per transaction. Snapshot to a temp table when multiple statements need to read from the same stream.

---

### Issue 8: Snowpipe Not Loading — Wrong Stage URL

**Symptom:**
`list @bpo_azure_stage` returns no files or `ContainerNotFound` error.

**How to Find Correct URL:**
Azure Portal → Storage Account → Containers → click container → copy browser URL.

**Fix:**
```sql
create or replace stage bpo_db.bronze.bpo_azure_stage
  url = 'azure://<your_storage_account>.blob.core.windows.net/<your_container>/'
  storage_integration = bpo_azure_integration
  file_format = bpo_db.bronze.csv_format;

list @bpo_azure_stage;
```

**Key Lesson:** Stage URL = storage account + container only. Subfolders go in `COPY INTO` or `LIST`, not the stage definition.

---

### Issue 9: Snowpark Ambiguous Column After Join

**Symptom:**
```
SnowparkSQLAmbiguousJoinException: The reference to the column 'AGENT_ID' is ambiguous.
```

**Fix — rename before joining:**
```python
df_agents_renamed = df_agents.rename(df_agents["AGENT_ID"], "AGENT_ID_B")

df_joined = df_perf.join(
    df_agents_renamed,
    df_perf["AGENT_ID"] == df_agents_renamed["AGENT_ID_B"]
)
```

**Key Lesson:** Always rename duplicate columns in one DataFrame before joining in Snowpark. Don't rely on `lsuffix/rsuffix` — it generates unpredictable internal names.

---

### Issue 10: Snowpark Column Not Found After Transform

**Symptom:**
```
invalid identifier 'DATE' — available columns: 'DATEABS'
```

**Root Cause:**
Column names in Silver were transformed by the task (e.g., `date` → `DATEABS`, `aht_mins` → `AHT_MINSABS`).

**Fix:**
Always check exact column names before writing Snowpark code:
```sql
desc table bpo_db.silver.performance;
```

---

### Issue 11: Power BI PEM Key Error

**Symptom:**
```
Failed to parse PEM block containing the private key
```

**Fix:**
1. Power BI → File → Options → Data Source Settings
2. Clear Permissions on the Snowflake connection
3. Reconnect → select **Database** authentication
4. Enter username and password manually

---

## Quick Reference: Troubleshooting Checklist

| Symptom | First Check | Fix |
|--|--|--|
| Role sees no data | `select current_role()` | Add role to access map with correct case |
| All roles see no data | `select distinct team from table` | Fix case mismatch in access map |
| Policy won't drop | `show row access policies` | Remove from all tables first |
| Duplicate policies | `show row access policies in database` | Drop the one in wrong schema |
| No grants error | `show grants to role <role>` | Grant warehouse → database → schema → table |
| Dynamic tables not visible | Check grants | Add separate dynamic table grants |
| Stream empty after load | Check stream creation time | Recreate stream, truncate, reload |
| Stream has DELETE events | Check after truncate | Consume stream with dummy read |
| Quarantine empty | Check task history | Use temp table to snapshot stream |
| Stage URL error | Check Azure portal URL | Use storage account + container only |
| Snowpark ambiguous column | Print df.columns after join | Rename duplicate column before joining |
| Power BI auth error | Check auth method selected | Clear permissions, select Database auth |

---

## Known Issues & Future Improvements

---

### 1. Quarantine & Self-Healing — Planned Feature Addition

**Current status:** Quarantine table (`bronze.quarantine_records`) and retry task (`task_retry_quarantine`) are designed and created but not yet fully operational. The pipeline is currently in active data flow troubleshooting — bad records are staying in Bronze raw tables but are not being automatically routed to quarantine.

**Root cause of quarantine not working:**
Both INSERT statements in `BEGIN...END` read from the same stream. First INSERT (valid records → Silver) consumes the stream offset, leaving the second INSERT (bad records → quarantine) with an empty stream.

**Planned fix (future feature):**
```sql
-- snapshot stream into temp table first
create or replace temporary table temp_<table> as
select * from stream_<table> where metadata$action = 'INSERT';

-- step a: valid records from temp table
insert into bpo_db.silver.<table>
select ... from temp_<table> where <valid conditions>;

-- step b: quarantine from temp table
insert into bpo_db.bronze.quarantine_records
select ... from temp_<table> where <invalid conditions>;
```

**Also planned:** Retry task that reprocesses `PENDING` quarantine records every 30 minutes and marks as `FAILED` after 3 retries.

**Priority:** High — core self-healing feature, will be added as next iteration after data flow is stable.

---

### 2. One Row Access Policy Per Table

**Current behavior:** `agent_scorecard` uses `team_access_policy` only. `agent_self_policy` could not be applied simultaneously.

**Proper fix:** Combine both into a single multi-column policy:
```sql
create or replace row access policy combined_scorecard_policy
  as (agent_id varchar, team varchar) returns boolean ->
    current_role() = 'ACCOUNTADMIN'
    or (current_role() in ('BPO_ADMIN', 'BPO_ANALYST', 'BPO_OPERATIONS')
      and exists (
        select 1 from bpo_db.silver.role_access_map
        where role_name = current_role()
          and team = role_access_map.team
      ))
    or exists (
      select 1 from bpo_db.silver.agent_user_map
      where snowflake_user = current_user()
        and agent_id = agent_user_map.agent_id
    );
```

**Priority:** Medium — agent role currently sees all teams instead of own records only.

---

### 3. Sample Data Too Small for Anomaly Detection

**Current behavior:** Anomaly detection threshold had to be loosened from `-10` to `-5` because sample data only has 1-2 records per agent.

**Proper fix:** Generate at least 90 days of daily performance records per agent (minimum 900 rows) to produce realistic anomaly patterns.

**Priority:** Low — cosmetic, doesn't affect pipeline logic.

---

### 4. Silver Column Naming Inconsistency

**Current behavior:** Some Silver columns have unexpected suffixes (`DATEABS`, `AHT_MINSABS`) causing Snowpark failures.

**Proper fix:** Audit all Silver table column names after task creation and standardize naming conventions before building downstream code.

**Priority:** Medium — causes confusion in Snowpark and downstream queries.

---

### 5. Query Optimization — Not Implemented

**Current status:** Query optimization (clustering keys, search optimization, query profiling) was not implemented in this project.

**Reasons:**
- Sample dataset is too small (15–50 rows per table) to produce meaningful optimization results — clustering and search optimization require large tables (100k+ rows) to show measurable improvement
- Snowflake free trial account was used — trial credits are limited and running large optimization benchmarks would consume credits unnecessarily
- Optimization on small data can actually be slower due to overhead of maintaining clustering metadata

**Planned implementation:**
When the project is scaled with a larger dataset or migrated to a paid account:
- Add clustering keys on `date`, `site`, `team` columns in Silver and Gold tables
- Enable search optimization on agent lookup queries
- Run Query Profile before and after to benchmark partition pruning
- Right-size warehouses per workload type (ingestion vs transformation vs reporting)

**Priority:** Medium — relevant only when dataset scales beyond trial size.

---

### 6. No dbt Layer

**Current behavior:** Transformations written directly in Snowflake Tasks and Dynamic Tables — no version control, testing, or documentation framework.

**Proper fix:** Add dbt on top of Silver → Gold layer:
- Models replace Dynamic Tables
- Tests validate data quality automatically
- Documentation auto-generated from schema.yml
- Git version control for all transformations

**Priority:** High for production — most in-demand Analytics Engineering pattern.

---

### 7. No CI/CD Pipeline

**Current behavior:** All SQL and Python scripts run manually.

**Proper fix:**
- Store all SQL in GitHub
- Use GitHub Actions to deploy schema changes
- Add dbt test runs on every PR
- Automate Snowpark script execution via Azure Functions or Airflow

**Priority:** Medium for portfolio, High for production.

---

### 8. Dashboard Migration — Power BI to Looker Studio

**Current behavior:** Power BI Desktop connected to Snowflake Gold layer via reporting views. Works for local development but has limitations for sharing and live data demonstration.

**Power BI Desktop limitations:**
- Manual refresh only — does not reflect Dynamic Table updates automatically
- Sharing requires Power BI Pro license
- PEM key authentication errors on first setup

**Planned migration to Looker Studio:**
- Free native Snowflake connector
- No license required for public sharing
- Live connection reflects Dynamic Table refreshes in real time
- Better for portfolio demonstration purposes

**Priority:** Medium — current Power BI setup works for demo, Looker Studio migration planned as next step.

---

## Architecture Summary

```
azure blob storage (raw csv/json files)
        ↓ event grid → storage queue
        ↓ snowpipe auto_ingest
bronze layer
  ├── raw_agents
  ├── raw_tickets
  ├── raw_performance
  ├── raw_queue_logs
  └── quarantine_records  ← designed, data flow being debugged
        ↓ streams + tasks (every 5 mins)
silver layer
  ├── agents
  ├── tickets
  ├── performance
  ├── queue_events
  ├── role_access_map     ← governance
  └── agent_user_map      ← governance
        ↓ dynamic tables (every 2 mins)
gold layer
  ├── agent_scorecard     ← row access + masking policies
  ├── team_performance    ← row access policy
  ├── ticket_summary
  ├── daily_ops_report
  ├── queue_health
  ├── agent_ml_scores     ← snowpark output
  └── tenure_cohort_analysis ← snowpark output
        ↓
audit layer
  └── performance_anomalies ← snowpark anomaly detection
        ↓
power bi desktop (current) → looker studio (planned migration)
```

---


