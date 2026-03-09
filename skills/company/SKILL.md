---
name: company
description: Search and explain company-related lambdas, API endpoints, data models, creation flows, target accounts, company identifiers, centralized companies, client UI, and debugging across datalayerapp-v2, llservices-salesforce, and datalayerapp-v2-function-code repositories.
---

# Company Skill

This skill helps you understand, debug, and navigate the company-related code across the Datalayer platform. It covers lambda functions, API endpoints, data models, creation flows, target accounts, company identifiers, centralized companies, client UI, and debugging guidance.

## Invocation

When the user asks about companies, use this knowledge base to:
1. Identify which lambdas and files are relevant to their question
2. Explain the end-to-end company creation, enrichment, and reveal flow
3. Explain target accounts, company identifiers, and centralized company features
4. Help debug company-related issues by pointing to the right logs, tables, and code

## Repositories

| Repository | Path | Role |
|---|---|---|
| **datalayerapp-v2-function-code** | `D:\Projects\Datalayer\datalayerapp-v2-function-code` | AWS Lambda functions for company processing |
| **datalayerapp-v2** | `D:\Projects\Datalayer\datalayerapp-v2` | Main Express.js application (API endpoints, models, analytics, client UI) |
| **llservices-salesforce** | `D:\Projects\Datalayer\llservices-salesforce` | Salesforce sync service (CompanyJobs queue processing) |

---

## What Are Companies?

Companies represent B2B organizations identified through website visitor IP/domain resolution. The system:

- **Detects** visitor IP/domain against company databases (KickFire, TheCompaniesAPI)
- **Identifies** which B2B organizations are browsing your website
- **Enriches** with business intelligence (employees, revenue, industry, location, social links)
- **Tracks** company engagement metrics across all visitor sessions
- **Enables** B2B targeting and account-based marketing via Target Accounts

---

## Company Data Sources

Companies enter the system through three channels, tracked via `dataSources`:

| Source | Constant | Description |
|---|---|---|
| Revealed | `REVEALED` | Automatically detected from website visitor IP/domain resolution |
| Provided | `PROVIDED` | Submitted via API by external systems |
| Imported | `IMPORTED` | Bulk imported via HandleImportBulkCompanies |

---

## Lambda Functions (datalayerapp-v2-function-code)

### 1. HandleImportBulkCompanies
- **Path**: `lambda-functions/HandleImportBulkCompanies/`
- **Handler**: `index.mjs`
- **Purpose**: Entry point for bulk company imports
- **What it does**:
  - Receives a Redis key containing a domain list and metadata
  - Parses and deduplicates domains, filters out personal email domains
  - Checks if domains already exist in DynamoDB (DOMAIN_COMPANY table) or Salesforce DB (CompanyJobs table)
  - Creates CompanyJob records in Salesforce DB with status `WAITING`
- **Key files**: `models/db_sf_handler.mjs`, `models/dynamo_handler.mjs`
- **Triggered by**: API call / manual import

### 2. HandleThreadCreateCompanies
- **Path**: `lambda-functions/HandleThreadCreateCompanies/`
- **Handler**: `index.mjs`
- **Purpose**: Processes waiting CompanyJobs in batches
- **What it does**:
  - Queries Salesforce DB for CompanyJobs with status `WAITING` (limit 500, older than 30 min)
  - Extracts domain from each job
  - Sends SQS message per domain to `HANDLE_REVEALED_COMPANIES_QUEUE`
  - Updates job status to `DONE`
  - If exactly 500 jobs processed, sends SQS to itself for pagination
- **Key files**: `models/db_sf_handler.mjs`, `utils/sqs.mjs`
- **Triggered by**: Scheduled event / SQS (self-invocation for pagination)

### 3. Handle_RevealedCompanies
- **Path**: `lambda-functions/Handle_RevealedCompanies/`
- **Handler**: `index.js`
- **Purpose**: Core company revelation and enrichment engine
- **What it does**:
  1. **Account check**: Verifies reveal feature is enabled, checks quota
  2. **Data prefetch**: Batch gets visitors, IP/domain mappings, existing companies from DynamoDB
  3. **Deduplication**: Pre-processes unique IPs/domains across all payloads
  4. **External API calls**:
     - **Kickfire API v3**: `https://api.kickfire.com/v3/company:(all)?website={domain}&key={apiKey}`
     - **TheCompaniesAPI**: `https://api.thecompaniesapi.com/v1/companies/{domain}?token={token}`
  5. **Data transformation**: UUID v5 generation, NAICS code mapping, org type classification
  6. **Atomic quota reservation**: DynamoDB conditional update on AccountStatistics
  7. **Batch storage**: Writes to Company, IP-Company, Domain-Company tables; updates visitors
  8. **SQS batch dispatch** (batches of 10 via `sendSQSBatch`):
     - `HANDLE_CENTRALIZED_COMPANY_QUEUE` — CentralizedCompany upsert + session sync
     - `HANDLE_COMPANIES_BIGQUERY_QUEUE` — BigQuery sync (deduped against centralized messages, expired companies only)
     - `HANDLE_GRAPH_MERGE_DATA_QUEUE` — person identity merge
  - **Note**: Handle_RevealedCompanies no longer sends SQS to HandleAddMoreDataToTheCompany
