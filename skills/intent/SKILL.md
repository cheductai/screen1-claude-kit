---
name: intent
description: Search and explain intent-related lambdas, API endpoints, data models, LLM integration, datasets, properties, trigger types (preview, rerun, server-side), prompt building, and debugging across datalayerapp-v2 and datalayerapp-v2-function-code repositories.
---

# Intent Skill

This skill helps you understand, debug, and navigate intent-related code across the Datalayer platform. Intents are AI-powered business rules that use LLM models to classify, summarize, or generate structured data for People and Company records.

## Invocation

When the user asks about intents, use this knowledge base to:
1. Identify which files, lambdas, and services are relevant
2. Explain the end-to-end intent creation, execution, and review flow
3. Help debug intent-related issues by pointing to the right logs, tables, queues, and code
4. Explain trigger types (preview, rerun, server-side trigger, manual run) and their differences
5. Explain prompt building, dataset usage, and LLM integration

## Repositories

| Repository | Path | Role |
|---|---|---|
| **datalayerapp-v2** | `D:\Projects\Datalayer\datalayerapp-v2` | Main Express.js app (API, models, services, constants, queues, client UI) |
| **datalayerapp-v2-function-code** | `D:\Projects\Datalayer\datalayerapp-v2-function-code` | AWS Lambda functions for intent processing and property value storage |

---

## What Are Intents?

Intents are AI-powered rules that automatically process data and populate custom properties using LLM capabilities. Each intent rule:

- Targets a specific **object type** (People or Company)
- Uses an **LLM model** (OpenAI, Claude, xAI, etc.) to analyze record data
- Applies a **dataset** (classification categories or context) to guide AI output
- Populates an **IntentProperty** with the AI-generated result
- Can be triggered automatically via **server-side triggers** or manually via **preview/rerun**

**Analysis Types:**
- **Classification** — Categorize records into dataset-defined categories
- **Summary** — Generate free-text summaries or insights

---

## Lambda Functions (datalayerapp-v2-function-code)

### 1. IntentHandleProcess
- **Path**: `lambda-functions/IntentHandleProcess/`
- **Handler**: `index.mjs`
- **Purpose**: Main orchestrator for intent rule processing — handles both SQS trigger events AND DynamoDB stream events
- **Dual Input Routing**: The handler inspects the event to determine the source:
  - `event.Records[0].dynamodb` → DynamoDB stream event → routes to `handleDynamodbStreamEvent()`
  - Otherwise → SQS/direct invocation → normal rule processing flow

#### Normal Flow (SQS / Direct Invocation)
1. Receives event with `accountId`, `triggerRecordIds`, `triggerType`, `redisKeys`
2. Retrieves active intent rules linked to triggers
3. Deduplicates via BigQuery `TriggerObjectExistence` table
4. Fetches datasets and LLM model configuration
5. Builds prompts for each object using `build_prompt.mjs`
6. **Priority property check**: If rule has `propertiesData` with `section === 'Intent Properties'`, creates an `IntentRequestLog` record instead of immediately queuing the LLM request
7. For rules without priority properties, queues LLM requests to SQS directly
8. Tracks execution in BigQuery `TriggerObjectExistence` table

#### IntentRequestLogs & DynamoDB Stream Flow
When a rule depends on **priority properties** (other intent properties that must be filled first), the Lambda uses IntentRequestLogs with DynamoDB Streams to coordinate:

**Record Creation** (normal flow):
```javascript
{
  id: "{objectId}_{ruleId}",           // Composite key
  ruleId, objectId, accountId,
  triggerLogId: intentTriggerLog.id,
  messageData: JSON.stringify({ ...initMessage, rule, dataset, record }),
  priorityPropertyIds: JSON.stringify(priorityPropertyIds),  // Properties that must be filled
  createdAt, expiresAt                  // 30-day TTL
}
```
- `priorityPropertyIds` — Array of property IDs (from `propertiesData` where `section === 'Intent Properties'`) that must ALL have values before this rule can execute
- `messageData` — Full context (rule config, dataset, record data) needed to build the LLM prompt later
- Records are batch-written via `dynamoHandler.batchPutItems()`

