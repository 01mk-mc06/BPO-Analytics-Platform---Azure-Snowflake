# Azure Infrastructure Setup [Free Trial]
## Analytics Platform — Snowflake + Azure Integration

---

## Overview

This document covers the Azure infrastructure built to support the BPO Analytics Platform. The Azure side handles raw file storage and event-driven notifications that trigger Snowflake's auto-ingestion pipeline.

```
azure blob storage (file landing zone)
        ↓ blob created event
azure event grid (event routing)
        ↓ routes to queue
azure storage queue (event buffer)
        ↓ snowpipe polls queue
snowflake snowpipe (auto-ingest)
```

---

## Prerequisites

- Azure account (free trial or paid)
- Snowflake account with ACCOUNTADMIN role

---

## Part 1: Resource Group

A resource group is a logical container for all Azure resources in this project.

### Steps

1. Azure Portal → **Resource Groups → + Create**
2. Fill in:
```
Subscription:    <your_subscription>
Resource group:  <your_resource_group>
Region:          Southeast Asia
```
3. Click **Review + Create → Create**

---

## Part 2: Storage Account

The storage account hosts the Blob container where raw data files are uploaded.

### Steps

1. Azure Portal → **Storage Accounts → + Create**
2. Fill in:
```
Subscription:        <your_subscription>
Resource group:      <your_resource_group>
Storage account name: <your_storage_account>
Region:              Southeast Asia
Performance:         Standard
Redundancy:          LRS (Locally Redundant Storage)
```
3. Click **Review + Create → Create**

### Why LRS?
LRS is the lowest cost redundancy option — sufficient for a development/demo environment. For production, use GRS (Geo-Redundant Storage).

---

## Part 3: Blob Container

The container is the folder-level grouping inside the storage account where files are uploaded.

### Steps

1. Storage Account → **Containers → + Container**
2. Fill in:
```
Name:               <your_container>
Public access level: Private (no anonymous access)
```
3. Click **Create**

### Create Raw Folder

Blob Storage doesn't have real folders — folders are simulated by file path prefixes. To create a `raw` folder:

1. Click into your container
2. Click **+ Add Directory**
3. Name it `raw`

### Upload Data Files

1. Click into the `raw` folder
2. Click **Upload**
3. Upload all 4 files:
   - `agents.csv`
   - `tickets.csv`
   - `agent_performance.csv`
   - `queue_logs.json`

---

## Part 4: Storage Queue

The storage queue acts as a message buffer between Azure Event Grid and Snowpipe. When a file is uploaded to Blob Storage, Event Grid sends a message to this queue, and Snowpipe polls the queue to trigger ingestion.

### Steps

1. Storage Account → **Queues → + Queue**
2. Fill in:
```
Name: <your_queue_name>
```
3. Click **OK**

---

## Part 5: Snowflake Storage Integration

Before setting up Event Grid, establish trust between Snowflake and Azure so Snowflake can read from Blob Storage.

### Step 1: Create Storage Integration in Snowflake

```sql
create or replace storage integration <your_storage_integration>
  type = external_stage
  storage_provider = 'azure'
  enabled = true
  azure_tenant_id = '<your_tenant_id>'
  storage_allowed_locations = ('azure://<your_storage_account>.blob.core.windows.net/<your_container>/');
```

### Step 2: Get Consent URL and App Name

```sql
desc integration <your_storage_integration>;
```

Copy two values from the output:
- `AZURE_CONSENT_URL` — URL to grant Snowflake access
- `AZURE_MULTI_TENANT_APP_NAME` — the Snowflake app identity in Azure

### Step 3: Accept Consent URL

1. Open `AZURE_CONSENT_URL` in your browser
2. Sign in with your Azure account
3. Click **Accept**

### Step 4: Assign IAM Role — Blob Access

1. Azure Portal → Storage Account → **Access Control (IAM)**
2. Click **+ Add → Add Role Assignment**
3. Fill in:
```
Role:    Storage Blob Data Contributor
Assign access to: User, group, or service principal
Member:  <AZURE_MULTI_TENANT_APP_NAME from DESC INTEGRATION>
```
4. Click **Review + Assign**

