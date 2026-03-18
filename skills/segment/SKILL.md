---
name: segment
description: Search and explain segment-related lambdas, API endpoints, data models, query builders, UI components, creation flows, and debugging across datalayerapp-v2 and datalayerapp-v2-function-code repositories.
---

# Segment Skill

This skill helps you understand, debug, and navigate segment-related code across the Datalayer platform. Segments allow users to define custom filters and rule groups to partition analytics data (sessions, users, people, companies, web conversions, CRM objects) into dedicated BigQuery tables for reporting and analysis.

## Invocation

When the user asks about segments, use this knowledge base to:
1. Identify which files, lambdas, and query builders are relevant
2. Explain the end-to-end segment creation, rebuild, and reporting flow
3. Help debug segment-related issues by pointing to the right logs, tables, caches, and code
4. Explain the rule group structure, condition types, and query generation

## Repositories

| Repository | Path | Role |
|---|---|---|
| **datalayerapp-v2** | `D:\Projects\Datalayer\datalayerapp-v2` | Main Express.js app (API, models, query builders, client UI) |
| **datalayerapp-v2-function-code** | `D:\Projects\Datalayer\datalayerapp-v2-function-code` | AWS Lambda functions for BigQuery job processing |

---

## What Are Segments?

Segments are user-defined filter rules that group analytics data into subsets. Each segment:

- Targets a specific **object type** (session, user, people, company, web conversion, or CRM objects)
- Contains one or more **rule groups** with conditions (property, metric, relationship, event-based)
- Generates a dedicated **BigQuery table** (`segment_{segmentId}`) containing matching object IDs
- Can be **applied to reports** to filter dashboard and analytics data
- Supports **incremental updates** via cronjob queries

Every account has a default **"All Data"** segment (non-custom, no rules, always active).

---

## Segment Objects (What You Can Segment)

### ListenLayer Objects
| Object | Constant | Description |
|---|---|---|
| Event | `event` | Individual tracked events |
| Session | `session` | Browser sessions |
| User | `user` | Anonymous users (cookie-based) |
| People | `people` | Identified people (email/form) |
| Company | `revealed-company` | Companies (revealed from IP/domain + imported via CRM/target accounts) |
| Web Conversion | `web-conversion` | Web conversion events |

### CRM Objects (Salesforce Integration)
| Object | Constant |
|---|---|
| Lead | `crm-lead` |
| Contact | `crm-contact` |
| Account | `crm-account` |
| Parent Account | `crm-parent-account` |
| Opportunity | `crm-opportunity` |
| Quote | `crm-quote` |
| Order | `crm-order` |
| Contract | `crm-contract` |
| Campaign | `crm-campaign` |
| Event | `crm-event` |

---

## Condition Focus Types

Each rule group uses one condition focus type:

| Focus | Constant | Description |
|---|---|---|
| Property | `property` | Filter by object property values (dimension, custom, date, etc.) |
| Metric | `metric` | Filter by numeric metric values (counts, durations, scores) |
| Web Conversion Event | `web-conversion-event` | Filter by web conversion event properties |
| Ecommerce Event | `ecommerce-event` | Filter by ecommerce event properties |
| Sales Conversion Event | `sales-conversion-event` | Filter by sales conversion event properties |
| Relationship | `relationship` | Filter by cross-object relationship counts |
| Other Event | `other-event` | Filter by custom event listener data |

---

## Segment Field Types

| Type | Constant | Description |
|---|---|---|
| Dimension | `dimension` | Standard string dimensions |
| Metric | `metric` | Numeric metrics |
| Custom | `custom` | Custom properties |
| Conversion | `conversion` | Web conversion metrics |
| Sales Conversion | `sales-conversion` | Sales conversion metrics |
| Ecommerce | `ecommerce` | Ecommerce metrics |
| Trigger | `trigger` | Server-side trigger metrics |
| CRM Lifetime Count | `crm-lifetime-count` | CRM relationship counts |
| Date | `date` | Date-type fields |
| Multi Value | `multi-value` | Array/multi-select fields |
| Boolean | `boolean` | True/false fields |
| Number | `number` | Numeric fields |
| Intent | `intent` | Intent property data |

---

## RuleGroups Data Structure

```javascript
ruleGroups: [
  {
    mainOperator: 'include' | 'exclude',      // Include/exclude matching records
    conjunction: 'and' | 'or',                // Combine conditions within rule
    segmentObject: 'session' | 'user' | ...,  // Target object type
    conditionFocus: 'property' | 'metric' | ..., // Condition focus type
    embeddedSegmentId: 'uuid-or-null',         // (Optional) ID of another segment embedded into this rule group
    dateRangeFilter: {
      type: 'lifetime-of-the-object' | 'previous-period' | 'specific-date-range',
      numberOfDaysPrevious: 30,
      startDate: '2024-01-01',
      endDate: '2024-01-31'
    },
    // Conditions (g0, g1, g2, ...)
    g0: {
      conjunction: 'or',
      type: 'dimension' | 'metric' | 'custom' | ...,
      key: 'propertyName',
      subKey: 'subPropertyName',
      subKeyLv2: 'deepPropertyName',
      condition: 'equals' | 'contains' | ...,
      value: 'filterValue',
      property: 'eventProperty',
      id: 'conversionId'
    },
    // ... g1, g2, etc.
  }
]
```

---

## API Endpoints (datalayerapp-v2)

**File**: `server/routes/segments.js`

### Segment CRUD
| Method | Path | Controller | Purpose |
|---|---|---|---|
| GET | `/client/segments/:accountId` | `getAllSegmentByAccount` | List all segments for account |
| GET | `/client/segment/:id` | `getSegmentsById` | Get segment by ID |
| POST | `/client/segment` | `createSegment` | Create new segment |
| PUT | `/client/segment` | `updateSegment` | Update segment |
| DELETE | `/client/segment` | `removeSegment` | Delete segment |
| PUT | `/client/segment/expire/:accountId` | `expireAllSegmentsByAccount` | Expire/refresh all segments |

### Embedded Segments
| Method | Path | Controller | Purpose |
|---|---|---|---|
| GET | `/client/segments/:accountId/available-for-embedding/:segmentId?` | `getAvailableSegmentsForEmbedding` | Get segments available to embed (filters out self, circular deps, non-custom) |