**MODIFY Stream Event** (triggered when IntentHandlePropertyValues updates a property on the log):
1. Receives the modified IntentRequestLog record
2. Checks if ALL `priorityPropertyIds` now have values: `priorityPropertyIds.every(id => updateItem[id])`
3. If all properties are filled → deletes the record via `dynamoHandler.deleteIntentRequestLog(id)`
4. If not yet complete → no action (waits for more property updates)

**REMOVE Stream Event** (triggered by the deletion above, or by TTL expiry):
1. Extracts `messageData` from `OldImage` (the deleted record)
2. Verifies all priority properties have values
3. If verified:
   - Reconstructs the record with all populated property values
   - Builds final `userPrompt` using `buildIntentPrompt()` with the complete record
   - Sends to `HANDLE_REQUEST_LLM_QUEUE_URL` via SQS for LLM processing

**How Properties Get Updated on the Log:**
- `IntentHandlePropertyValues` Lambda processes an LLM result for a different rule targeting the same object
- It queries IntentRequestLogs by `objectId` (via GSI)
- For each matching log, checks if the completed property is in its `priorityPropertyIds`
- If yes, calls `dynamoHandler.updateIntentTriggerLog(log.id, propertyId, value)` — adding a new attribute to the record
- This UPDATE triggers the MODIFY stream event back to IntentHandleProcess

- **Batching**: Processes up to 500 records per invocation; sends SQS to itself for continuation
- **Concurrency**: Breaks rules into max 5 concurrent chunks with 60-second delays
- **Key files**: `index.mjs`, `utils/build_prompt.mjs`, `model/triggerObjectExistence.mjs`, `model/db_handler.mjs`, `model/dynamo_handler.mjs`
- **Triggered by**: SQS from server-side triggers, manual runs, or reruns; **DynamoDB Streams** from IntentRequestLogs table (MODIFY and REMOVE events)

### 2. IntentHandlePropertyValues
- **Path**: `lambda-functions/IntentHandlePropertyValues/`
- **Handler**: `index.mjs`
- **Purpose**: Store LLM results into IntentPropertyValues
- **What it does**:
  1. Receives LLM result with `accountId`, `ruleId`, `objectId`, `llmResult`, `llmTrackingRequestId`
  2. Fetches intent rule to get property config
  3. Stores/updates `IntentPropertyValues` based on `storeValue` (single or multiple)
  4. Applies `existingValues` logic (BLOCK, APPEND, OVERRIDE)
  5. Updates priority property tracking in DynamoDB
  6. Syncs to BigQuery (People/Companies tables)
  7. Updates `LLMTrackingRequest` with `referenceId`
- **Key files**: `index.mjs`, `model/db_handler.mjs`, `model/dynamo_handler.mjs`
- **Triggered by**: SQS from LLM result processing

---

## API Endpoints (datalayerapp-v2)

**Route File**: `server/routes/intentRules.js`

### Intent Rules CRUD
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/client/intent-rules` | Create intent rule |
| GET | `/client/intent-rules/:id` | Get rule by ID |
| PUT | `/client/intent-rules/:id` | Update rule |
| DELETE | `/client/intent-rules/:id` | Delete rule |
| GET | `/client/intent-rules/account/:accountId/paginated` | List rules (paginated) |
| GET | `/client/intent-rules/account/:accountId/object/:object/field-options` | Get available fields for object type |

### Preview & Analysis
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/client/intent-rules/sample-data` | Get sample data for preview |
| POST | `/client/intent-rules/run-analysis` | Run preview analysis (MANUAL_PREVIEW) |

### Review & History
| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/client/intent-rules/review/account/:accountId/paginated` | List execution history |
| GET | `/client/intent-rules/review/account/:accountId/most-recent-insight` | Get recent runs/averages |
| GET | `/client/intent-rules/review/:id` | Get detailed review for rule |
| GET | `/client/intent-rules/review/:id/server-side-triggers` | Get SST execution results |

### Rerun
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/client/intent/rerun-property` | Rerun intent on specific records (RERUN_INTENT) |