- **Key files**:
  - `services/companyDataService.js` - External API integration with rate limiting
  - `processors/companyBatchProcessor.js` - Batch deduplication across payloads
  - `processors/payloadProcessor.js` - Core business logic per payload
  - `services/syncService.js` - Visitor sync, account-company sync, request logging
  - `services/storageService.js` - DynamoDB batch operations, SQS batch sending, BigQuery dedup
  - `models/dynamoHandler.js` - DynamoDB operations including atomic quota reservation
  - `models/dbHandler.js` - PostgreSQL operations (account/package data, CentralizedCompany lookups)
  - `constants/index.js` - 442+ personal domains, rate limits, source types
  - `constants/naics.js` - NAICS industry classification
  - `constants/blacklist.js` - ISP/mobile blacklist
- **Triggered by**: SQS (HANDLE_REVEALED_COMPANIES_QUEUE)

### 4. HandleCheckDomainCompany
- **Path**: `lambda-functions/HandleCheckDomainCompany/`
- **Handler**: `index.js`
- **Purpose**: On-demand domain-based company lookup
- **What it does**:
  - Checks if domain exists in DynamoDB company table
  - If not found, calls Kickfire/TheCompaniesAPI
  - Stores new company record and domain mapping
  - Logs the API request for quota tracking
- **Key files**: `models/index.js`, `utils/index.js`
- **Triggered by**: API Gateway / direct invocation

### 5. HandleAddMoreDataToTheCompany
- **Path**: `lambda-functions/HandleAddMoreDataToTheCompany/`
- **Handler**: `index.js`
- **Purpose**: Extended company data enrichment
- **What it does**:
  - Batch gets companies from DynamoDB by companyId
  - Calls TheCompaniesAPI for extended data (concurrent limit: 50)
  - Merges extended data (employee ranges, revenue, etc.) with existing records
  - Batch updates companies in DynamoDB
  - Fetches AccountCompany records and sends to BigQuery queue
- **Key files**: `models/dynamoHandler.js`, `helpers.js`
- **Triggered by**: SQS (HANDLE_ADD_MORE_DATA_TO_THE_COMPANY_QUEUE)

### 6. HandleCentralizedCompany
- **Path**: `lambda-functions/HandleCentralizedCompany/`
- **Handler**: `index.mjs`
- **Purpose**: Upserts CentralizedCompany records in PostgreSQL, validates domains/LinkedIn, syncs company to DynamoDB sessions
- **What it does**:
  1. **Fetch companies from DynamoDB**: Batch get by companyId, parse socialNetworks from JSON string
  2. **Validate domain & LinkedIn** (parallel): HTTP checks for domain reachability (concurrency: 10) and LinkedIn URL validity (concurrency: 5), updates DynamoDB Company table with results
  3. **Prepare records**: Build CentralizedCompany records with companyName, domain, dataSources, validDomain, validLinkedin, firstSessionId, date fields
  4. **Upsert to PostgreSQL**: `INSERT ... ON CONFLICT (companyId, accountId)` with merge logic:
     - `dataSources`: append new sources (skip if 'imported' or already present)
     - `firstSessionId`: `COALESCE(existing, excluded)` — set only if previously null
     - Date fields (`revealedAt`, `providedAt`, `importedAt`): keep earliest via COALESCE
     - Returns `xmax::text` for INSERT vs UPDATE detection
  5. **Send BigQuery SQS**: Only for new records (xmax='0') or records where dataSources changed
  6. **Sync company to session**: For new records or records where firstSessionId was just set — fetches session from DynamoDB, adds company to session.companies array, batch updates sessions
- **Key files**:
  - `models/db_handler.mjs` - PostgreSQL upsert with ON CONFLICT, xmax detection
  - `models/dynamo_handler.mjs` - DynamoDB batch get/update (companies, sessions, validation)
  - `services/centralizedCompanyService.mjs` - Record preparation
  - `services/domainValidationService.mjs` - HTTP domain validation
  - `services/linkedinValidationService.mjs` - LinkedIn URL validation
  - `helpers.mjs` - SQS batch sending (batches of 10)
- **Triggered by**: SQS (HANDLE_CENTRALIZED_COMPANY_QUEUE)

### 7. HandleCompanyBigQuery
- **Path**: `lambda-functions/HandleCompanyBigQuery/`
- **Handler**: `index.js`
- **Purpose**: Sync company data to BigQuery via S3
- **What it does**:
  - Parses SQS messages: `{ accountId, companyId }`
  - Gets companies and account-company records from DynamoDB
  - Gets accounts from PostgreSQL (timezone info)
  - Gets intent property values from PostgreSQL
  - Deduplicates (keeps latest by sentTimestamp)
  - Maps data using `getMappingData()` into BigQuery schema
  - Uploads JSON records to S3: `s3://bucket/accountId_{id}/YYYY-MM-DD-HH/Companies/`
- **Key files**: `models/db_handler.js`, `models/dynamo_handler.js`, `s3/`
- **Triggered by**: SQS (HANDLE_COMPANIES_BIGQUERY_QUEUE)

---

## API Endpoints (datalayerapp-v2)

### Company Identifier
- `GET /admin/company-identifier/:accountId` - List company identifiers (paginated, filterable by value/type/source/status/enrichment)
- `GET /admin/company-identifier/retrieve/:id` - Retrieve single company identifier
- **Files**: `server/routes/companyIdentifier.js`, `server/controllers/companyIdentifier.js`, `server/services/companyIdentifier.js`, `server/models/companyIdentifier.js`