### Segment Options & Preview
| Method | Path | Controller | Purpose |
|---|---|---|---|
| POST | `/client/segment/option` | `getOptionByAccount` | Get filter options for segment builder |
| POST | `/client/preview-inside-segment` | `previewInsideSegment` | Preview sample data for condition |
| POST | `/client/segment-testing/` | `handleSegmentTesting` | Test/validate segment queries |

### Segment Reports
| Method | Path | Controller | Purpose |
|---|---|---|---|
| GET | `/client/segment-report/:accountId` | `getAllSegmentReportByAccount` | Get segment report configs |
| PUT | `/client/segment-report` | `updateSegmentReport` | Link/unlink segment to report |
| DELETE | `/client/segment-report` | `removeSegmentReport` | Remove segment report config |
| PUT | `/client/segment-chart` | `applySegmentForChart` | Apply segment to charts |

### Other
| Method | Path | Controller | Purpose |
|---|---|---|---|
| POST | `/client/rebuild-segment` | `handleRebuildSegment` | Rebuild segment BigQuery table |
| POST | `/client/create-default-segment` | `handleCreateDefaultSegment` | Create default "All Data" segment |

---

## Server-Side Architecture

### Controller
**File**: `server/controllers/segments.js`

Thin middleware layer that extracts request data and delegates to services. Each function receives `req.body` or `req.params` and calls the corresponding service function with `accountTimeZone` from the active account.

### Service
**File**: `server/services/segments.js`

Core business logic (key functions):

| Function | Purpose |
|---|---|
| `handleCreateSegment` | Generate UUID, validate embedded segments, build BigQuery CREATE TABLE query via `getSegmentData()`, queue BigQuery job, serialize ruleGroups, insert DB record, sync `isEmbeddedBy` tracking, clear Redis cache |
| `handleUpdateSegment` | Update name/description/status/queryStatus, validate embedded segments, regenerate BigQuery table if queryStatus is IN_PROGRESS, sync `isEmbeddedBy` tracking only when embedded IDs changed, clear caches, update related report views |
| `handleRemoveSegment` | Block deletion if segment is embedded by others (returns parent names in error), clean up `isEmbeddedBy` on child segments, delete from DB, drop BigQuery table, clear cache |
| `handleGetSegmentById` | Fetch by ID, deserialize ruleGroups, strip query field |
| `handleGetAllSegmentByAccount` | Redis-cached retrieval, fallback to DB, deserialize all ruleGroups |
| `handleGetOptionDataByAccount` | Fetch filter options (CHECKLIST, OTHER_FIELD, PIPELINE_STATUS, INTENT, METRIC, CONVERSION, ECOMMERCE, EVENT_NAME, EVENT_VARIABLE, SALES_CONVERSION), 1-hour Redis cache |
| `handlePreviewInsideSegment` | Query BigQuery for popularValue, sampleValue, uniqueValue; 1-hour Redis cache |
| `handleExpireAllSegmentsByAccount` | Refresh all custom segments, rebuild cronjob queries, optionally drop BigQuery tables |
| `handleRebuildSegment` | Rebuild specific segment, set queryStatus to IN_PROGRESS, queue new BigQuery job |
| `handleUpdateSegmentReport` | Link/unlink segments to reports, max 4 concurrent segments per report, handle sticky behavior |
| `handleApplySegmentForChart` | Apply selected segments to chart visualizations |
| `scriptCreateDefaultSegment` | Create default "All Data" segment per account |
| `extractEmbeddedSegmentIds` | Extract embedded segment IDs from parsed ruleGroups array |
| `checkCircularDependency` | BFS traversal to detect circular embedding chains (supports in-memory dependency map for batch checks) |
| `validateEmbeddedSegments` | Validates no duplicates, no self-embedding, all exist, no circular dependencies |
| `syncEmbeddedByTracking` | Diffs old vs new embedded IDs and calls `updateEmbeddedByBulk` to sync `isEmbeddedBy` arrays |
| `handleGetAvailableSegmentsForEmbedding` | Returns custom segments available for embedding, filtered by circular dependency checks using in-memory dependency graph |

### Segment Sync Service
**File**: `server/services/segmentSync.js`

Handles cascade rebuilds when an embedded segment is updated:

| Function | Purpose |
|---|---|
| `updateSegmentStatus` | Helper that updates segment `queryStatus` in DB and sends Pusher notification in one call. Used by `queueSegmentRebuild` |
| `triggerCascadeRebuild` | When an embedded segment changes, finds all parents via `isEmbeddedBy` and queues rebuilds in parallel (`Promise.all`). Redis-backed cycle protection (visited set, 1hr TTL). No debounce — relies on cycle protection and queue dedup |
| `queueSegmentRebuild` | Calls `updateSegmentStatus` to set `queryStatus` to IN_PROGRESS, builds new BigQuery table, queues check job. Redis dedup prevents double-queuing. On failure or empty query, sets FAILED via `updateSegmentStatus` |

### Segment Testing Service
**File**: `server/services/segmentTesting.js`

Function `handleSegmentTesting`: Tests segment queries for validity across all object types and condition focus combinations. Validates paths against event schema. Returns query validation results and error reporting.

### Model
**File**: `server/models/segments.js`

#### Segments Table Operations
| Method | Purpose |
|---|---|
| `createSegment(data)` | Insert new segment |
| `removeSegment(id)` | Delete by UUID |
| `updateSegment(id, data)` | Update fields |
| `updateSegmentByAccount(accountId, payload)` | Batch update by account |
| `findSegmentById(id)` | Get by ID |
| `findSegmentByIds(ids)` | Get multiple |
| `findSegmentByAllData(accountId)` | Get default "All Data" segment |
| `findSegmentByAccountId(accountId)` | Get all for account |
| `getCustomSegmentsByAccountId(accountId)` | Get only custom segments |
| `findSegmentEnableByAccountId(accountId)` | Get active segments only |
| `addEmbeddedByParent(segmentId, parentId)` | Add parent ID to segment's `isEmbeddedBy` JSONB array (prevents duplicates via `@>` check) |
| `removeEmbeddedByParent(segmentId, parentId)` | Remove parent ID from segment's `isEmbeddedBy` array using `jsonb_agg` filter |
| `findSegmentsEmbeddedBy(parentId)` | Find all segments whose `isEmbeddedBy` contains a given parent ID |
| `getEmbeddedByParents(segmentId)` | Get the `isEmbeddedBy` array for a segment |
| `updateEmbeddedByBulk(toAdd, toRemove, parentId)` | Batch add/remove parent from multiple segments' `isEmbeddedBy` arrays |