### Properties, Datasets & Calculator
| Method | Endpoint | Purpose |
|--------|----------|---------|
| * | `/client/intent/properties` | Manage intent properties |
| * | `/client/intent/datasets` | Manage classification datasets |
| * | `/client/intent/dataset-rule` | Manage dataset rules |
| GET | `/intent/calculator` | Trigger daily intent calculator |

**Controller**: `server/controllers/intentRulesController.js`
**Service**: `server/services/intentRules.js`

---

## Server-Side Services

### intentRules.js
**File**: `server/services/intentRules.js`

| Function | Purpose |
|----------|---------|
| `handleCreateIntentRule` | Create new intent rule with validation |
| `handleGetIntentRule` | Fetch rule by ID with related data |
| `handleUpdateIntentRule` | Update rule configuration |
| `handleDeleteIntentRule` | Delete rule and clean up |
| `handleRunAnalysis` | Run preview analysis (lines 969-1014) |
| `handleRerunIntentProperty` | Rerun on specific records (lines 594-760) |
| `handleGetSampleData` | Fetch sample records from BigQuery |
| `handleGetIntentReview` | Get execution review data |
| `handleGetIntentReviewServerSideTriggers` | Get SST execution results |
| `buildIntentPrompt` | Build LLM prompt from rule config |

### Prompt Building
**File**: `lambda-functions/IntentHandleProcess/utils/build_prompt.mjs`

Prompt sections (in order):
1. **Main Instructions** — Always included, core task description
2. **Dataset** — Classification list or context data
3. **Object Data** — Formatted JSON of People/Company fields
4. **Business Description** — Optional, can be excluded/customized
5. **Customer Profile** — Optional, can be excluded/customized
6. **Example Outputs** — Optional, can be excluded/customized
7. **Rules/Filters** — Output constraints and formatting rules

### LLM Queue Processing
**File**: `server/queues/queueLLMRequest.js`

- Consumes from RabbitMQ queue `QueueLLMRequest`
- Calls LLM provider API (OpenAI, Claude, xAI, etc.)
- Stores results in `LLMTrackingRequest` table
- Triggers `IntentHandlePropertyValues` via SQS

---

## Trigger Types

```javascript
// server/constants/intent.js
AI_INTENT_TRIGGER_TYPE = {
    SERVER_SIDE_TRIGGER: 'server_side_trigger',
    MANUAL_PREVIEW:      'manual_preview',
    MANUAL_RUN:          'manual_run',
    RERUN_INTENT:        'rerun_intent',
}

// Chargeable types (cost user credits)
AI_INTENT_TRIGGER_TYPES_TO_CHARGE = [
    'server_side_trigger',
    'rerun_intent'
]
```

### Server-Side Trigger
- **Automatic execution** when a ServerSideTrigger fires on People/Company record changes
- `IntentRules.triggers` is a JSONB array of ServerSideTrigger IDs
- Flow: SST fires → `serverSideTrigger.js` calls `isRuleIdInIntentRule(ruleId)` (line ~2846) → sends `triggerRecordIds` to `IntentHandleProcess` Lambda via SQS
- Deduplicates via BigQuery `TriggerObjectExistence` table
- Results stored in DB and synced to BigQuery
- SSTs linked to intent rules **cannot be deleted** (validated in model)
- **Charged**: Yes

### Manual Preview
- **Test run** on sample records; results are **NOT** persisted to database
- Endpoint: `POST /client/intent-rules/run-analysis`
- Service: `handleRunAnalysis` in `server/services/intentRules.js`
- Stores queue in Redis: `INTENT_PREVIEW_LLM_QUEUE_{accountId}_{uuid}`
- Results sent via **Pusher** WebSocket to client
- **Charged**: No (unless `isPreviewCharged: true`)

### Rerun Intent
- **Re-execute** intent rules on specific records after initial execution
- Endpoint: `POST /client/intent/rerun-property`
- Service: `handleRerunIntentProperty` in `server/services/intentRules.js`
- Accepts `{ accountId, accountTimeZone, objectProperties: [{propertyId, objectId}] }`
- Applies **existingValues** policy before execution
- Stores queue in Redis: `INTENT_RERUN_QUEUE_{accountId}_{uuid}`
- **Charged**: Yes

