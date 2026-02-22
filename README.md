# BPO Analytics Platform
## [*SQL and Python code to follow*]
### End-to-End Data Pipeline | Azure + Snowflake | Medallion Architecture

---

## Overview

A production-grade data engineering project simulating a real-world BPO (Business Process Outsourcing) analytics platform for content moderation operations. Built on Azure and Snowflake, the pipeline ingests raw operational data, validates and transforms it through a medallion architecture, and surfaces KPIs through a governance-controlled reporting layer.

This project was designed to mirror real enterprise patterns used in large-scale BPO environments — including automated ingestion, incremental processing, self-healing data quality, role-based access control, and operational dashboards.

---

## Architecture

```
Azure Blob Storage (raw files)
        │
        ▼
┌─────────────────────────────────────────┐
│           BRONZE LAYER                  │
│  Raw ingestion via Snowpipe + Event Grid│
│  Self-healing quarantine system         │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│           SILVER LAYER                  │
│  Incremental processing via Streams     │
│  Validation, standardization, flagging  │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│           GOLD LAYER                    │
│  Auto-refreshing KPIs via Dynamic Tables│
│  Agent scorecards, team performance,    │
│  ticket analytics, queue health         │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│        GOVERNANCE LAYER                 │
│  Role-based access, column masking,     │
│  row-level security                     │
└─────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────┐
│           ANALYTICS LAYER               │
│  Snowpark Python scoring & anomaly      │
│  detection + Power BI dashboard    │
└─────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology |
|--|--|
| Cloud Storage | Azure Blob Storage (ADLS Gen2) |
| Event Notification | Azure Event Grid + Storage Queue |
| Data Warehouse | Snowflake |
| Ingestion | Snowpipe (AUTO_INGEST) |
| Incremental Processing | Snowflake Streams + Tasks |
| Transformation | Snowflake Dynamic Tables |
| Python Processing | Snowpark Python |
| Governance | Snowflake RBAC, Masking Policies, Row Access Policies |
| Visualization | Power BI |
| Architecture Pattern | Medallion (Bronze / Silver / Gold) |

---

## Data Model

### Source Data
| Dataset | Description | Records |
|--|--|--|
| Agents | Agent profiles — name, team, site, tenure | 10 agents |
| Tickets | Content moderation tickets — type, priority, SLA | 15 tickets |
| Performance | Daily agent metrics — AHT, accuracy, quality score | 15 records |
| Queue Logs | Raw JSON event logs from moderation queues | 10 events |

### Bronze Layer (Raw)
- `raw_agents` — agents as loaded from source
- `raw_tickets` — tickets as loaded from source
- `raw_performance` — performance metrics as loaded
- `raw_queue_logs` — JSON queue events (VARIANT column)
- `quarantine_records` — invalid records flagged for review and retry

### Silver Layer (Cleaned)
- `agents` — standardized agent data with record_status flag
- `tickets` — validated tickets with SLA breach indicator
- `performance` — validated metrics with performance tier
- `queue_events` — flattened JSON into structured columns

### Gold Layer (KPIs)
- `agent_scorecard` — per-agent composite score and performance tier
- `team_performance` — team-level aggregations by site
- `ticket_summary` — resolution rates and SLA compliance by date/category
- `daily_ops_report` — daily operational metrics with quality trend
- `queue_health` — backlog, escalation rate, throughput by queue

---

## Key Features

### 1. Automated Ingestion via Snowpipe
Files uploaded to Azure Blob Storage trigger an Event Grid notification, which is delivered to an Azure Storage Queue. Snowpipe consumes the queue and automatically runs COPY INTO — no manual intervention required. Data lands in Bronze within seconds of upload.

### 2. Self-Healing Bronze Layer 
Every record is evaluated on ingestion. Valid records are promoted to Silver. Invalid records — null IDs, negative values, out-of-range metrics — are quarantined with a human-readable error reason and a retry counter. A scheduled retry task attempts reprocessing every 30 minutes. After 3 failed retries, records are marked as permanently failed. Nothing is deleted.

### 3. Incremental Processing with Streams and Tasks
Snowflake Streams act as CDC (Change Data Capture) logs on Bronze tables. Tasks consume only new records since the last run — not full table scans. This keeps processing efficient regardless of table size.

### 4. Auto-Refreshing Gold Layer with Dynamic Tables
Gold KPIs are defined declaratively using Dynamic Tables with a TARGET_LAG of 2-5 minutes. Snowflake handles refresh scheduling, dependency tracking, and incremental updates automatically — no orchestration tool needed.

### 5. Snowpark Python Analytics
Agent performance is scored using a weighted model combining accuracy (40%), quality score (40%), and ticket volume (20%). Anomaly detection flags agents with quality drops greater than 10 points between consecutive days. Cohort analysis segments agents by tenure to compare KPI trends across experience levels.

### 6. Enterprise-Grade Governance
Four roles are implemented with explicit privilege separation: bpo_admin, bpo_analyst, bpo_operations, and bpo_agent. Column masking policies protect PII — email addresses, composite scores, and tenure values are masked based on role. Row access policies restrict data visibility at the team level and enforce agent self-service access (agents can only query their own records).

---

## Pipeline Flow

```
1. File uploaded to Azure Blob (raw folder)
2. Azure Event Grid detects BlobCreated event
3. Event published to Azure Storage Queue
4. Snowpipe reads queue → runs COPY INTO Bronze
5. Snowflake Stream captures new rows
6. Task runs every 5 minutes:
   a. Valid rows → Silver (standardized, enriched)
   b. Invalid rows → Quarantine (with error reason)