#### SegmentReport Table Operations
| Method | Purpose |
|---|---|
| `createSegmentReports(data)` | Create report/segment association |
| `findSegmentReportById(id)` | Get config by ID |
| `updateSegmentReport(id, data)` | Update config |
| `updateApplyForChart(data)` | Mark segments as applied to charts |
| `unsetApplyForChart(data)` | Remove chart application |
| `disableSegmentReportByAccount(data)` | Disable for report/user |
| `findSegmentReportByAccountId(accountId)` | Get all configs with segment names |
| `findAllSegmentReportByAccountId(accountId, reportName, userId, object)` | Filtered retrieval |
| `removeSegmentReport(id)` | Delete config |

---

## Database Tables

### Segments Table
**Migration**: `server/migrations/20240517005932_create_table_segments.js`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `accountId` | UUID | FK to Accounts (CASCADE) |
| `name` | VARCHAR(200) | Not null |
| `description` | TEXT | Nullable |
| `isCustom` | BOOLEAN | Default FALSE |
| `ruleGroups` | TEXT | Serialized JSON |
| `status` | BOOLEAN | Default TRUE (enabled/disabled) |
| `queryStatus` | VARCHAR(200) | Default 'in-progress' |
| `query` | TEXT | BigQuery cronjob query |
| `isEmbeddedBy` | JSONB | Array of parent segment IDs that embed this segment. Default `[]`. GIN-indexed for fast `@>` lookups |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

### SegmentReport Table
**Migration**: `server/migrations/20240530040609_tcreate_table_segmentReport.js`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `accountId` | UUID | FK to Accounts (CASCADE) |
| `segmentId` | UUID | FK to Segments (CASCADE) |
| `reportName` | VARCHAR(200) | Nullable |
| `userId` | UUID | Nullable |
| `reportType` | VARCHAR(200) | Nullable |
| `object` | VARCHAR(200) | For CRM objects |
| `apply` | BOOLEAN | Default TRUE |
| `applyChart` | BOOLEAN | Nullable |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

---

## BigQuery Query Builders

**Directory**: `server/models/query-builders/common/segment/`

### Main Query Builder (`index.js`)

| Export | Purpose |
|---|---|
| `getSegmentData(data)` | **Entry point**. Generates `createTableQuery` (CREATE OR REPLACE TABLE) and `cronjobQuery` (INSERT INTO for incremental updates). For reports: returns `cteQuery` + `filterCondition`. |
| `getSegmentQuery(data)` | Builds complete segment BigQuery table with CTEs per rule group. Combines rules with AND/OR. Handles both LL and CRM objects. Salesforce path: COMPANY and PEOPLE segments add UNION ALL subqueries for imported records (from `companiesStats`/`peopleStats` where `dataSourcePath = 'Imported'`). Non-Salesforce path: standard session→user→people join chain only. |
| `getRuleQuery(data)` | Routes individual rule to appropriate builder by conditionFocus. Returns SELECT DISTINCT objectId with CTE conditions. If rule group has `embeddedSegmentId`, adds `objectId IN (SELECT objectId FROM segment_table)` filter (AND with regular conditions, or standalone if no conditions). Salesforce-aware: includes `type = 'ObjectType'` filter when enabled. |
| `getPreventInsideSegmentQuery(data)` | Builds preview queries: popular values, sample values, unique count. |
| `getFinalSegmentQuery(data)` | Constructs final report query applying segment filters with LIMIT/OFFSET/sorting. |

### Sub-Builders

| File | Function | Handles |
|---|---|---|
| `property-metric/index.js` | `getObjectDataByPropertyMetric` | Routes to object-specific builders (session, user, people, company, webConversion, CRM) |
| `property-metric/sessionObject.js` | Session property/metric conditions | Session-level CTEs and filters |
| `property-metric/userObject.js` | User property/metric conditions | User-level CTEs and filters |
| `property-metric/peopleObject.js` | People property/metric conditions | People-level CTEs and filters |
| `property-metric/companyObject.js` | Company property/metric conditions | Company-level CTEs and filters |
| `property-metric/webConversionObject.js` | Web conversion property/metric conditions | Web conversion CTEs and filters |
| `property-metric/crmObject.js` | CRM property/metric conditions | CRM object CTEs and filters |
| `optimizedView.js` | `getObjectDataByOptimizedView` | WEB_CONVERSION_EVENT, ECOMMERCE_EVENT, SALES_CONVERSION_EVENT conditions |
| `relationship.js` | `getObjectDataByRelationship` | RELATIONSHIP conditions (cross-object count filters). COMPANY uses `companiesStats` (RevealedCompaniesStats) as main table with pre-computed `sessionCount`, `userCount`, `peopleCount`; eventCount via metric CTE from session join chain. PEOPLE uses `peopleStats` with pre-computed `revealedCompanyCount`, `userID` array. Other LL objects use their own stats table; CRM objects use `crmRelation`. |
| `otherEvent.js` | `getObjectDataByOtherEvent` | OTHER_EVENT conditions (custom event variables, listener data) |
| `segmentUtils.js` | Utility functions | Date range filters, field aliases, UNNEST subqueries, table name resolution, property/metric filters, checklist filters |

### Query Generation Flow