### Manual Run
- **One-time full execution** on all matching records
- Similar to rerun but targets all records rather than specific ones
- **Charged**: Depends on configuration

---

## Existing Values Policy

Controls behavior when re-running on records that already have property values:

| Policy | Constant | Behavior |
|--------|----------|----------|
| **Block** | `BLOCK` | Skip record if property already has a value |
| **Override** | `OVERRIDE` | Replace existing values with new result |
| **Append** | `APPEND` | Add new values to existing (no duplicates) |

---

## Database Tables

### PostgreSQL Tables

**IntentRules** (main configuration)
- `id` (uuid PK), `accountId`, `name`, `description`, `type` (classification/summary)
- `object` ('people'/'company'), `propertyId`, `llmModelId`, `datasetId`
- `storeValue` ('one-value'/'multiple-values')
- `propertiesData` (JSONB — selected fields with metadata)
- `metricData` (JSONB — engagement/conversion metrics)
- `triggers` (JSONB — array of ServerSideTrigger IDs)
- `existingValues` ('block'/'append'/'override')
- `mainInstructions`, `businessDescription`, `customerProfile`, `exampleOutputs`, `rulesAndFilters` (text)
- `isMainInstructionsCustom`, `isBusinessDescriptionCustom`, etc. (boolean)
- `metricLookback` ('previous_week', 'lifetime_metrics', etc.)
- `status` ('draft'/'active'/'inactive')
- `created_at`, `updated_at`

**IntentProperties** (custom property definitions)
- `id` (uuid PK), `accountId`, `object`, `fieldType`, `name`, `description`

**IntentPropertyValues** (computed LLM results per record)
- `id` (uuid PK), `accountId`, `propertyId`, `objectId`
- `values` (JSONB — array of `{ruleId, value}`)

**IntentDatasets** (classification datasets)
- `id` (uuid PK), `accountId`, `name`, `description`, `DatasetRuleId`, `appleAI` (JSONB), `isDefault`

**IntentDatasetRules** (dataset items)
- `id` (uuid PK), `accountId`, `name`, `description`
- `data` (JSONB — classification items with descriptions)
- `oldData` (JSONB — version history)

**LLMTrackingRequest** (execution log)
- `id` (serial PK), `accountId`, `llmModelId`, `runAt`, `status` ('pending'/'completed'/'failed')
- `prompt` (text), `result` (text), `inputTokens`, `outputTokens`, `totalCost`
- `triggerType` ('server_side_trigger'/'manual_preview'/'manual_run'/'rerun_intent')
- `sourceId` (intent rule ID), `objectId` (people/company ID)
- `referenceId` (links to IntentPropertyValues)
- `numberOfRetries`, `csvData`, `isPreviewCharged`

### DynamoDB Tables

**IntentRequestLogs** (`dynamodb_table_intent_request_logs`)
- **PK**: `id` = `{objectId}_{ruleId}` (composite key ensuring uniqueness per rule per object)
- **GSI**: `objectId` (used by IntentHandlePropertyValues to find logs for a given object)
- **DynamoDB Streams**: Enabled — MODIFY and REMOVE events trigger IntentHandleProcess Lambda
- **Fields**:
  - `accountId`, `ruleId`, `objectId`, `triggerLogId`
  - `messageData` — Stringified JSON containing full rule config, dataset, and record data (deleted from logs before CloudWatch output)
  - `priorityPropertyIds` — Stringified JSON array of property IDs that must all be filled
  - `{propertyId}` — Dynamic attributes added by IntentHandlePropertyValues as each priority property is filled
  - `createdAt`, `expiresAt` (30-day TTL for automatic cleanup)
- **Purpose**: Coordinate multi-property intent rules — holds the request until ALL priority properties are populated by other intent rules, then triggers final LLM processing via DynamoDB stream
- **Lifecycle**:
  1. **Created** by IntentHandleProcess when a rule has priority properties
  2. **Updated** by IntentHandlePropertyValues as each priority property gets its LLM result
  3. **Stream MODIFY** → IntentHandleProcess checks if all properties are filled → deletes record if complete
  4. **Stream REMOVE** → IntentHandleProcess rebuilds prompt with all property values → sends to LLM queue
  5. **TTL cleanup** → Auto-expires after 30 days if never completed

