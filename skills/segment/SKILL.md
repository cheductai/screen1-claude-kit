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
| Company | `revealed-company` | Revealed companies from IP/domain |
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
| `handleCreateSegment` | Generate UUID, build BigQuery CREATE TABLE query via `getSegmentData()`, queue BigQuery job, serialize ruleGroups, insert DB record, clear Redis cache |
| `handleUpdateSegment` | Update name/description/status/queryStatus, regenerate BigQuery table if queryStatus is IN_PROGRESS, clear caches, update related report views |
| `handleRemoveSegment` | Delete from DB, drop BigQuery table, clear cache |
| `handleGetSegmentById` | Fetch by ID, deserialize ruleGroups, strip query field |
| `handleGetAllSegmentByAccount` | Redis-cached retrieval, fallback to DB, deserialize all ruleGroups |
| `handleGetOptionDataByAccount` | Fetch filter options (CHECKLIST, OTHER_FIELD, PIPELINE_STATUS, INTENT, METRIC, CONVERSION, ECOMMERCE, EVENT_NAME, EVENT_VARIABLE, SALES_CONVERSION), 1-hour Redis cache |
| `handlePreviewInsideSegment` | Query BigQuery for popularValue, sampleValue, uniqueValue; 1-hour Redis cache |
| `handleExpireAllSegmentsByAccount` | Refresh all custom segments, rebuild cronjob queries, optionally drop BigQuery tables |
| `handleRebuildSegment` | Rebuild specific segment, set queryStatus to IN_PROGRESS, queue new BigQuery job |
| `handleUpdateSegmentReport` | Link/unlink segments to reports, max 4 concurrent segments per report, handle sticky behavior |
| `handleApplySegmentForChart` | Apply selected segments to chart visualizations |
| `scriptCreateDefaultSegment` | Create default "All Data" segment per account |

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
| `getSegmentQuery(data)` | Builds complete segment BigQuery table with CTEs per rule group. Combines rules with AND/OR. Handles both LL and CRM objects. |
| `getRuleQuery(data)` | Routes individual rule to appropriate builder by conditionFocus. Returns SELECT DISTINCT objectId with CTE conditions. |
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
| `relationship.js` | `getObjectDataByRelationship` | RELATIONSHIP conditions (cross-object count filters via crmRelation table) |
| `otherEvent.js` | `getObjectDataByOtherEvent` | OTHER_EVENT conditions (custom event variables, listener data) |
| `segmentUtils.js` | Utility functions | Date range filters, field aliases, UNNEST subqueries, table name resolution, property/metric filters, checklist filters |

### Query Generation Flow

```
getSegmentData({ segment, accountId, accountTimeZone })
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
- **Trigger**: SQS (`SEND_MAIL_SCHEDULE_SAVE_REPORT_QUEUE_URL`)
- **Purpose**: Processes segment cronjob queries for all accounts
- **What it does**:
  1. Receives SQS message with `accountId` and `timeZone`
  2. Reads server-side trigger rules from S3 (`server-side-trigger/{accountId}.json`)
  3. Queries PostgreSQL for active, non-blocked segments for the account
  4. For each segment: replaces date placeholders, creates BigQuery batch job
  5. If People/Company targets exist and hour < 12, sends SQS to schedule report queue
- **Filters**: Only segments where `isBlocked = false`, `queryStatus = 'done'`, `query IS NOT NULL`, `totalBytesBilled < 10 GB`
- **Key files**: `models/segments.js`, `lib/bigquery.js`, `s3/readFileStaticJson.js`, `helpers.js`

### 2. HandleCheckBigQueryJob
- **Path**: `lambda-functions/HandleCheckBigQueryJob/`
- **Handler**: `index.mjs`
- **Trigger**: SQS (`AWS_QUEUE_CHECK_BIG_QUERY_JOB`)
- **Purpose**: Monitors BigQuery job completion for segment creation/rebuild
- **What it does**:
  1. Receives SQS with `jobId`, `accountId`, `action`, `jobInputData`
  2. Checks BigQuery API for job status every 15 seconds (max 5 minutes)
  3. For `CREATE_SEGMENT` / `REBUILD_SEGMENT` actions:
     - Maps BigQuery status to app status (DONE/PENDING/RUNNING/CANCELED/ERROR)
     - Updates `Segments` table with `totalBytesBilled`, `buildAt`, `queryStatus`
     - Clears Redis caches: `redis_segments_{accountId}`, `{accountId}_report_segment_{segmentId}_*`, `{accountId}_report_segments_*`
     - Sends Pusher notification to frontend (`channel-{accountId}`)
  4. If still running: re-queues SQS with 15-second delay
  5. If timeout: cancels remaining jobs, updates status
- **Key files**: `models/segments.mjs`, `models/bigqueryJobs.mjs`, `lib/bigquery.mjs`, `pusher.mjs`, `redis/index.mjs`
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
  - Call getSegmentData() to build:
    - createTableQuery (CREATE OR REPLACE TABLE)
    - cronjobQuery (INSERT INTO for incremental updates)
  - Queue BigQuery job via QueueService.sendQueueCreateBigQueryJob()
  - Serialize ruleGroups, insert into Segments table
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
[Scheduled trigger]
       |
       v
HandleSegmentBigQuery Lambda
  - Query PostgreSQL for active segments (isBlocked=false, queryStatus=done, <10GB)
  - For each segment: replace date placeholders in cronjob query
  - Submit BigQuery batch jobs (INSERT INTO segment_{id})
       |
       v
[BigQuery processes incremental data insertion]
       |
       v
HandleCheckBigQueryJob Lambda (if triggered)
  - Track completion, update status
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

---

## Segment Lifecycle

1. **Creation**: User defines rules -> BigQuery table created -> queryStatus: in-progress -> done
2. **Caching**: All segments cached in Redis per account (1-hour TTL)
3. **Updating**: Rules modified -> BigQuery table recreated -> cache cleared
4. **Application**: Applied to reports (max 4) -> used as CTE filters in report queries
5. **Incremental Update**: Cronjob queries INSERT new matching data into existing tables
6. **Preview**: User previews conditions -> BigQuery queried for sample/popular/unique values
7. **Expiration/Rebuild**: Full refresh via expire or rebuild -> drops and recreates table
8. **Deletion**: DB record deleted -> BigQuery table dropped -> cache cleared

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
- `AWS_QUEUE_CHECK_BIG_QUERY_JOB` - SQS queue for BigQuery job monitoring
- `SEND_MAIL_SCHEDULE_SAVE_REPORT_QUEUE_URL` - SQS queue for scheduled reports
- `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_PROJECT` - BigQuery authentication
- `PUSHER_APP_ID`, `PUSHER_KEY`, `PUSHER_SECRET`, `PUSHER_CLUSTER` - Pusher real-time notifications
- `S3_BUCKET_STATIC` - S3 bucket for server-side trigger configs