### Company Match
- `POST /admin/company-matches` - Get all company match data (global)
- `POST /admin/contract-plan/company-matches` - Get company matches by contract plan
- `POST /admin/account/company-matches` - Get company matches for specific account
- **Files**: `server/routes/companyMatch.js`, `server/controllers/companyMatch.js`, `server/services/companyMatch.js`

**Match Statistics Tracked**:
- Match types: ipData, domainData, ipLookup, domainLookup
- API calls: KickFire, IP, Domain
- Provider/Company presence, noMatch counts
- Confidence ranges: 0-25, 26-50, 51-75, 76-85, 86-95

### Centralized Company (Admin)
- `GET /admin/centralized-company/:accountId` - List centralized companies (filterable by companyName, domain, sourceFirst, sourceLast, dataSources, validDomain, validLinkedin, enriched, validPersonMatch, invalidPersonMatch)
- `GET /admin/centralized-company/retrieve/:id` - Retrieve single centralized company with account info
- `POST /admin/centralized-company` - Find centralized company by account context
- **Files**: `server/routes/centralizedCompany.js`, `server/controllers/centralizedCompany.js`, `server/services/centralizedCompany.js`, `server/models/centralizedCompany.js`

### Record Profile
- `POST /client/record-profile/company` - Get full enriched company profile with metrics
- Returns: Company data, lifetime metrics (sessions, users, people, conversions, page views), engagement scores, source attribution

### Account Company Management
- `PUT /admin/account/update-revealed-company` - Enable/disable revealed companies per account, set monthly quota
- `PUT /client/account/companyStatus` - Toggle company status (ON/OFF)

### Target Account Endpoints
| Method | Path | Purpose |
|---|---|---|
| POST | `/client/target-account/create-table-bigquery` | Create BigQuery table for target accounts |
| POST | `/client/target-account/get-list-domain` | Get domain list for account |
| POST | `/client/target-account/import` | Import target accounts |
| POST | `/client/target-account/check-import-domain` | Validate import data |
| POST | `/client/target-account/full-list` | Retrieve all target accounts (paginated) |
| PUT | `/client/target-account` | Update target accounts |
| DELETE | `/client/target-account` | Remove target account |
| GET | `/client/target-account/condition-value/:accountId` | Get condition values for filtering |
| POST | `/client/target-account/find-account` | Find companies matching conditions |

### Account Group Endpoints
| Method | Path | Purpose |
|---|---|---|
| GET | `/client/account-groups/:accountId` | List groups for account |
| GET | `/client/account-group/:id` | Get specific group with rules |
| POST | `/client/account-groups` | Create new group |
| PUT | `/client/account-groups` | Update group |
| DELETE | `/client/account-groups` | Delete group |
| DELETE | `/client/account-group` | Remove target account from group |
| POST | `/client/account-groups/script-create-default` | Create default groups |

- **Files**: `server/routes/targetAccount.js`, `server/controllers/targetAccount.js`, `server/services/targetAccount.js`, `server/models/targetAccount.js`, `server/constants/targetAccount.js`

---

## Server-Side Services

### Target Account Service
**File**: `server/services/targetAccount.js` (1004 lines, 17 exports)

| Function | Purpose |
|---|---|
| `handleCreateTableTargetAccountBigquery` | Create TargetAccounts BigQuery table with time partitioning |
| `handleUpdateTargetAccount` | Process updates, handle group assignments, manage BO status |
| `handleImportTargetAccount` | Bulk import with domain validation, company data enrichment |
| `handleRemoveTargetAccount` | Delete and track in history |
| `handleGetListDomainByAccount` | Query domains with optional group filtering |
| `handleGetListTargetAccount` | Paginated retrieval with complex joins to groups |
| `handleGetFullValueCompanyInfo` | Extract distinct company attributes for filter dropdowns |
| `handleGetListCompanyFindAccount` | Query companies by filter conditions |
| `handleGetListGroups` | List groups per account |
| `handleGetRuleGroup` | Retrieve group with deserialized conditions |
| `handleCreateDefaultAccountGroups` | Generate default "Big Opportunities" group |
| `handleCreateGroups` | Create groups with optional dynamic rules |
| `handleUpdateGroups` | Update group metadata and conditions |
| `handleRemoveGroups` | Delete group and track related accounts |
| `handleRemoveTargetAccountGroups` | Unassign account from group |
| `scriptCreateDefaultAccountGroups` | Batch script for all accounts |
| `handleAddSectionTargetAccount` | Add new fields to BigQuery analytics tables |

### Target Account Model
**File**: `server/models/targetAccount.js` (1127 lines, 40+ exports)

Key operation groups:
- **Target Account CRUD**: createTargetAccount, createMultiTargetAccount, updateTargetAccount, removeTargetAccount
- **Target Account Queries**: getListDomainInfo, getCompanyInfo, getCompanyHavingCondition, getFullValueCompanyInfo, findTargetAccountByAccountId
- **Groups CRUD**: createGroups, updateGroups, removeGroups, findListGroupByAccountId
- **Group-Account Relations**: createTargetAccountGroups, removeTargetAccountGroupsByAccountGroup, getTargetAccountsByGroupId
- **Dynamic Rule Processing**: processRuleDynamicAccountGroup, getTargetAccountInGroupDynamic
- **BigQuery Operations**: insertDataTargetAccountToBigQuery, updateDataTargetAccountToBigQuery, deleteTargetAccountBigQuery, updateTargetAccountInRevealedCompaniesStats, updateTargetAccountInPeopleStats