**ServerSideDataActionIntentPropertyLog** (`dynamodb_table_ss_data_action_intent_property_log`)
- **PK**: `objectId`
- Fields: `accountId`, dynamic `propertyId → value` mapping
- Purpose: Track server-side action logs for intent properties

### BigQuery Tables

**TriggerObjectExistence** (per account dataset)
- `objectID`, `objectType` ('People'/'Company'), `triggerID`, `ruleID`, `ruleType` ('AI_Intent'), `createdAt`
- Purpose: Deduplication — prevents reprocessing same rule+object combination

---

## Processing Pipeline

```
Trigger (SST / Manual Run / Rerun / Preview)
  │
  ▼
IntentHandleProcess Lambda
  - Batching (max 500 records, max 5 concurrent chunks)
  - Deduplication via BigQuery TriggerObjectExistence
  - Prompt building (build_prompt.mjs)
  - Priority property tracking (DynamoDB IntentRequestLogs)
  │
  ▼
SQS → RabbitMQ QueueLLMRequest
  - LLM API call (OpenAI, Claude, xAI, etc.)
  - Token/cost tracking in LLMTrackingRequest
  │
  ▼
IntentHandlePropertyValues Lambda
  - Apply existingValues policy (BLOCK/APPEND/OVERRIDE)
  - Store results in IntentPropertyValues (PostgreSQL)
  - Update priority property tracking (DynamoDB)
  - Sync to BigQuery (People/Companies tables)
  │
  ▼
Pusher notification to client
```

**Preview flow differs**: Results are sent via Pusher WebSocket only, NOT stored in database.

---

## Client-Side Architecture

### Main Components

| Component | Path | Purpose |
|-----------|------|---------|
| Intent Analysis (root) | `client/src/components/cms/subscriber/goals/intelligence/intent-analysis.js` | Feature toggle, tabs for Rules/Datasets/Properties |
| Manage Intent Rules | `client/src/components/cms/subscriber/goals/intent-analysis/manage-intent-rules.js` | List, create/edit modal, status toggle |
| Review Intent Rules | `client/src/components/cms/subscriber/goals/intent-analysis/manage-intent-rules/ReviewIntentRules.js` | Execution history container |
| Review Prompt | `client/src/components/cms/subscriber/goals/intent-analysis/manage-intent-rules/components/ReviewPrompt.js` | Paginated review with SST/Rerun/Preview sections, status badges, token/cost info |
| Classification Datasets | `client/src/components/cms/subscriber/goals/intent-analysis/classification-data-sets.js` | Dataset CRUD, import/export, AI generation |
| Sample Data | `client/src/components/cms/subscriber/goals/intent-analysis/manage-intent-rules/components/SampleData.js` | Sample data display for testing |
| Instructions Preview | `client/src/components/cms/subscriber/goals/intent-analysis/manage-intent-rules/components/InstructionsPreview.js` | Prompt instructions display |

### Client Constants
**File**: `client/src/constants/intent-rules.js`

- `ANALYSIS_TYPE`: classification, summary
- `INTENT_RULES_OBJECT`: people, company
- `INTENT_RULES_STORE`: multiple-values, one-value
- `INTENT_PROPERTIES_STORE`: multiple-tag, single-tag, text-tag
- `INTENT_RULES_EXISTING`: block, override, append
- `AI_INTENT_TRIGGER_TYPE`: server_side_trigger, manual_preview, manual_run, rerun_intent
- LLM model configurations with credit costs
- Status enums: active, inactive, draft

### Client API Constants
**File**: `client/src/constants/apiRest.js`

| Constant | Path |
|----------|------|
| `API_AI_INSIGHTS_INTENT_RULES` | `client/intent-rules` |
| `API_AI_INSIGHTS_INTENT_PROPERTIES` | `client/intent/properties` |
| `API_AI_INSIGHTS_INTENT_DATASETS` | `client/intent-datasets` |
| `API_AI_INSIGHTS_INTENT_PROPERTY_RERUN` | `client/intent/rerun-property` |