```
getSegmentData({ segment, accountId, accountTimeZone? })
  |
  v
Auto-fetch accountTimeZone from AccountModel if not provided
  |
  v
For each ruleGroup:
  getRuleQuery({ group, conditionFocus, segmentObject })
    |
    +-- PROPERTY/METRIC --> getObjectDataByPropertyMetric()
    |     +-- SESSION --> sessionObject.js
    |     +-- USER --> userObject.js
    |     +-- PEOPLE --> peopleObject.js
    |     +-- COMPANY --> companyObject.js
    |     +-- WEB_CONVERSION --> webConversionObject.js
    |     +-- CRM_* --> crmObject.js
    |
    +-- WEB_CONVERSION_EVENT/ECOMMERCE_EVENT/SALES_CONVERSION_EVENT
    |     --> getObjectDataByOptimizedView()
    |
    +-- RELATIONSHIP --> getObjectDataByRelationship()
    |
    +-- OTHER_EVENT --> getObjectDataByOtherEvent()
    |
    +-- If embeddedSegmentId present:
          +-- WITH conditions: AND objectId IN (SELECT objectId FROM segment_{embeddedId})
          +-- WITHOUT conditions: SELECT DISTINCT objectId FROM segment_{embeddedId}
  |
  v
Combine rules with AND (INTERSECT DISTINCT) or OR (UNION)
  |
  v
Return: { createTableQuery, cronjobQuery }
```

---

## Lambda Functions (datalayerapp-v2-function-code)

### 1. HandleSegmentBigQuery
- **Path**: `lambda-functions/HandleSegmentBigQuery/`
- **Handler**: `index.js`
- **Trigger**: SQS (`HANDLE_SEGMENT_BIG_QUERY`)
- **Purpose**: Processes segment cronjob queries for all accounts using SQS self-queuing for level-by-level execution
- **Operating Modes**:
  - **Initial invocation**: Receives `accountId` and `timeZone` from SQS, fetches segments, computes dependency levels, starts processing level 0
  - **Continuation invocation**: Receives serialized state (levels, pendingJobs, failedSegments, checkCount), checks if previous level's BigQuery jobs are complete, advances to next level or re-queues
- **Constants**: `CHECK_DELAY_SECONDS = 180` (3 min between status checks), `MAX_CHECK_COUNT = 10` (30 min max wait per level)
- **What it does**:
  1. Receives SQS message with `accountId` and `timeZone`
  2. Queries PostgreSQL for active, non-blocked segments via `SegmentModel.findByAccountId()`
  3. Uses `getProcessingOrder()` to topologically sort segments into dependency levels
  4. Serializes levels (keeping only: id, name, query, ruleGroups) for SQS state passing
  5. Calls `processNextLevel()` for level 0:
     - `filterSegmentsByDependencies()` — excludes segments whose embedded dependencies failed
     - `submitLevelJobs()` — submits all BigQuery batch jobs in parallel via `Promise.allSettled`. For each segment: parses ruleGroups, replaces date placeholders (`{{startDate_*}}`, `{{endDate_*}}`) via `replaceDateRangeQuery()`, sanitizes table name, calls `queryBQBatch()`
     - **Last level**: Submits jobs and returns without waiting (no dependents need the result)
     - **Non-last levels**: Self-queues SQS message with 180s delay containing `pendingJobs`, `levels`, `failedSegments`, `checkCount`
  6. On continuation invocation: `checkPendingJobs()` checks BigQuery job statuses via `checkBigQueryJobStatus()`:
     - All done → advance to next level via `processNextLevel()`
     - Still running and `checkCount < MAX_CHECK_COUNT` → re-queue with incremented `checkCount`
     - `checkCount >= MAX_CHECK_COUNT` → treat remaining as failed, advance to next level
  7. After all levels complete: `processSSTriggers()` reads server-side trigger rules from S3 (`server-side-trigger/{accountId}.json`). If People/Company/Sales Conversion Event targets exist and hour < 12 in account timezone, fetches schedule triggers via `getAllScheduleSSTrigger()` and sends SQS to schedule report queue (chunked in batches of 100, 900s delay)
- **Filters** (`findByAccountId`): `isBlocked = false`, `queryStatus = 'done'`, `query IS NOT NULL`, `totalBytesBilled < 10` (GB), `platformAccountStatus` NOT IN `['Canceled', 'Dormant (permanent)', 'Dormant (temporary)']`
- **Key files**: `models/segments.js`, `lib/bigquery.js`, `s3/readFileStaticJson.js`, `helpers.js`, `utils/dependencyGraph.js`, `utils/index.js`
- **Key functions**:
  - `submitLevelJobs(segments, accountId, timeZone)` — Parse ruleGroups, replace date placeholders, submit BigQuery batch jobs in parallel. Returns `{ success, segmentId, bigQueryJobId }` per segment
  - `checkPendingJobs(pendingJobs)` — Check BigQuery job statuses. Returns `{ allDone, failedSegmentIds }`
  - `filterSegmentsByDependencies(segments, failedSegments)` — Exclude segments with failed embedded dependencies
  - `processNextLevel(params)` — Orchestrate level processing: filter → submit → self-queue or return
  - `processSSTriggers(accountId, timeZone)` — Post-processing: send scheduled report triggers
  - `sendScheduleTriggerSavedReport(triggers, accountId, timeZone, targetObject)` — Chunk and send SQS messages for schedule triggers

### Dependency Graph Utility (HandleSegmentBigQuery)
- **Path**: `lambda-functions/HandleSegmentBigQuery/utils/dependencyGraph.js`
- **Purpose**: Topological sorting and cycle detection for segment processing order
- **Exports**:
  - `extractEmbeddedSegmentIds(segment)` — Extract embedded IDs from a segment's ruleGroups (handles string or object). Returns empty array on parse error
  - `buildDependencyGraph(segments)` — Build nodes/edges graph from segment array. Only includes edges where both segments exist in current dataset
  - `hasCycle(graph)` — DFS-based cycle detection using recursion stack
  - `topologicalSort(segments)` — Kahn's algorithm using reverse dependency map and in-degree counting. Returns segments grouped by processing level. Falls back to single level on cycle detection
  - `getProcessingOrder(segments)` — Entry point. Fast path (single level) if no segments have embeddings, otherwise runs full topological sort

### BigQuery Integration (HandleSegmentBigQuery)
- **Path**: `lambda-functions/HandleSegmentBigQuery/lib/bigquery.js`
- **Exports**:
  - `queryBQBatch({ query, accountId, type, tableName })` — Creates BigQuery batch job with `useLegacySql: false`, `priority: 'BATCH'`, labels for `account_id`/`type`/`table_name`. Returns job object or undefined on error
  - `checkBigQueryJobStatus(jobId)` — Retrieves job metadata, returns `{ jobId, status }`. Status values: `PENDING`, `RUNNING`, `DONE`, `ERROR`, `NOT_FOUND`