### Account Service (Company Functions)
**File**: `server/services/account.js`

| Function | Purpose |
|---|---|
| `handleUpdateAccountCompanyStatus` | Toggle companyStatus, track in history |
| `updateAccountRevealedCompany` | Set isRevealedCompanies, maxCompanyRequestMonthly, sync with plan |
| `handleTurnOnRevealedCompanyAccount` | Enable revealed companies, set unlimited quota (-1) |
| `handleUpdateUsingTargetAccount` | Toggle usingTargetAccount feature, disable related goals |
| `handleCheckCompanyContractPlanMonthly` | Scheduled job to reset company request counters per billing cycle |

---

## Database Tables

### DynamoDB Tables (datalayerapp-v2-function-code)

| Table | Purpose | Key Fields |
|---|---|---|
| `DYNAMODB_TABLE_COMPANY` | Company master records (unique by domain) | `id` (UUID v5 from domain), `domain` (has `domain-index` GSI), `companyName`, `employees`, `revenue`, `sector`, `industry`, `confidence` |
| `DYNAMODB_TABLE_IP_COMPANY` | IP to company mapping | `ip`, `companyId`, `createdAt`, `endTime` (45-day TTL) |
| `DYNAMODB_TABLE_DOMAIN_COMPANY` | Domain to company mapping | `domain`, `companyId`, `createdAt`, `endTime` |
| `DYNAMODB_TABLE_ACCOUNT_COMPANY` | Account-specific company data | `id` (accountId+companyId), `accountId`, `companyId`, `dataSources`, `createdAt`, `providedAt`, `revealedAt`, `importedAt` |
| `DYNAMODB_TABLE_VISITORS` | Visitor records with company link | `id`, `personId`, `lastSessionId`, `companyId` |
| `DYNAMODB_TABLE_SESSIONS` | Session records with company link | `id`, `accountId`, `visitorId`, `companyId` |
| `DYNAMODB_TABLE_ACCOUNT_STATISTICS` | Quota tracking | `accountId`, `numberOfCompaniesRequestMonthly` |
| `DYNAMODB_TABLE_LOGS_COMPANY_REQUEST` | API request logs | `id`, `accountId`, `domain`, `isCallApi`, `confidence`, `createdAt` |
| `DYNAMODB_TABLE_ERROR_LOG` | Error tracking | `inputData`, `error`, `timestamp` |

### PostgreSQL Tables (datalayerapp-v2)

| Table | Purpose | Key Fields |
|---|---|---|
| `CompanyIdentifier` | Company identifier metadata | id, accountId, visitorId, sessionId, companyId, type, value, source, status, enrichmentStatus, llmStatus |
| `TargetAccounts` | Target accounts with company mapping | id, accountId, domain, companyId, isBO, isSyncBigQuery, companyData (serialized JSON) |
| `CentralizedCompany` | Centralized company master data | id, companyId, accountId, firstSessionId, companyName, domain, sourceFirst, sourceLast, dataSources, linkedinURL, validDomain, validLinkedin, enriched, validPersonMatch, invalidPersonMatch, peopleCount, revealedAt, providedAt, importedAt. **Unique constraint**: (companyId, accountId). **Indexes**: accountId, companyId, domain, (accountId, domain). **Filterable fields**: companyName, domain, sourceFirst, sourceLast, dataSources, validDomain, validLinkedin (ILIKE); enriched, validPersonMatch, invalidPersonMatch (boolean) |
| `AccountGroups` | Target account group definitions | id, accountId, name, type (LIST_OF_ACCOUNTS/DYNAMIC_LIST), conditions |
| `TargetAccountGroups` | Many-to-many: accounts to groups | targetAccountId, accountGroupId |

**TargetAccounts Migration**: `server/migrations/20241010091500_tcreate_table_targetAccount.js`

### Salesforce DB (llservices-salesforce)

| Table | Purpose |
|---|---|
| `CompanyJobs` | Company import job queue (id, accountId, domain, recordId, object, status, turnId, logs) |

---

## BigQuery Query Builders & Schemas

### Company Data Query Builders
| File | Purpose |
|---|---|
| `server/models/query-builders/dashboard/metric-dashboard/companyGroup.js` | Company grouping with reveal status |
| `server/models/query-builders/all-pages/sub-queries/companyData.js` | Company data extraction for pages |
| `server/models/query-builders/acquisition-channels-details/sub-queries/companyData.js` | Company data for acquisition |
| `server/models/query-builders/conversion/all-conversion/sub-queries/companyData.js` | Company data for conversions |
| `server/models/query-builders/event-details/sub-queries/companyData.js` | Company data for events |
| `server/models/query-builders/merge-report-query/sub-queries/companyData.js` | Company data for merged reports |
| `server/models/query-builders/merge-report-query/sub-queries/companyMetrics.js` | Company metrics |

### Target Account Query Builders
| File | Purpose |
|---|---|
| `server/models/query-builders/dashboard/metric-dashboard/targetAccount.js` | `getTargetAccountRevealedQuery`, `getTargetAccountActiveQuery` |
| `server/models/query-builders/people-details/sub-queries/targetAccountData.js` | People with target account domains |
| `server/models/query-builders/session-details/sub-queries/targetAccountData.js` | Session-level target account joins |
| `server/models/query-builders/user-details/sub-queries/targetAccountData.js` | User-level target account joins |