### Real-Time Updates
- Pusher channel: `channel-{accountId}`
- Event: `sendIntentPropertyRerunResult` — sends rerun/preview results to client

---

## Supported Object Fields

### People Object
name, email, title, company, domain, phone, city, country, state, region, linkedinUrl, and custom properties

### Company Object
domain, companyName, revenue, employees, industry, primaryIndustry, businessType, country, state, region, yearFounded, monthlyVisitors, and custom properties

---

## Key Constants
**File**: `server/constants/intent.js`

- `AI_INTENT_TRIGGER_TYPE` — Trigger type enum
- `AI_INTENT_TRIGGER_TYPES_TO_CHARGE` — Chargeable trigger types
- `INTENT_RULES_OBJECT` — Supported object types (People, Company)
- `INTENT_RULES_EXISTING` — Existing values policy (block, override, append)
- `ANALYSIS_TYPE` — classification, summary
- `REDIS_PREFIX_KEYS.INTENT_PREVIEW_LLM_QUEUE` — Redis key prefix for preview queues
- `REDIS_PREFIX_KEYS.INTENT_RERUN_QUEUE` — Redis key prefix for rerun queues
- Rolling date ranges: `previous_week_sun_sat`, `previous_month`, `quarter_to_date`, `lifetime_metrics`, etc.

---

## SQS Queues

| Queue | Purpose | Producer | Consumer |
|-------|---------|----------|----------|
| Intent processing queue | Trigger intent rule execution | Server-side triggers / API | IntentHandleProcess |
| `HANDLE_REQUEST_LLM_QUEUE_URL` | LLM request processing | IntentHandleProcess | QueueLLMRequest consumer |
| Intent property values queue | Store LLM results | QueueLLMRequest consumer | IntentHandlePropertyValues |
| `HANDLE_PERSON_BIGQUERY_QUEUE_URL` | Sync people to BigQuery | IntentHandlePropertyValues | BigQuery sync lambda |
| `HANDLE_COMPANIES_BIGQUERY_QUEUE_URL` | Sync companies to BigQuery | IntentHandlePropertyValues | BigQuery sync lambda |
| `HANDLE_SYNC_OBJECT_FIELD_VALUES_QUEUE_URL` | Sync field values | IntentHandlePropertyValues | Field sync lambda |

---

## Environment Variables

- `POSTGRES_HOST`, `POSTGRES_USERNAME`, `POSTGRES_PASSWORD` — PostgreSQL connection
- `REDIS_URL` — Redis for queue storage
- `AWS_CONFIG_REGION`, `AWS_CONFIG_ACCESS_KEY_ID`, `AWS_CONFIG_SECRET_ACCESS_KEY` — AWS credentials
- `DYNAMODB_TABLE_INTENT_REQUEST_LOGS` — DynamoDB table for intent request tracking
- `DYNAMODB_TABLE_ERROR_LOG` — DynamoDB error log table
- `HANDLE_REQUEST_LLM_QUEUE_URL` — SQS URL for LLM processing
- `HANDLE_PERSON_BIGQUERY_QUEUE_URL` — SQS URL for people BigQuery sync
- `HANDLE_COMPANIES_BIGQUERY_QUEUE_URL` — SQS URL for company BigQuery sync
- `HANDLE_SYNC_OBJECT_FIELD_VALUES_QUEUE_URL` — SQS URL for field value sync
- `LOG_LEVEL` — Logging level for RequestTracer

---

## Debugging Guide

### Intent rule not executing via server-side trigger
1. **Check IntentRules.triggers**: Verify the SST ID is in the rule's `triggers` JSONB array
2. **Check rule status**: Must be `active`
3. **Check SST integration**: `server/services/serverSideTrigger.js` line ~2846 — `isRuleIdInIntentRule(ruleId)`
4. **Check deduplication**: Query BigQuery `TriggerObjectExistence` table — the rule+object may already be processed
5. **Check IntentHandleProcess Lambda logs**: Look for batching, prompt building, or SQS errors