### Utilities (HandleSegmentBigQuery)
- **Path**: `lambda-functions/HandleSegmentBigQuery/utils/index.js`
- **Exports**:
  - `parseJSON(jsonString)` — Safe JSON parse, returns null on failure
  - `getDateFormat(date, timeZone, format)` — dayjs date formatting with timezone
  - `splitIntoChunks(arr, limit=100)` — Chunk array for batching SQS messages
  - `replaceDateRangeQuery(query, ruleGroups)` — Replace `{{startDate_N}}` / `{{endDate_N}}` placeholders using `dateRangeFilter.numberOfDaysPrevious` from the Nth ruleGroup

### 2. HandleCheckBigQueryJob
- **Path**: `lambda-functions/HandleCheckBigQueryJob/`
- **Handler**: `index.mjs`
- **Trigger**: SQS (`AWS_QUEUE_CHECK_BIG_QUERY_JOB`)
- **Purpose**: Monitors BigQuery job completion for segment creation/rebuild
- **Retry limiting**: Skips processing if `ApproximateReceiveCount > 1` (prevents duplicate processing)
- **Input validation**: `validateInput()` checks required fields (`jobId`, `accountId`, `action`), validates action type against `BQ_JOB_ACTION` enum, parses `jobInputData` as array
- **What it does**:
  1. Receives SQS with `jobId`, `accountId`, `action`, `jobInputData`
  2. Validates input, finds job in DB, skips if job status is not `in-progress`
  3. Checks all BigQuery job statuses via `checkBigQueryJobStatus()` for each `bigQueryJobId` in payload
  4. If not all jobs completed:
     - Tracks `executionCount` (increments per check, 15s per execution)
     - If under max time (5 minutes): re-queues SQS with 15s delay and current `executionCount`
     - If max time exceeded: cancels all remaining PENDING/RUNNING jobs via `cancelBigQueryJob()`
  5. For `CREATE_SEGMENT` / `REBUILD_SEGMENT` actions:
     - Maps BigQuery status to app status (DONE/PENDING/RUNNING/CANCELED/ERROR)
     - For `REBUILD_SEGMENT`: always sets `queryStatus` to DONE (not from BQ status mapping)
     - For `CREATE_SEGMENT`: sets `queryStatus` from BQ status mapping
     - Updates `Segments` table with `totalBytesBilled` (in GB, 0 for INSERT queries), `buildAt`
     - Clears Redis caches: `redis_segments_{accountId}`, `{accountId}_report_segment_{segmentId}_*`, `{accountId}_report_segments_*`
     - Updates `BigQueryJobs` table with status and `bigQueryJobId`
  6. Sends Pusher notification to frontend (`channel-{accountId}`)
  7. **Cascade rebuild**: After successful completion (`queryStatus = DONE`), in parallel (`Promise.all`):
     - For all rebuild segments: clears `segment_rebuild_queue:{segmentId}` Redis key
     - For segments with `isEmbeddedBy` parents: calls `triggerCascadeRebuild()` which makes HTTP POST to `{REACT_APP_API}/internal/segment/cascade-rebuild` with `{ segmentId, cascadeId }` to trigger next level via the main app's `triggerCascadeRebuild`
     - DB update, Pusher, and queue key cleanup are all handled in the lambda — no `handleRebuildComplete` round-trip to the main app
- **Key files**: `models/segments.mjs` (`find`, `findByIds`, `update`), `models/bigqueryJobs.mjs`, `lib/bigquery.mjs`, `pusher.mjs`, `redis/index.mjs`, `helpers.mjs`, `constants/index.mjs`
- **Action types**: `create-segment`, `rebuild-segment`, `query-report`, `query-report-with-segment`

### 3. ReportRequests (Supporting)
- **Path**: `lambda-functions/ReportRequests/`
- **Purpose**: Orchestrates BigQuery queries for reports with segment filtering
- **Segment integration**: Handles `QUERY_REPORT_WITH_SEGMENT` action, creates temporary tables from segment queries, returns `segmentTables` and `comparisonSegmentTables`

### 4. ExportReport (Supporting)
- **Path**: `lambda-functions/ExportReport/`
- **Purpose**: Exports report data to CSV/Excel with segment support
- **Segment integration**: Generates segment-specific column headers (`S1:Name`, `S2:Name`), creates per-segment metric columns

---

## SQS Queues

| Queue | Purpose | Producer | Consumer |
|---|---|---|---|
| `HANDLE_SEGMENT_BIG_QUERY` | Trigger segment cronjob processing and self-queue for level-by-level execution | Scheduler, HandleSegmentBigQuery (self) | HandleSegmentBigQuery |
| `AWS_QUEUE_CHECK_BIG_QUERY_JOB` | Monitor BigQuery job completion | Segment creation/rebuild, ReportRequests | HandleCheckBigQueryJob |
| `SEND_MAIL_SCHEDULE_SAVE_REPORT_QUEUE_URL` | Trigger scheduled report processing | HandleSegmentBigQuery | Schedule report handler |

---

## Client-Side Architecture

### Pages & Components

**Segment Management Page**:
- `client/src/components/cms/subscriber/analytics/data-settings/manager-segment/index.js` - Entry point (`ManageSegment`)
- `client/src/components/cms/subscriber/analytics/data-settings/manager-segment/segment.js` - Main listing with search, create button, default vs custom sections
- `client/src/components/cms/subscriber/analytics/data-settings/manager-segment/ListSegment.js` - `SegmentItem` rows with edit/delete/toggle, pagination (10/page)

**Segment Creation/Editing**:
- `client/src/components/cms/subscriber/analytics/data-settings/manager-segment/SegmentDetail.js` - Multi-step wizard with `SegmentDetailContext`
  - Step 1 (`SegmentRuleStep.js`): Configure rule groups with AND/OR logic
  - Step 2 (`SegmentInfoStep.js`): Name and description (max 200 chars, unique)