### BigQuery Schemas
**File**: `server/constants/bigquery/schemas/companies/companyParent.js`
- Company RECORD schema with `id`, `name`, `domain`, `descriptionShort`, `socialNetworks` (REPEATED RECORD with 13 social platform URLs)

**File**: `server/constants/bigquery/schemas/revealedCompaniesStats/companyTargetAccount.js`
- Target Account columns: `targetAccountID` (STRING), `isTargetAccount` (BOOLEAN), `targetAccountTimestampUTC` (TIMESTAMP), `targetAccountTimestamp` (STRING)

---

## Client-Side Architecture

### Pages & Components

**Company Profile Detail**:
- `client/src/components/cms/subscriber/analytics/record-profile/CompanyDetail.js`
  - Comprehensive revealed company profile with enriched data
  - Displays: company name, logo, domain, social links, identified date, latest session
  - Lifetime metrics: sessions, engaged sessions, users, people, conversions, page views
  - Accordion sections: Description, Size, Industry/NAICS, Location (with Google Maps), Other Data
  - Related tables: All Pages, Sources, Conversions, People Details, User Details, Session Details

**Reveal Companies Settings**:
- `client/src/components/cms/subscriber/sources/reveal/reveal-companies/index.js`
  - Toggle to enable "Reveal companies in your analytics data"
  - Monthly company matches limit configuration
  - Requires "Reveal" subscription

**Target Accounts Management**:
- `client/src/components/cms/subscriber/goals/target-accounts/index.js` - Main page with Full List and Account Groups
- `client/src/components/cms/subscriber/goals/target-accounts/find-account/CompanyTableInfo.js` - Searchable grid for finding companies
- `client/src/components/cms/subscriber/goals/target-accounts/full-list/table/index.js` - Full list management
- `client/src/components/cms/subscriber/goals/target-accounts/account-groups/index.js` - Account groups (dynamic lists, list of accounts, big opportunities)
- `client/src/components/cms/subscriber/goals/target-accounts/actions/index.js` - Request handlers
- `client/src/components/cms/subscriber/goals/target-accounts/actions/apis.js` - API call functions

**Admin Components**:
- `client/src/components/cms/admin/centralized-company/index.js` - Legacy centralized company list (uses old qualityScore/status fields)
- `client/src/components/cms/admin/centralized-company/CentralizedCompanyDetails.js` - Legacy single company detail
- `client/src/components/cms/admin/company-matches/CompanyMatchesDetail.js` - KickFire API usage stats
- `client/src/components/cms/admin/accounts/AccountRevealCompany.js` - Enable/disable reveal per account
- `client/src/components/cms/admin/accounts/EnrichmentRule/retrieve/CompanyIdentifier.js` - Company identifier enrichment pipeline status
- `client/src/components/cms/admin/accounts/EnrichmentRule/tabs/CentralizedCompanyPanel.js` - Current centralized company list (per-account, with enriched/validPersonMatch/invalidPersonMatch columns)
- `client/src/components/cms/admin/accounts/EnrichmentRule/retrieve/CentralizedCompany.js` - Current centralized company detail (shows all fields including enriched, validDomain/validLinkedin badges, person match, peopleCount, source timestamps)

### Client Routing

**Subscriber Routes**:
- `/:secondId/goals/target-accounts` - Target Accounts management
- `/:secondId/explore-data/revealed-companies` - Revealed Companies report
- `/:secondId/privacy/reveal/revealed-companies` - Reveal settings

**Admin Routes**:
- `/company-matches` - Global company match stats
- `/accounts/company-matches/:accountId` - Per-account match stats
- `/contract-plans/company-matches/:packageId` - Per-plan match stats
- `/centralized-company` - Centralized company list
- `/centralized-company/:id` - Centralized company detail

### Client API Constants

| Constant | Path | Purpose |
|---|---|---|
| `API_RECORD_PROFILE_COMPANY` | `client/record-profile/company` | Get enriched company profile |
| `ADMIN_ACCOUNT_UPDATE_REVEALED_COMPANY` | `admin/account/update-revealed-company` | Enable/disable reveal |
| `API_UPDATE_ACCOUNT_COMPANY_STATUS` | `/client/account/companyStatus` | Toggle company status |
| `ADMIN_CENTRALIZED_COMPANY` | `admin/centralized-company` | List centralized companies |
| `ADMIN_CENTRALIZED_COMPANY_BY_ACCOUNT` | `admin/centralized-company/:accountId` | By account |
| `ADMIN_COMPANY_IDENTIFIERS_BY_ACCOUNT` | `admin/company-identifier/:accountId` | List identifiers |
| `ADMIN_CENTRALIZED_COMPANY_RETRIEVE` | `admin/centralized-company/retrieve/:id` | Retrieve single |
| `ADMIN_COMPANY_IDENTIFIER_RETRIEVE` | `admin/company-identifier/retrieve/:id` | Retrieve identifier |
| `ADMIN_COMPANY_MATCHES` | `admin/company-matches` | Global match data |
| `ADMIN_ACCOUNT_COMPANY_MATCHES` | `admin/account/company-matches` | Account match data |
| `CLIENT_TARGET_ACCOUNT` | `client/target-account` | Target account operations |

### Client Constants