### Preview not returning results
1. **Check Redis queue**: Key pattern `INTENT_PREVIEW_LLM_QUEUE_{accountId}_{uuid}`
2. **Check QueueLLMRequest consumer**: `server/queues/queueLLMRequest.js` — verify RabbitMQ is consuming
3. **Check Pusher**: Results are sent via WebSocket, not stored in DB
4. **Check LLMTrackingRequest**: Verify status is not `failed`, check `prompt` and `result` fields

### Rerun blocked or skipping records
1. **Check existingValues policy**: `BLOCK` skips records that already have values
2. **Check IntentPropertyValues**: Query by `propertyId` and `objectId` to see existing values
3. **Check rerun service**: `server/services/intentRules.js` `handleRerunIntentProperty` (lines 594-760)
4. **Check Pusher response**: Blocked records are reported back to client via Pusher

### LLM results not stored
1. **Check IntentHandlePropertyValues Lambda logs**: Look for errors in value storage
2. **Check storeValue config**: `one-value` vs `multiple-values` affects storage logic
3. **Check priority properties**: If rule has priority properties, values may be held in DynamoDB `IntentRequestLogs` until all are populated
4. **Check BigQuery sync**: Verify SQS messages sent to BigQuery queues

### Wrong or unexpected LLM output
1. **Check prompt**: Review `build_prompt.mjs` output — verify dataset, object data, and instructions
2. **Check LLMTrackingRequest**: Read the `prompt` and `result` fields for the specific execution
3. **Check dataset**: Verify `IntentDatasetRules.data` contains correct classification items
4. **Check model config**: Verify `llmModelId` points to correct provider and model

### Intent property values not appearing in BigQuery
1. **Check IntentHandlePropertyValues**: Verify SQS messages sent to `HANDLE_PERSON_BIGQUERY_QUEUE_URL` or `HANDLE_COMPANIES_BIGQUERY_QUEUE_URL`
2. **Check HandleCompanyBigQuery/HandlePersonBigQuery**: These lambdas include intent property values in their BigQuery mapping
3. **Check LLMTrackingRequest.referenceId**: Should link to the `IntentPropertyValues` record

### Cannot delete a server-side trigger
- If the SST is linked to an intent rule via `IntentRules.triggers`, deletion is blocked
- Error: "You cannot [action] this trigger because it is being used by the following IntentRules"
- Remove the SST from the intent rule's triggers array first

---

## End-to-End Flow Diagrams

### Intent Rule Creation Flow
```
User creates Intent Rule (UI)
  → POST /client/intent-rules
  → Validates: object type, property, dataset, LLM model, instructions
  → Stores in IntentRules table (status: draft)
  → User activates rule (status: active)
  → User optionally links server-side triggers
```

### Server-Side Trigger Execution Flow
```
People/Company record changes
  → ServerSideTrigger fires
  → Checks isRuleIdInIntentRule(ruleId)
  → Sends triggerRecordIds to IntentHandleProcess (SQS)
  → IntentHandleProcess deduplicates, builds prompts, queues LLM requests
  → QueueLLMRequest calls LLM API, stores in LLMTrackingRequest
  → IntentHandlePropertyValues stores results in IntentPropertyValues
  → BigQuery sync via SQS
  → Pusher notification to client
```

### Preview Flow
```
User clicks "Run Analysis" (UI)
  → POST /client/intent-rules/run-analysis
  → handleRunAnalysis fetches sample records from BigQuery
  → Builds prompts, queues to Redis (INTENT_PREVIEW_LLM_QUEUE)
  → sendQueueLLMRequest processes via RabbitMQ
  → Results sent via Pusher WebSocket (NOT stored in DB)
  → UI displays results in ReviewPrompt component
```

### Rerun Flow
```
User selects records and clicks "Rerun" (UI)
  → POST /client/intent/rerun-property
  → handleRerunIntentProperty finds active rules for propertyIds
  → Applies existingValues policy (BLOCK/APPEND/OVERRIDE)
  → Queues LLM requests to Redis (INTENT_RERUN_QUEUE)
  → sendQueueLLMRequest processes via RabbitMQ
  → Results stored in IntentPropertyValues + synced to BigQuery
  → Pusher notification to client
```