**Rule Condition Components** (in `steps/` directory):
| Component | Purpose |
|---|---|
| `CProperty.js` | Property condition builder with section-based property selection |
| `CMetric.js` | Metric condition builder with numeric operators |
| `CEventCondition.js` | Web conversion/ecommerce/sales conversion event conditions |
| `CRelationship.js` | Cross-object relationship count conditions |
| `COtherEvent.js` | Custom event listener conditions |
| `CDateRange.js` | Date range filter (lifetime, previous period, specific range) |
| `CEmbeddedSegment.js` | Checkbox + dropdown to embed another segment into a rule group. Filters out already-embedded, failed, and in-progress segments. Shows loading state and building warnings |

**Segment Usage in Reports**:
- `client/src/components/cms/subscriber/analytics/segments/SegmentDropdown.js` - Segment selector dropdown (All/Default/Custom tabs, search, sticky toggle)
- `client/src/components/cms/subscriber/analytics/segments/AddSegments.js` - Multi-segment management with drag-and-drop reordering (S1, S2, S3...)
- `client/src/components/cms/subscriber/analytics/segment-chart/ChartReportSegment.js` - Chart rendering for segment analysis
- `PopupPreviewInsideSegment.js` - Preview modal showing matching field values

### Redux State Management

**Actions** (`client/src/actions/subscriber.js`):
- `fetchSegmentRequest(accountId)` - Fetch all segments
- `setListSegment(segment)` - Update segments in store
- `setListSegmentReport(segmentReport)` - Set report segment configs
- `setListSegmentReload` / `setListSegmentApplyReload` - Trigger reloads
- `setIsDoneSegment` / `setIsReadyRebuildSegment` - Track operation status

**Action Types** (`client/src/actions/types.js`):
`SET_LIST_SEGMENT`, `SET_LIST_SEGMENT_REPORT`, `SET_LIST_SEGMENT_APPLY_RELOAD`, `SET_LIST_SEGMENT_RELOAD`, `SET_LIST_SEGMENT_DB`, `SET_DB_SEGMENT_SELECTED`, `SET_IS_LOADING_ADD_SEGMENT_CHART`, `SET_IS_LOADING_ADD_SEGMENT_METRIC`, `SET_IS_DONE_SEGMENT`, `SET_IS_READY_REBUILD_SEGMENT`, `PUSHER_UPDATE_SEGMENT`

**Reducer** (`client/src/reducers/subscriber.js`): Manages listSegment, listSegmentReport, listSegmentDB, segment job statuses (in-progress, done, failed, canceled), totalBytesBilled, buildAt, queryStatus.

### Client API Endpoints

| Constant | Path | Method | Purpose |
|---|---|---|---|
| `API_CLIENT_GET_LIST_SEGMENT` | `client/segments/:accountId` | GET | Fetch all segments |
| `API_CLIENT_SEGMENT_DETAil` | `client/segment/:id` | GET | Get segment detail |
| `API_CLIENT_SEGMENT` | `client/segment` | POST/PUT/DELETE | CRUD operations |
| `API_CLIENT_REBUILD_SEGMENT` | `client/rebuild-segment` | POST | Rebuild segment |
| `API_CLIENT_SEGMENT_REPORT` | `client/segment-report` | PUT/DELETE | Manage report configs |
| `API_CLIENT_SEGMENT_CHART` | `client/segment-chart` | PUT | Apply to charts |
| `API_CLIENT_EXPIRE_ALL_SEGMENT` | `client/segment/expire/:accountId` | PUT | Expire all segments |
| `API_CLIENT_GET_SEGMENT_OPTION` | `client/segment/option` | POST | Get filter options |
| `API_CLIENT_PREVIEW_INSIDE_SEGMENT` | `client/preview-inside-segment` | POST | Preview condition data |
| `API_CLIENT_GET_AVAILABLE_SEGMENTS_FOR_EMBEDDING` | `client/segments/:accountId/available-for-embedding/:segmentId` | GET | Fetch segments available to embed |

### Client Constants
**File**: `client/src/constants/segment.js`

Contains all `SEGMENT_OBJECT`, `CONDITION_FOCUS`, `SEGMENT_FIELD_TYPE`, `SEGMENT_CONDITION_TYPE` enums, condition operators, date range options, section categories for property grouping, and comprehensive `PROPERTY_METRIC_SEGMENT_OPTIONS` mappings for each object type.

---

## End-to-End Flows

### Segment Creation Flow

```
[User clicks "CREATE A NEW Segment"]
       |
       v
SegmentDetail modal opens (SegmentDetailContext)
       |
       v
Step 1: SegmentRuleStep
  - Select segment object (Session, User, People, Company, etc.)
  - Add rule groups with AND/OR conjunction
  - For each rule: select conditionFocus -> configure conditions
  - Preview field values via PopupPreviewInsideSegment
       |
       v
Step 2: SegmentInfoStep
  - Enter name (required, max 200 chars, unique)
  - Enter description (optional)
  - Click Save
       |
       v
POST /client/segment
       |
       v
Service: handleCreateSegment()
  - Generate UUID
  - Extract embedded segment IDs from ruleGroups
  - Validate embedded segments (circular deps, duplicates, self-embed)
  - Call getSegmentData() to build:
    - createTableQuery (CREATE OR REPLACE TABLE)
    - cronjobQuery (INSERT INTO for incremental updates)
  - Queue BigQuery job via QueueService.sendQueueCreateBigQueryJob()
  - Serialize ruleGroups, insert into Segments table
  - Sync isEmbeddedBy tracking on embedded segments
  - Clear Redis cache (REDIS_KEYS.SEGMENTS)
       |
       v
SQS: AWS_QUEUE_CHECK_BIG_QUERY_JOB
       |
       v
HandleCheckBigQueryJob Lambda
  - Poll BigQuery API every 15s (max 5 min)
  - On completion: update Segments table, clear Redis, send Pusher notification
       |
       v
[Frontend receives Pusher event, updates UI with queryStatus: 'done']
```

### Segment Application to Reports Flow