**Component Names**: `RECORD_PROFILE_COMPANY`, `TARGET_ACCOUNTS_FIND_ACCOUNT`, `ADMIN_COMPANY_IDENTIFIER`, `ADMIN_CENTRALIZED_COMPANY`

**Target Account Group Types**: `DYNAMIC_LIST`, `LIST_OF_ACCOUNTS`, `BIG_OPPORTUNITIES`

**Find Account Company Fields**: Domain, Company Name, Revenue, Employees, Industry, Products & Services Tags, Country, State, Business Type, Monthly Visitors, Year Founded

**Company Data Mapping** (31 fields): companyName, descriptionShort, description, yearFounded, countryData, countryShort, region, regionShort, cityData, city, postal, latitude, longitude, timeZoneId, timeZoneName, employees, totalEmployeesExact, revenue, alexaRank, monthlyVisitors, businessType, stockExchange, stockSymbol, primaryIndustry, industries, naicsCode, naicsSector, naicsSubsector, naicsGroup, naicsDesc, sicCode, sicGroup, socialNetworks

---

## Company Data Structure

```javascript
{
  companyId: string,           // UUID v5 (from domain) - ensures 1 domain = 1 company
  companyName: string,
  domain: string,
  logo: string,
  identifiedDate: string,
  latestSession: string,

  // Lifetime Metrics
  countSession: number,
  countEngaged: number,
  countUser: number,
  countPeople: number,
  countConversions: number,
  countPageView: number,
  sessionDuration: number,
  avgSessionDuration: number,
  engagementScore: number,
  isTargetAccount: boolean,

  // Source Attribution
  sourceFirst: object,
  sourceLast: object,
  sourcePath: array,
  dataSources: array,          // ['REVEALED', 'PROVIDED', 'IMPORTED']

  // Description
  descriptionShort: string,
  description: string,

  // Size/Scale
  monthlyVisitors: number,
  revenue: string,
  stockExchange: string,
  stockSymbol: string,
  phone: string,
  employees: number,
  totalEmployeesExact: number,
  yearFounded: number,
  alexaRank: number,

  // Industry
  businessType: string,
  primaryIndustry: string,
  industries: array,
  naicsCode: string,
  naicsSector: string,
  naicsSubsector: string,
  naicsGroup: string,
  naicsDesc: string,
  sicCode: string,
  sicGroup: string,

  // Location
  address: string,
  city: string,
  county: object,
  country: string,
  countryShort: string,
  state: object,
  continent: object,
  region: string,
  regionShort: string,
  postal: string,
  latitude: number,
  longitude: number,
  timeZoneId: string,
  timeZoneName: string,

  // Other
  confidence: number,
  tradeName: string,
  isISP: boolean,
  isWifi: boolean,
  isMobile: boolean,
  createdAt: timestamp,
  updatedAt: timestamp,

  // Social Networks
  socialNetworks: {
    linkedin: string,
    linkedinSalesNavigator: string,
    facebook: string,
    instagram: string,
    twitter: string,
    youtube: string,
    pinterest: string,
    angellist: string,
    crunchbase: string,
    dribbble: string,
    github: string
  }
}
```

---

## Target Account Group Types

| Type | Constant | Description |
|---|---|---|
| List of Accounts | `LIST_OF_ACCOUNTS` | Manual static list of target companies |
| Dynamic List | `DYNAMIC_LIST` | Rules-based automatic grouping by company attributes |
| Big Opportunities | `BIG_OPPORTUNITIES` | Default group for high-value opportunities (BO flag) |

---

## Company Enrichment Pipeline

```
Company Identifier
  |
  +-- Validation Stage
  |     - Valid Status (validated/invalid/pending)
  |     - Validation Run timestamp
  |     - Validation Data (details)
  |
  +-- Enrichment Stage
  |     - Enrich Status (enriched/failed/pending)
  |     - Enrichment Run timestamp
  |     - Enrichment Data (from 3rd party APIs)
  |
  +-- LLM Stage (AI processing)
  |     - LLM Status (good/poor/processing)
  |     - LLM Run timestamp
  |     - LLM Data (AI-generated insights)
  |
  +-- Company Status (active/inactive/archived)
```

---

## End-to-End Company Creation Flow

```
[Client/API Import]
       |
       v
HandleImportBulkCompanies
  - Receives domains from Redis
  - Deduplicates, filters personal emails
  - Creates CompanyJobs in SF DB (status: WAITING)
       |
       v
HandleThreadCreateCompanies
  - Reads waiting CompanyJobs (batch 500)
  - Sends each domain to SQS
  - Updates jobs to DONE
       |
       v
Handle_RevealedCompanies
  - Checks account feature & quota
  - Calls Kickfire API / TheCompaniesAPI
  - Generates UUID v5 company ID
  - Validates (not ISP/mobile/personal domain)
  - Atomic quota reservation
  - Stores in DynamoDB (company, IP-company, domain-company)
  - Updates visitor records
  - SQS fan-out (batch sending):
    → HandleCentralizedCompany (upsert + session sync)
    → HandleCompanyBigQuery (expired companies only, deduped)
    → GraphMergeData (person merge)
       |
       ├───────────────────────────────┐
       v                               v
HandleCentralizedCompany        HandleCompanyBigQuery
  - Validates domain + LinkedIn     - Maps company data to BigQuery schema
  - Updates DynamoDB Company        - Uploads JSON to S3
  - Upserts CentralizedCompany     - BigQuery ingests from S3
    in PostgreSQL (ON CONFLICT)            |
  - Syncs company to session               v
  - Sends to BigQuery queue ──────> [Analytics / Dashboards]
    (new/changed records)             - Query builders serve data
                                      - CompanyIdentifier & CompanyMatch

HandleAddMoreDataToTheCompany (triggered separately, not by Handle_RevealedCompanies)
  - Fetches extended data from TheCompaniesAPI (concurrency: 50)
  - Merges with existing DynamoDB company records
  - Sends to BigQuery queue for all linked accounts
```