> Wait 5–15 minutes for IAM propagation before testing.

---

## Part 6: Snowflake Notification Integration

A separate integration is needed specifically for Snowpipe to read event notifications from the Azure Storage Queue.

### Step 1: Create Notification Integration in Snowflake

```sql
create or replace notification integration <your_notification_integration>
  enabled = true
  type = queue
  notification_provider = azure_storage_queue
  azure_storage_queue_primary_uri = 'https://<your_storage_account>.queue.core.windows.net/<your_queue_name>'
  azure_tenant_id = '<your_tenant_id>';
```

### Step 2: Get Consent URL

```sql
desc integration <your_notification_integration>;
```

Copy `AZURE_CONSENT_URL` and `AZURE_MULTI_TENANT_APP_NAME`.

### Step 3: Accept Consent URL

Open `AZURE_CONSENT_URL` in browser → Sign in → **Accept**

### Step 4: Assign IAM Roles — Queue Access

Two roles are required for the queue:

1. Azure Portal → Storage Account → **Access Control (IAM) → + Add Role Assignment**

**Role 1:**
```
Role:   Storage Queue Data Contributor
Member: <AZURE_MULTI_TENANT_APP_NAME from notification integration>
```

**Role 2:**
```
Role:   Storage Queue Data Message Processor
Member: <AZURE_MULTI_TENANT_APP_NAME from notification integration>
```

> Both roles are required. Missing either one causes 403 errors.

---

## Part 7: Event Grid Subscription

Event Grid routes Blob Storage events (file uploads) to the Storage Queue so Snowpipe gets notified automatically.

### Step 1: Create System Topic

> **Important:** Create the System Topic from the Storage Account — not from the Event Grid menu directly.

1. Azure Portal → Storage Account → **Events**
2. Click **+ Event Subscription**

### Step 2: Fill in Event Subscription Details

```
Name:          <your_event_subscription_name>
Event Schema:  Event Grid Schema
```

**Event Types:**
- Uncheck all
- Check only ✅ **Blob Created**

**Endpoint Details:**
```
Endpoint Type: Storage Queue
```

Click **Configure an endpoint**:
```
Subscription:    <your_subscription>
Storage Account: <your_storage_account>
Queue:           <your_queue_name>
```

Click **Confirm Selection**

### Step 3: Create

Click **Review + Create → Create**

### Verify Event Subscription

1. Storage Account → **Events → Event Subscriptions**
2. Your subscription should show `Succeeded` status

---

## Part 8: External Stage in Snowflake

Create the external stage that points Snowflake to the Azure Blob container.

```sql
create or replace stage <your_database>.<your_schema>.<your_stage_name>
  url = 'azure://<your_storage_account>.blob.core.windows.net/<your_container>/'
  storage_integration = <your_storage_integration>
  file_format = <your_file_format>;

-- verify files are visible
list @<your_stage_name>;
list @<your_stage_name>/raw/;
```

---

## Part 9: Snowpipe Setup

Snowpipe auto-ingests files when it receives a notification from the Storage Queue.

```sql
create or replace pipe <your_database>.<your_schema>.<your_pipe_name>
  auto_ingest = true
  integration = '<your_notification_integration>'
as
copy into <your_table>
from (
  select $1, $2, ..., metadata$filename
  from @<your_stage_name>/raw/<your_file.csv>
)
file_format = <your_file_format>;
```

### Verify Pipe Status

```sql
select system$pipe_status('<your_database>.<your_schema>.<your_pipe_name>');
```

Look for:
- `executionState: "RUNNING"` — pipe is active
- `pendingFileCount` — files waiting to be processed
- `lastIngestedTimestamp` — when last file was loaded

### Load Existing Files

```sql
-- force load files already in the stage
alter pipe <your_pipe_name> refresh;
```

---

## Part 10: IAM Role Summary