```
[User opens report -> clicks Add Segment]
       |
       v
SegmentDropdown opens (All/Default/Custom tabs)
  - Search and select segments (max 4)
       |
       v
PUT /client/segment-report
       |
       v
Service: handleUpdateSegmentReport()
  - Create/update SegmentReport records
  - Enforce max 4 concurrent segments
  - Handle sticky behavior across reports
       |
       v
[Report query includes segment CTEs as filters]
  - getSegmentData() returns cteQuery + filterCondition
  - Report query JOINs segment table to filter results
```

### Segment Cronjob (Incremental Update) Flow

```
[Scheduled SQS trigger → HANDLE_SEGMENT_BIG_QUERY queue]
       |
       v
HandleSegmentBigQuery Lambda (Initial Invocation)
  - Query PostgreSQL for active segments (isBlocked=false, queryStatus=done, <10GB)
  - getProcessingOrder() → topological sort into dependency levels
  - processNextLevel(level=0):
    - Filter out segments with failed dependencies
    - For each segment: replace date placeholders, submit BigQuery batch job
    - Last level: submit and return (no wait)
    - Other levels: self-queue SQS with 180s delay
       |
       v (SQS self-trigger after 180s)
HandleSegmentBigQuery Lambda (Continuation Invocation)
  - checkPendingJobs() → check BigQuery job statuses
  - If still running & checkCount < 10: re-queue with checkCount++
  - If all done or checkCount >= 10: processNextLevel(level+1)
  - Repeat until all levels processed
       |
       v
processSSTriggers()
  - Read S3 trigger rules, check schedule
  - If targets exist & hour < 12: send SQS to schedule report queue
```

---

## Redis Caching

| Key Pattern | Purpose | TTL |
|---|---|---|
| `redis_segments_{accountId}` | All segments for account | 1 hour |
| `{accountId}_report_segment_{segmentId}_{hash}` | Segment query results | Cleared on rebuild |
| `{accountId}_report_segments_*` | Report segment caches | Cleared on segment change |
| `REDIS_KEYS.SEGMENT_INSIDE_PREVIEW` | Preview data | 1 hour |
| Segment option caches | Filter options per variable type | 1 hour |
| `segment_cascade_visited:{cascadeId}` | Visited set for cascade rebuild cycle protection | 1 hour |
| `segment_rebuild_queue:{segmentId}` | Queue metadata for pending rebuilds (dedup). Cleared on rebuild complete so segment can rebuild again on next trigger | 1 hour |

---

## Segment Lifecycle

1. **Creation**: User defines rules -> validate embedded segments -> BigQuery table created -> sync `isEmbeddedBy` -> queryStatus: in-progress -> done
2. **Caching**: All segments cached in Redis per account (1-hour TTL)
3. **Updating**: Rules modified -> validate embedded segments -> BigQuery table recreated -> sync `isEmbeddedBy` (only if changed) -> cache cleared
4. **Application**: Applied to reports (max 4) -> used as CTE filters in report queries
5. **Incremental Update**: Cronjob queries INSERT new matching data into existing tables (processed in topological order for embedded dependencies)
6. **Preview**: User previews conditions -> BigQuery queried for sample/popular/unique values
7. **Expiration/Rebuild**: Full refresh via expire or rebuild -> drops and recreates table -> cascade rebuild triggers parent segments
8. **Deletion**: Block if embedded by others -> clean up `isEmbeddedBy` on child segments -> DB record deleted -> BigQuery table dropped -> cache cleared

---

## Embed Segment Inside Another

Segments can be composed by embedding one segment into another segment's rule group. This narrows the rule group's results to only records that also belong to the embedded segment.

### How It Works

Each rule group can optionally have an `embeddedSegmentId` field referencing another segment. The query builder adds a subquery filter:
- **With regular conditions**: `WHERE ... AND objectId IN (SELECT objectId FROM segment_{embeddedId})`
- **Without conditions (embed only)**: `SELECT DISTINCT objectId FROM segment_{embeddedId})`

### Bidirectional Tracking

| Direction | Storage | Purpose |
|---|---|---|
| Parent → Child | `ruleGroups[].embeddedSegmentId` in parent's ruleGroups JSON | Query generation — parent knows which segment tables to reference |
| Child → Parents | `isEmbeddedBy` JSONB array on child segment | Cascade rebuilds — child knows which parents to notify on change. Deletion protection — prevents deleting embedded segments |

### Safety Mechanisms

| Mechanism | Where | How |
|---|---|---|
| Circular dependency prevention | Service: `checkCircularDependency()` | BFS traversal through embedding chain |
| Self-embedding prevention | Service: `validateEmbeddedSegments()` | Checks `embeddedSegmentIds.includes(parentId)` |
| Duplicate embedding prevention | Client: `alreadyEmbeddedIds` + Service validation | Tracks all embedded IDs across rule groups |
| Deletion protection | Service: `handleRemoveSegment()` | Checks `isEmbeddedBy` array, returns parent names in error |
| Dependency ordering | Lambda: `dependencyGraph.js` | Topological sort ensures embedded segments build before parents in cronjobs |
| Cascade rebuild | Service: `segmentSync.js` | When embedded segment changes, all parents auto-rebuild |
| Cycle protection in cascade | Service: `triggerCascadeRebuild()` | Redis visited set (1hr TTL) per cascade chain |
| Queue dedup | Service: `queueSegmentRebuild()` | Redis key prevents double-queuing same segment. Cleared by lambda on rebuild complete |

### Cascade Rebuild Flow

```
Segment B (embedded) is updated/rebuilt
       |
       v
triggerCascadeRebuild(segmentB.id)  [main app]
  - Check Redis visited set for cycle protection
  - Read segmentB.isEmbeddedBy -> [segmentA.id, segmentC.id]
       |
       v
For each parent (in parallel via Promise.all):
  queueSegmentRebuild(parentId)  [main app]
    - Set queryStatus to IN_PROGRESS + Pusher notification
    - Build new BigQuery table
    - Queue HandleCheckBigQueryJob
       |
       v
HandleCheckBigQueryJob Lambda completes:
  - Set queryStatus to DONE in DB
  - Send Pusher notification
  - Clear segment_rebuild_queue:{segmentId}
  - If segment has parents:
    - HTTP POST to main app /internal/segment/cascade-rebuild
    - triggerCascadeRebuild(parentId) -> rebuilds grandparents
```