---

## Client-Side UI Flows

### Company Discovery & Reveal Flow
1. User enables "Reveal companies" in Sources > Revealed Companies settings
2. System begins matching visitor IPs/domains to company database
3. Revealed companies appear in Insights > Revealed Companies report
4. Users click on company for full enriched profile modal

### Target Accounts Management Flow
```
Target Accounts Page
+-- FIND ACCOUNTS Button
|   +-- Opens FindAccountContext Modal
|   +-- Users filter by Domain, Company Name, Revenue, etc.
|   +-- CompanyTableInfo displays matching companies (grid, 20/page)
|   +-- Users select via checkboxes
|   +-- "Save Change" to add to target accounts list
|
+-- Target Accounts Full List
|   +-- Domains already marked as target accounts
|   +-- Edit BO (Big Opportunities) flag
|   +-- Assign to Account Groups
|   +-- Delete or export
|
+-- Account Groups
    +-- DYNAMIC LIST - Rules-based automatic grouping
    +-- LIST OF ACCOUNTS - Manual account list
    +-- BIG OPPORTUNITIES - Default high-value group
```

### Company Profile Detail Flow
```
Click on Company Name/Link (from any report)
  |
  v
Modal opens (RecordProfileCompany)
  +-- Header: logo, name, domain, social links
  +-- Key metrics: sessions, users, conversions
  +-- Accordions: Description, Size, Industry, Location, Other Data
  +-- Related tables: All Pages, Sources, Conversions
  +-- People/User/Session details nested tables
```

### Admin Flow
```
Admin > Accounts > Account Detail
  +-- Revealed Companies section: enable/disable, set monthly quota

Admin > Centralized Company
  +-- View all centralized company records
  +-- Filter by companyName, domain, sources, validation status, enriched, person match
  +-- View Detail: enrichment pipeline status (validation, enrichment, LLM)

Admin > Company Matches
  +-- Daily company matching statistics
  +-- KickFire API usage metrics, confidence distribution
```

---

## SQS Queues

| Queue | Purpose | Producer | Consumer |
|---|---|---|---|
| `HANDLE_REVEALED_COMPANIES_QUEUE` | Domain revelation requests | HandleThreadCreateCompanies | Handle_RevealedCompanies |
| `HANDLE_CENTRALIZED_COMPANY_QUEUE` | CentralizedCompany upsert + session sync | Handle_RevealedCompanies | HandleCentralizedCompany |
| `HANDLE_ADD_MORE_DATA_TO_THE_COMPANY_QUEUE` | Enrichment requests | External trigger | HandleAddMoreDataToTheCompany |
| `HANDLE_COMPANIES_BIGQUERY_QUEUE` | BigQuery sync | HandleCentralizedCompany, HandleAddMoreDataToTheCompany, Handle_RevealedCompanies (expired only) | HandleCompanyBigQuery |
| `HANDLE_GRAPH_MERGE_DATA_QUEUE` | Person merge data | Handle_RevealedCompanies | Graph processing lambda |

**BigQuery dedup**: Handle_RevealedCompanies filters `sqsCompanyBigQueryMessages` to exclude items already in `sqsCentralizedCompanyMessages`, since HandleCentralizedCompany sends its own BigQuery SQS after upsert.

**SQS batching**: Both Handle_RevealedCompanies and HandleCentralizedCompany use `sendSQSBatch` (batches of 10 via `SendMessageBatchCommand`) to reduce SQS API calls.

---

## Debugging Guide

### Company not appearing after import
1. **Check CompanyJobs table** (llservices-salesforce): Verify job status is not stuck at `WAITING`
   - File: `llservices-salesforce/server/models/companyJobs.js` - `findUniqueAccountIds()`
   - Jobs must be 30+ min old to be picked up
2. **Check HandleThreadCreateCompanies logs**: Verify SQS messages were sent
3. **Check Handle_RevealedCompanies logs**: Look for quota exceeded, feature disabled, or API errors
4. **Check DynamoDB tables**: Query `DYNAMODB_TABLE_COMPANY` and `DYNAMODB_TABLE_DOMAIN_COMPANY` for the domain

### Company data incomplete or wrong
1. **Check external API responses**: Kickfire and TheCompaniesAPI may return partial data
   - Rate limiting: Check `companyDataService.js` for backoff/retry logic
2. **Check enrichment**: Verify `HandleAddMoreDataToTheCompany` ran successfully
3. **Check BigQuery sync**: Verify `HandleCompanyBigQuery` uploaded to S3

### Quota issues
1. **Check AccountStatistics table**: `numberOfCompaniesRequestMonthly` vs account's `maxRequestMonthly`
2. **Atomic reservation logic**: `Handle_RevealedCompanies/models/dynamoHandler.js` - `atomicReserveQuota()`
3. **Unlimited quota**: `maxRequestMonthly = -1` means unlimited
4. **Monthly reset**: `handleCheckCompanyContractPlanMonthly` in `server/services/account.js`