7. Dynamic Tables auto-refresh Gold KPIs every 2-5 minutes
8. Snowpark scores agents and detects anomalies
9. Power BI reads Gold views for dashboards
```

---

## Governance & Access Control

| Role | Schema Access | Row Access | Column Masking |
|--|--|--|--|
| bpo_admin | All schemas | All teams | No masking |
| bpo_analyst | Silver + Gold | Alpha + Beta teams | Partial email |
| bpo_operations | Gold only | All teams | Masked name, masked score |
| bpo_agent | Gold only | Own records only | Masked score, masked tenure |

---

## Dashboard (Power BI)

Six reporting pages connected to Snowflake Gold views:

- **Agent Scorecard** — individual KPIs, composite score, performance tier ranking
- **Team Performance** — team comparison by quality, AHT, and throughput
- **Ticket Analytics** — volume trends, SLA breach rate, resolution rate by category
- **Daily Operations** — daily headcount, quality trend, site comparison
- **Queue Health** — backlog by queue, escalation rate, open vs closed trends
- **Data Quality** — quarantine summary, error breakdown, pending vs failed records

---

## Known Issues & Planned Improvements

- **Quarantine stream consumption** — the stream is consumed by Step A (valid insert) before Step B (quarantine) can process bad records. Fix: stage stream into a temp table before splitting valid/invalid. Currently bad records remain in Bronze raw tables and do not silently disappear.
- **Schema drift handling** — currently not implemented. Incoming files with added or removed columns are not automatically detected. Planned: schema registry table with comparison task.
- **Larger dataset** — current sample data is small (10-15 records per table). Production version would use generated datasets of 100K+ rows per table for meaningful KPI aggregations.

---

## Project Structure

```
bpo_db/
├── bronze/
│   ├── raw_agents
│   ├── raw_tickets
│   ├── raw_performance
│   ├── raw_queue_logs
│   ├── quarantine_records
│   ├── stream_agents
│   ├── stream_tickets
│   ├── stream_performance
│   ├── stream_queue_logs
│   ├── pipe_agents
│   ├── pipe_tickets
│   ├── pipe_performance
│   ├── pipe_queue_logs
│   ├── task_silver_agents
│   ├── task_silver_tickets
│   ├── task_silver_performance
│   ├── task_silver_queue_logs
│   └── task_retry_quarantine
├── silver/
│   ├── agents
│   ├── tickets
│   ├── performance
│   ├── queue_events
│   ├── role_access_map
│   └── agent_user_map
├── gold/
│   ├── agent_scorecard (Dynamic Table)
│   ├── team_performance (Dynamic Table)
│   ├── ticket_summary (Dynamic Table)
│   ├── daily_ops_report (Dynamic Table)
│   ├── queue_health (Dynamic Table)
│   ├── agent_ml_scores
│   ├── tenure_cohort_analysis
│   ├── vw_agent_scorecard
│   ├── vw_team_performance
│   ├── vw_ticket_summary
│   ├── vw_daily_ops
│   ├── vw_queue_health
│   └── vw_quarantine_summary
└── audit/
    └── performance_anomalies
```