### Client-Side Flow

```
SegmentRuleStep
  |-- State: availableSegmentsForEmbed, isLoadingEmbedSegments, alreadyEmbeddedIds
  |-- Fetches: GET /client/segments/:accountId/available-for-embedding/:segmentId
  |-- Syncs queryStatus from Redux listSegment to available segments
  |-- Validates: no self-embed, no duplicates, selection required if toggled on
  |
  +-- CEmbeddedSegment (per rule group)
       |-- Checkbox: "Embed another segment into this rule group"
       |-- Dropdown: Select segment (filters out already-embedded, failed, in-progress)
       |-- Warning: Shows if selected segment is still building
```

---

## Debugging Guide

### Segment not building (stuck in-progress)
1. **Check BigQuery job status**: Look at `HandleCheckBigQueryJob` logs for the segment's job
2. **Check SQS delivery**: Verify `AWS_QUEUE_CHECK_BIG_QUERY_JOB` received the message
3. **Check Pusher**: Verify notification was sent to `channel-{accountId}`
4. **Check queryStatus in DB**: Query `Segments` table for the segment's `queryStatus`, `totalBytesBilled`
5. **Files**: `HandleCheckBigQueryJob/index.mjs`, `server/services/segments.js`

### Segment query failed
1. **Check query syntax**: Use `POST /client/segment-testing/` to validate the segment query
2. **Check BigQuery errors**: Look at `HandleCheckBigQueryJob` logs for ERROR status
3. **Check rule groups**: Deserialize `ruleGroups` from DB and verify conditions are valid
4. **Files**: `server/services/segmentTesting.js`, `server/models/query-builders/common/segment/`

### Segment data not appearing in reports
1. **Check SegmentReport table**: Verify `apply = true` for the segment/report combination
2. **Check max segments**: Only 4 segments can be applied per report simultaneously
3. **Check segment status**: Verify `status = true` (enabled) and `queryStatus = 'done'`
4. **Check Redis cache**: Stale cache may need clearing (`redis_segments_{accountId}`)

### Segment bytes billed too high (>10 GB)
1. **Check totalBytesBilled**: `Segments` table, `totalBytesBilled` column
2. **Threshold**: `SEGMENT_BYTES_BILLED_THRESHOLD = 10` GB
3. **Impact**: Segments exceeding threshold are excluded from cronjob processing
4. **Resolution**: Rebuild segment or modify rules to reduce data scanned
5. **Files**: `HandleSegmentBigQuery/constants/index.js`, `server/services/segments.js`

### Segment options not loading
1. **Check Redis cache**: Options are cached for 1 hour per variable type
2. **Check variable type**: Service fetches different data per `SEGMENT_VARIABLE_OPTION` type
3. **Files**: `server/services/segments.js` -> `handleGetOptionDataByAccount()`

### Segment preview showing no data
1. **Check BigQuery tables**: Ensure the account's analytics tables have data
2. **Check Redis cache**: Preview results cached for 1 hour
3. **Files**: `server/services/segments.js` -> `handlePreviewInsideSegment()`

### Embedded segment issues
1. **Cannot delete segment**: Check `isEmbeddedBy` column — segment is embedded by other segments. Remove the embedding from parent segments first.
2. **Circular dependency error on save**: `checkCircularDependency()` BFS detected a cycle. Check the chain: A embeds B embeds C embeds A.
3. **Cascade rebuild not triggering**: Check `segmentSync.js` logs (filter by `[CascadeRebuild]` prefix). Verify `isEmbeddedBy` array is populated. Check Redis queue key (`segment_rebuild_queue:{segmentId}`) and visited set (`segment_cascade_visited:{cascadeId}`).
4. **Segments building in wrong order (cronjob)**: Check `dependencyGraph.js` topological sort. Verify `getProcessingOrder()` returns correct levels. Look for cycle fallback in logs.
5. **Available segments list empty**: `handleGetAvailableSegmentsForEmbedding()` filters out non-custom, self, and circular-dependency-causing segments. Verify account has other custom segments with `queryStatus = 'done'`.
6. **Files**: `server/services/segments.js`, `server/services/segmentSync.js`, `HandleSegmentBigQuery/utils/dependencyGraph.js`

### Real-time update not reaching frontend
1. **Check Pusher**: `HandleCheckBigQueryJob` sends to `channel-{accountId}` with event `PUSHER_UPDATE_SEGMENT`
2. **Check Redux reducer**: `PUSHER_UPDATE_SEGMENT` action should update segment in store
3. **Files**: `HandleCheckBigQueryJob/pusher.mjs`, `client/src/reducers/subscriber.js`

---

## Key Constants File

**File**: `server/constants/segment.js`

Contains all enums: `SEGMENT_OBJECT`, `CONDITION_FOCUS`, `SEGMENT_FIELD_TYPE`, `DEFAULT_SEGMENT_RULE`, `SEGMENT_VARIABLE_OPTION`, and mapping objects (`MAPPING_SEGMENT_OBJECT_TYPE`, `MAPPING_SEGMENT_OBJECT_ID`, `MAPPING_CRM_OBJECT_TABLE`, `MAPPING_SF_OBJECT_TABLE`, `MAPPING_SALES_CONVERSION_OBJECT_TYPE`, `MAPPING_SALES_CONVERSION_OBJECT_ID`).

---

## Key Environment Variables

- `REDIS_URL` - Redis connection for caching
- `POSTGRES_HOST`, `POSTGRES_USERNAME`, `POSTGRES_PASSWORD` - PostgreSQL connection
- `HANDLE_SEGMENT_BIG_QUERY` - SQS queue for segment cronjob processing (also used for self-queuing)
- `AWS_QUEUE_CHECK_BIG_QUERY_JOB` - SQS queue for BigQuery job monitoring
- `SEND_MAIL_SCHEDULE_SAVE_REPORT_QUEUE_URL` - SQS queue for scheduled reports
- `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT` - BigQuery authentication
- `PUSHER_APP_ID`, `PUSHER_KEY`, `PUSHER_SECRET`, `PUSHER_CLUSTER` - Pusher real-time notifications
- `S3_BUCKET_STATIC` - S3 bucket for server-side trigger configs