| Role | Scope | Purpose |
|--|--|--|
| Storage Blob Data Contributor | Storage Account | Snowflake reads blob files |
| Storage Queue Data Contributor | Storage Account | Snowpipe writes to queue |
| Storage Queue Data Message Processor | Storage Account | Snowpipe reads and deletes queue messages |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│                   AZURE                         │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │         Storage Account                  │  │
│  │                                          │  │
│  │  ┌─────────────┐   ┌─────────────────┐  │  │
│  │  │    Blob     │   │  Storage Queue  │  │  │
│  │  │  Container  │   │                 │  │  │
│  │  │  /raw/      │   │  event buffer   │  │  │
│  │  └──────┬──────┘   └────────▲────────┘  │  │
│  │         │                   │           │  │
│  │         │  BlobCreated      │           │  │
│  │         ▼                   │           │  │
│  │  ┌─────────────────────┐    │           │  │
│  │  │    Event Grid       │────┘           │  │
│  │  │    Subscription     │               │  │
│  │  └─────────────────────┘               │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
                        │
                        │ queue notification
                        ▼
┌─────────────────────────────────────────────────┐
│                  SNOWFLAKE                      │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │  Notification Integration                │  │
│  │  (polls storage queue)                   │  │
│  └──────────────────┬───────────────────────┘  │
│                     │                          │
│                     ▼                          │
│  ┌──────────────────────────────────────────┐  │
│  │  Snowpipe (AUTO_INGEST = TRUE)           │  │
│  │  ├── pipe_agents                         │  │
│  │  ├── pipe_tickets                        │  │
│  │  ├── pipe_performance                    │  │
│  │  └── pipe_queue_logs                     │  │
│  └──────────────────┬───────────────────────┘  │
│                     │                          │
│                     ▼                          │
│           Bronze Layer Tables                  │
└─────────────────────────────────────────────────┘
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|--|--|--|
| `ContainerNotFound` | Wrong container name in stage URL | Check Azure Portal → copy exact container name |
| `Integration not allowed` | Stage URL doesn't match integration allowed locations | Recreate integration with correct URL |
| `403 AuthorizationPermissionMismatch` | IAM roles not propagated | Wait 5–15 mins after role assignment |
| `Integration cannot be null` | Pipe using SAS token stage instead of integration stage | Recreate stage using storage integration |
| `DIRECTION invalid` | Old parameter used for Azure Storage Queue notification | Remove `DIRECTION` parameter |
| Pipe not auto-triggering | Event Grid subscription not created | Create event subscription from Storage Account → Events |
| Files not showing in LIST | Wrong subfolder path | Use `list @stage/raw/` not `list @stage/` |

---

## Cost Considerations (Free Trial)

| Resource | Cost |
|--|--|
| Storage Account (LRS) | ~$0.02/GB/month |
| Storage Queue | ~$0.004 per 10,000 operations |
| Event Grid | First 100,000 events/month free |
| Blob Storage transactions | Minimal for small datasets |

> For a development/demo environment with small datasets, total Azure cost is well under $1/month.

---

## Cleanup — Suspend Resources When Not in Use

To avoid unnecessary costs:

```sql
-- suspend snowflake warehouses
alter warehouse <your_ingestion_warehouse> suspend;
alter warehouse <your_transform_warehouse> suspend;

-- pause snowpipes
alter pipe <your_database>.<your_schema>.pipe_agents      pause;
alter pipe <your_database>.<your_schema>.pipe_tickets     pause;
alter pipe <your_database>.<your_schema>.pipe_performance pause;
alter pipe <your_database>.<your_schema>.pipe_queue_logs  pause;

-- suspend tasks
alter task <your_database>.<your_schema>.task_silver_agents      suspend;
alter task <your_database>.<your_schema>.task_silver_tickets     suspend;
alter task <your_database>.<your_schema>.task_silver_performance suspend;
alter task <your_database>.<your_schema>.task_silver_queue_logs  suspend;
```

In Azure, storage costs are minimal even when idle — no need to delete resources unless you want to avoid all charges.

---