### Personal domain filtered
1. **Check constants**: `Handle_RevealedCompanies/constants/index.js` has 442+ personal email domains
2. Domains like gmail.com, yahoo.com, hotmail.com are excluded from company revelation

### ISP/Mobile/WiFi filtered
1. **Check blacklist**: `Handle_RevealedCompanies/constants/blacklist.js`
2. **Check API response flags**: `isp`, `mobile`, `wifi` fields from Kickfire API

### Company ID generation & domain uniqueness
- **Rule**: Each company is unique by its `id`, and `id = uuidv5(domain, uuidv5.URL)`, so each company is also unique by domain
- UUID v5 is generated from the **domain** using the URL namespace (`uuidv5.URL`)
- This is consistent across all lambdas:
  - `Handle_RevealedCompanies/services/companyHandler.js:32`: `id: company.id || uuidv5(company.domain, uuidv5.URL)`
  - `HandleCheckDomainCompany/index.js:111`: `id: uuidv5(domain, uuidv5.URL)`
  - `PayloadProcessor` (LLM path): delegates to `companyHandler.getCompanyDynamo()` which uses the same rule
- **CAPI domain protection**: When merging data from TheCompaniesAPI, `'domain'` is included in `deleteFields` to prevent CAPI from overwriting the original domain. This is enforced in all three lambdas:
  - `Handle_RevealedCompanies/services/companyDataService.js` (also has explicit `domain: this.domain` override)
  - `HandleCheckDomainCompany/utils/index.js` (caller also explicitly sets `domain` after merge)
  - `HandleAddMoreDataToTheCompany/utils/index.js` (critical — this was the only path where domain could be silently overwritten)
- **Dedup layers**: Before creating a company, the system checks:
  1. `DYNAMODB_TABLE_DOMAIN_COMPANY` (domain → companyId cache, 45-day TTL)
  2. `DYNAMODB_TABLE_COMPANY` via `domain-index` GSI (query by domain)
  3. Only if both miss, a new company is created with `uuidv5(domain)`

### Target account not syncing to BigQuery
1. **Check isSyncBigQuery flag**: `TargetAccounts` table
2. **Check BigQuery operations**: `server/models/targetAccount.js` - `insertDataTargetAccountToBigQuery`, `updateDataTargetAccountToBigQuery`
3. **Check schema**: `server/constants/bigquery/schemas/revealedCompaniesStats/companyTargetAccount.js`

### Target account dynamic group not updating
1. **Check group type**: Must be `DYNAMIC_LIST`
2. **Check conditions**: `server/models/targetAccount.js` - `processRuleDynamicAccountGroup`, `getTargetAccountInGroupDynamic`
3. **Check rule deserialization**: `server/services/targetAccount.js` - `handleGetRuleGroup`

### Company enrichment pipeline status
1. **Check CompanyIdentifier table**: Validation, enrichment, and LLM statuses
2. **Check CentralizedCompany table**: enriched flag, validDomain, validLinkedin, validPersonMatch, invalidPersonMatch, peopleCount
3. **Files**: `server/models/companyIdentifier.js`, `server/models/centralizedCompany.js`

### Session not showing company
1. **Session sync moved to HandleCentralizedCompany**: Session company sync is triggered after CentralizedCompany upsert, not in Handle_RevealedCompanies
2. **Sync conditions**: Only for new CentralizedCompany records (xmax='0') or existing records where `firstSessionId` was null (just got set via COALESCE)
3. **Check DynamoDB session**: Query SESSIONS table for the sessionId, inspect `companies` JSON array
4. **Check HandleCentralizedCompany logs**: Look for "Session sync: N session(s) updated"

### CentralizedCompany not created
1. **Check SQS delivery**: Handle_RevealedCompanies sends to `HANDLE_CENTRALIZED_COMPANY_QUEUE` via `sendSQSBatch`
2. **Check message format**: Must include `{ companyId, accountId, sessionId, companyDataSource }`
3. **Check HandleCentralizedCompany logs**: Look for upsert counts and errors
4. **Check unique constraint**: CentralizedCompany has `UNIQUE(companyId, accountId)` — ON CONFLICT handles duplicates

---

## Target Account Constants
**File**: `server/constants/targetAccount.js`

- `DEFAULT_TARGET_ACCOUNT_GROUPS` - Default "Big Opportunities" group template
- `DEFAULT_BO_NAME` - "Big Opportunities"
- `TARGET_ACCOUNT_GROUP_TYPE` - `LIST_OF_ACCOUNTS`, `DYNAMIC_LIST`
- `FIELD_UPDATE` - 31 updatable company fields
- `COMPANY_BLACKLIST` - 16 email domains (yahoo, gmail, hotmail, etc.)
- `MAPPING_COMPANY_DATA` - 31 company data fields for serialization
- `REGEX_DOMAIN` - Domain validation regex

---

## Key Environment Variables

- `API_KEY` - Kickfire API key
- `THE_COMPANIES_API_TOKEN` - TheCompaniesAPI token
- `REDIS_URL` - Redis connection for bulk import data
- All `DYNAMODB_TABLE_*` variables for table names
- All `*_QUEUE_URL` variables for SQS queues
- `POSTGRES_HOST`, `POSTGRES_USERNAME`, `POSTGRES_PASSWORD` - PostgreSQL connection
