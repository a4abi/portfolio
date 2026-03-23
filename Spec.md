# Tax Receipt Collector App — Architecture & Tech Stack Analysis

## Table of Contents

1. [Project Summary](#project-summary)
2. [Key Requirements](#key-requirements)
3. [Architecture Decisions](#architecture-decisions)
4. [Frontend: Flutter vs Compose Multiplatform](#frontend-flutter-vs-compose-multiplatform)
5. [Backend: AWS (DynamoDB vs RDS) vs Firebase](#backend-aws-dynamodb-vs-rds-vs-firebase)
6. [DynamoDB vs RDS PostgreSQL — Deep Dive](#dynamodb-vs-rds-postgresql--deep-dive)
7. [Recommended Combinations](#recommended-combinations)
8. [High-Level Architecture](#high-level-architecture)
9. [Data Model](#data-model)
10. [Cost Analysis by Scale](#cost-analysis-by-scale)
11. [Receipt File Handling Strategy](#receipt-file-handling-strategy)
12. [Recommendation](#recommendation)
13. [Next Steps](#next-steps)

---

## Project Summary

A white-label tax receipt collector app available on **Android, iOS, and Web**. Users upload receipts (photo/PDF), categorize them by CRA expense types, and organize them by tax year and income source. The app must support secure login (Google/Apple ID), user profiles, spouse linking, receipt metadata capture, and up to 7 years of data retention.

---

## Key Requirements

| Requirement | Detail |
|---|---|
| Platforms | Android + iOS + Web (shared codebase preferred) |
| Auth | Google/Apple Sign-In (no credential storage) |
| Storage | Receipt images/PDFs, up to 7 years retention |
| White-label | Configurable logos, labels (JSON), CSS/themes |
| Core MVP | Upload receipt, capture metadata (date, CRA type, amounts), attach file, categorize by income source & FY |
| Nice-to-have | Auto-scan/OCR on receipts |
| Compliance | PIPEDA, no SIN/DL collection |
| Scale | Start small (hundreds of users), plan for tens of thousands |

---

## Architecture Decisions

The stack is broken into two independent decisions:

1. **Frontend framework** — What builds the UI across iOS, Android, and Web?
2. **Backend platform** — What handles auth, database, file storage, and API logic?

These are mix-and-match. Any frontend works with any backend.

---

## Frontend: Flutter vs Compose Multiplatform

### Flutter

**What it is:** Google's UI toolkit. Single Dart codebase compiles to iOS, Android, Web, desktop. Uses its own rendering engine (Skia/Impeller) — does not use native platform widgets.

**Pros:**
- Most mature cross-platform framework for mobile — large community, battle-tested in production
- True single codebase for iOS, Android, AND Web from one Dart project
- Pixel-perfect consistency across platforms — your UI looks identical everywhere
- ThemeData system is excellent for white-label (swap entire theme at runtime via JSON config)
- Hot reload dramatically speeds up UI development
- Rich widget library (Material 3 + Cupertino) out of the box
- Strong plugin ecosystem for camera, file picker, image compression, push notifications
- Compiled to native ARM code — excellent mobile performance
- Large community: 160K+ GitHub stars, extensive packages on pub.dev
- Well-documented integration patterns for Firebase and AWS

**Cons:**
- Flutter Web produces larger initial bundles (~2-4 MB) and uses canvas-based rendering — not native DOM
- Web SEO is limited (canvas rendering is not crawlable by default)
- Dart developer pool is smaller than Kotlin/Java — hiring may take longer
- Web accessibility (screen readers, keyboard navigation) is less mature than native web frameworks
- Platform-specific behavior (e.g., iOS scroll physics, Android back button) requires manual tuning
- Not a native feel — power users may notice the UI isn't using real platform widgets

### Compose Multiplatform (CMP)

**What it is:** JetBrains' extension of Jetpack Compose (Android's modern UI toolkit) to iOS, Web, and Desktop. Uses Kotlin. Shares UI code across platforms while rendering with native components on each platform.

**Pros:**
- Kotlin is the primary language — widely used, strong type system, excellent tooling
- Native Android experience: Compose is Google's recommended UI toolkit for Android
- iOS support uses native rendering — feels more platform-authentic than Flutter on iOS
- Shares business logic AND UI code across platforms via Kotlin Multiplatform (KMP)
- Web target compiles to Kotlin/JS or Kotlin/WASM — produces lighter bundles than Flutter Web
- Kotlin developers are easier to find than Dart developers (especially in Android-heavy teams)
- Strong IDE support (IntelliJ/Android Studio — both JetBrains)
- Can incrementally adopt: start with shared business logic, then share UI
- Ktor (Kotlin HTTP client) integrates well with any backend
- Growing ecosystem with JetBrains actively investing

**Cons:**
- iOS support is newer and less mature than Flutter's (stable since late 2024, but still catching up)
- Fewer production apps on iOS compared to Flutter — less battle-tested edge cases
- Web support (Compose for Web) is still evolving — less stable than Flutter Web
- Smaller cross-platform community than Flutter (CMP-specific resources are limited)
- Fewer ready-made multiplatform plugins for camera, file picker, image compression
- Need to write platform-specific (expect/actual) code more often for native APIs
- White-label theming requires building your own theme system (no equivalent to Flutter's ThemeData JSON-swapping)
- Build toolchain is more complex (Gradle multiplatform configuration)

### Frontend Comparison Summary

| Criteria | Flutter | Compose Multiplatform |
|---|---|---|
| **Language** | Dart | Kotlin |
| **Mobile maturity** | Excellent (6+ years) | Android: Excellent, iOS: Good (maturing) |
| **Web maturity** | Good (functional, large bundles) | Early (evolving, lighter bundles) |
| **Single codebase** | Yes (all platforms) | Yes (all platforms, more expect/actual) |
| **UI consistency** | Pixel-perfect everywhere | Native-feeling per platform |
| **White-label theming** | Excellent (ThemeData, runtime swap) | Good (custom implementation needed) |
| **Developer availability** | Medium (Dart) | Medium-High (Kotlin) |
| **Plugin ecosystem** | Large (pub.dev) | Growing (fewer multiplatform plugins) |
| **Camera/file picker** | Mature plugins available | Requires platform-specific code |
| **Firebase SDK** | Official FlutterFire (excellent) | Official Firebase KMP SDK (newer) |
| **AWS Amplify SDK** | Official amplify_flutter | No official CMP SDK (use REST/Ktor) |
| **Production readiness** | High | Medium (iOS/Web still maturing) |

---

## Backend: AWS (DynamoDB vs RDS) vs Firebase

### AWS Serverless with DynamoDB

Fully serverless, scales to zero, pay-per-request.

| Component | Service |
|---|---|
| Auth | AWS Cognito (User Pool + Identity Pool, Google/Apple federation) |
| Database | Amazon DynamoDB (NoSQL, on-demand capacity) |
| File Storage | Amazon S3 (lifecycle policies for 7-year retention) |
| API | AWS Lambda (Node.js/Python/Kotlin) + API Gateway (HTTP API) |
| OCR (future) | Amazon Textract (AnalyzeExpense API) |
| CDN | Amazon CloudFront |
| Email | Amazon SES |
| Push | Amazon SNS |
| IaC | AWS CDK |

**Pros:**
- True pay-per-request — $0 at zero traffic, no minimum monthly cost for database
- Single-digit millisecond response times at any scale
- No connection pooling, no cold database — always ready
- Scales automatically from 0 to millions of requests with no configuration
- Pairs naturally with Lambda — both serverless, both event-driven
- Built-in encryption at rest, point-in-time recovery, global tables for DR
- No database maintenance — no patches, no upgrades, no vacuuming
- S3 lifecycle policies natively handle 7-year retention with tiering
- Textract is best-in-class for receipt OCR
- Can host in ca-central-1 (Canada) for data residency
- TTL attribute can auto-delete items after 7 years (in addition to S3 lifecycle)

**Cons:**
- NoSQL requires upfront access pattern design — you must know your queries before designing the schema
- No JOINs — tax summaries spanning multiple entity types require denormalization or separate queries
- Aggregate queries (SUM, GROUP BY, COUNT) are not native — must be computed in Lambda or pre-aggregated
- Single-table design has a steep learning curve
- Query flexibility is limited: you can only query on partition key + sort key (or GSIs)
- Adding a new access pattern later may require a new GSI and potentially backfilling data
- GSIs add cost (each GSI duplicates storage and write throughput)
- Maximum item size is 400 KB — large OCR result payloads may need to be stored in S3

### AWS Serverless with RDS PostgreSQL

Relational database with full SQL power, managed by AWS.

| Component | Service |
|---|---|
| Auth | AWS Cognito (User Pool + Identity Pool, Google/Apple federation) |
| Database | Amazon RDS PostgreSQL Serverless v2 |
| File Storage | Amazon S3 (lifecycle policies for 7-year retention) |
| API | AWS Lambda + API Gateway (HTTP API) |
| OCR (future) | Amazon Textract (AnalyzeExpense API) |
| CDN | Amazon CloudFront |
| Email | Amazon SES |
| Push | Amazon SNS |
| IaC | AWS CDK |

**Pros:**
- Full SQL — JOINs, aggregates (SUM, AVG, GROUP BY), subqueries, window functions
- Tax summaries are trivial: `SELECT cra_expense_type, SUM(gross_amount) FROM receipts WHERE user_id = $1 AND tax_year = 2025 GROUP BY cra_expense_type`
- Schema enforces data integrity (foreign keys, constraints, CHECK)
- Flexible querying — add new queries anytime without schema changes
- JSONB columns for semi-structured data (OCR results, theme config)
- Serverless v2 auto-scales ACUs based on load
- Rich ecosystem of ORMs and migration tools
- Easier to reason about for developers familiar with relational databases
- Same S3/Cognito/Textract advantages as DynamoDB option

**Cons:**
- Serverless v2 has a minimum cost (~$15/mo for 0.5 ACU) even at zero traffic
- Lambda + RDS requires connection management (RDS Proxy recommended at scale, adds ~$22/mo)
- Cold database connections from Lambda add latency (RDS Proxy mitigates this)
- Database maintenance windows for minor version upgrades (managed by AWS, but brief downtime)
- Scaling has limits — read replicas needed at very high read loads
- More operational overhead than DynamoDB (monitoring connections, storage, IOPS)

### Firebase (Firestore + Cloud Functions + Firebase Auth)

Google's managed app platform. Minimal backend code, rapid development.

| Component | Service |
|---|---|
| Auth | Firebase Authentication (Google/Apple providers, managed tokens) |
| Database | Cloud Firestore (NoSQL, real-time sync) |
| File Storage | Firebase Storage (backed by Google Cloud Storage) |
| API | Cloud Functions for Firebase (Node.js/Python, event-driven or HTTPS) |
| OCR (future) | Google Cloud Vision API or Document AI |
| CDN | Firebase Hosting (built-in CDN) |
| Email | Firebase Extensions (SendGrid/Mailgun) |
| Push | Firebase Cloud Messaging (FCM) |

**Pros:**
- Fastest time-to-MVP — auth, database, storage, hosting all configured in minutes
- Firebase Auth is the simplest to set up: Google/Apple sign-in in ~5 lines of code
- Firestore has real-time sync built-in — changes appear instantly across devices
- Official Flutter SDK (FlutterFire) is the most mature and well-maintained cross-platform SDK
- Generous free tier: 50K auth MAU, 1 GiB Firestore storage, 5 GB Cloud Storage, 50K reads/day
- Firebase Hosting provides CDN + SSL with zero config
- Cloud Functions trigger on Firestore writes, storage uploads — event-driven patterns are natural
- Security Rules provide per-user data isolation declaratively
- Push notifications via FCM are the easiest to implement across Android/iOS
- Offline support built into Firestore client SDK (works when phone has no connection)

**Cons:**
- Firestore is NoSQL — same limitations as DynamoDB for complex queries, but with even fewer query capabilities (no GSI equivalent; relies on composite indexes)
- No JOINs, no GROUP BY, no SUM — tax summaries must be pre-computed or aggregated client-side/Cloud Functions
- Firestore costs scale with reads/writes (not storage) — chatty list views can spike costs unexpectedly
- Cloud Functions cold starts (~1-3s for Node.js) are worse than AWS Lambda (100-500ms)
- No S3 lifecycle equivalent — Firebase Storage (GCS) has Nearline/Coldline but no Glacier-level pricing
- Vendor lock-in to Google ecosystem — migrating away from Firestore is non-trivial
- Limited backend control — stateless functions only, no long-running processes
- Data residency: single-region can be northamerica-northeast1 (Montreal) but multi-region is US/EU only
- Security Rules language has a learning curve and can become unwieldy

---

## DynamoDB vs RDS PostgreSQL — Deep Dive

This is the key database decision for the AWS backend. Here's how each handles the app's specific access patterns:

### Access Pattern Analysis

| Access Pattern | DynamoDB | RDS PostgreSQL |
|---|---|---|
| **Get user profile** | Simple: `GetItem(PK=USER#id, SK=PROFILE)` | Simple: `SELECT * FROM profiles WHERE user_id = $1` |
| **List receipts by tax year** | Good: `Query(PK=USER#id, SK begins_with RECEIPT#2025)` | Good: `SELECT * FROM receipts WHERE user_id = $1 AND tax_year = 2025` |
| **List receipts by income source** | Needs GSI: `Query(GSI1PK=USER#id#INCOME#srcId)` | Simple: `SELECT * FROM receipts WHERE income_source_id = $1` |
| **Sum expenses by CRA type for a year** | Hard: Scan + filter + aggregate in Lambda | Trivial: `SELECT cra_expense_type, SUM(gross_amount) ... GROUP BY` |
| **Total expenses per month for a year** | Hard: Scan + Lambda aggregation | Trivial: `SELECT DATE_TRUNC('month', receipt_date), SUM(...) ... GROUP BY 1` |
| **Search receipts by date range** | Moderate: Sort key range query (if modeled correctly) | Simple: `SELECT * FROM receipts WHERE receipt_date BETWEEN $1 AND $2` |
| **Spouse lookup by email** | Needs GSI: `Query(GSI2PK=EMAIL#email)` | Simple: `SELECT * FROM users WHERE email = $1` |
| **Cross-income-source summary** | Hard: Multiple queries + Lambda merge | Trivial: JOIN income_sources + GROUP BY type |
| **White-label config lookup** | Simple: `GetItem(PK=TENANT#id, SK=CONFIG)` | Simple: `SELECT * FROM app_config WHERE tenant_id = $1` |
| **Delete all user data (PIPEDA)** | Moderate: Query all items for PK, BatchDelete | Simple: `DELETE FROM receipts WHERE user_id = $1` (cascades) |

### Verdict for This App

The core value of this app is **receipt organization and tax summaries**. Users will frequently:
- View receipts grouped by year, income source, and CRA category
- See totals and breakdowns (how much did I spend on office supplies in 2025?)
- Generate reports for tax filing

These are inherently **relational, aggregate-heavy queries**. DynamoDB can handle them, but requires:
- Pre-computing aggregates in Lambda on every receipt write (denormalization)
- Maintaining GSIs for each access pattern
- More complex application code to assemble summaries

PostgreSQL handles all of these natively with standard SQL.

**However**, DynamoDB wins on:
- **Zero minimum cost** — $0/mo at zero traffic vs ~$15/mo for RDS Serverless v2
- **Zero operational overhead** — no connections to manage, no RDS Proxy needed
- **Simpler Lambda integration** — no connection pooling concerns
- **Infinite auto-scaling** — from 0 to millions of requests without configuration

### When DynamoDB is the Right Choice

DynamoDB works well for this app **if you're willing to**:
1. Pre-compute tax summaries in a Lambda triggered on receipt create/update/delete (write a summary item per user/year)
2. Accept that new reporting needs may require new GSIs and data backfilling
3. Keep application logic in Lambda for any aggregation

This is a valid pattern — many production apps do this. The trade-off is **more application complexity** in exchange for **lower cost and zero database ops**.

### When RDS PostgreSQL is the Right Choice

RDS PostgreSQL is the better fit **if you want**:
1. Simple, ad-hoc querying without pre-planning every access pattern
2. Tax summaries and reports with standard SQL (no Lambda aggregation code)
3. Data integrity enforced at the database level (foreign keys, constraints)
4. Future flexibility to add new reports or queries without infrastructure changes
5. Simpler application code — the database does the heavy lifting

---

## Recommended Combinations

### Option A: Flutter + AWS DynamoDB (Fully Serverless)

Lowest cost, zero operational overhead, true pay-per-use.

| Layer | Technology |
|---|---|
| Mobile + Web | Flutter (single Dart codebase) |
| Auth | AWS Cognito (Google/Apple federation) |
| Database | Amazon DynamoDB (on-demand, single-table design) |
| File Storage | Amazon S3 (7-year lifecycle tiering) |
| API | AWS Lambda + API Gateway (HTTP API) |
| OCR (future) | Amazon Textract (AnalyzeExpense) |
| Web Hosting | S3 + CloudFront or Amplify Hosting |
| Push | Amazon SNS + FCM |
| CDN | Amazon CloudFront |
| IaC | AWS CDK |
| CI/CD | GitHub Actions |

### Option B: Flutter + AWS RDS PostgreSQL (Serverless SQL)

Full SQL power with serverless scaling, best for complex tax reporting.

| Layer | Technology |
|---|---|
| Mobile + Web | Flutter (single Dart codebase) |
| Auth | AWS Cognito (Google/Apple federation) |
| Database | Amazon RDS PostgreSQL Serverless v2 |
| File Storage | Amazon S3 (7-year lifecycle tiering) |
| API | AWS Lambda + API Gateway (HTTP API) |
| OCR (future) | Amazon Textract (AnalyzeExpense) |
| Web Hosting | S3 + CloudFront or Amplify Hosting |
| Push | Amazon SNS + FCM |
| CDN | Amazon CloudFront |
| IaC | AWS CDK |
| CI/CD | GitHub Actions |

### Option C: Flutter + Firebase

Fastest MVP, best Flutter SDK, real-time sync, simplest setup.

| Layer | Technology |
|---|---|
| Mobile + Web | Flutter (single Dart codebase) |
| Auth | Firebase Authentication (Google/Apple sign-in) |
| Database | Cloud Firestore (NoSQL, real-time) |
| File Storage | Firebase Storage |
| API | Cloud Functions for Firebase (event-driven + HTTPS) |
| OCR (future) | Google Cloud Vision API via Cloud Functions |
| Web Hosting | Firebase Hosting (built-in CDN) |
| Push | Firebase Cloud Messaging (FCM) |
| CI/CD | GitHub Actions + Firebase CLI |

### Option D: CMP + AWS DynamoDB (Kotlin-First, Fully Serverless)

For Kotlin-experienced teams, lowest cost backend.

| Layer | Technology |
|---|---|
| Mobile + Web | Compose Multiplatform (Kotlin, single codebase) |
| Auth | AWS Cognito (Google/Apple federation) via Ktor |
| Database | Amazon DynamoDB (on-demand) |
| File Storage | Amazon S3 |
| API | AWS Lambda (Kotlin) + API Gateway |
| OCR (future) | Amazon Textract |
| Web Hosting | S3 + CloudFront |
| Push | Amazon SNS + FCM |
| IaC | AWS CDK |
| CI/CD | GitHub Actions |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│                                                                 │
│   ┌──────────┐    ┌──────────┐    ┌──────────────────────┐     │
│   │ iOS App  │    │ Android  │    │   Web App (Browser)  │     │
│   │ (Store)  │    │ (Store)  │    │                      │     │
│   └────┬─────┘    └────┬─────┘    └──────────┬───────────┘     │
│        │               │                     │                  │
│        └───────────────┼─────────────────────┘                  │
│                        │                                        │
│         Option A/B/C: Flutter (Dart, single codebase)           │
│         Option D:     Compose Multiplatform (Kotlin)            │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS
                         │
┌────────────────────────┼────────────────────────────────────────┐
│                    AUTH LAYER                                    │
│                                                                 │
│   Option A/B/D: AWS Cognito (User Pool + Identity Pool)         │
│   Option C:     Firebase Auth (managed tokens)                  │
│                                                                 │
│   All: Google Sign-In + Apple Sign-In → JWT issued              │
└────────────────────────┬────────────────────────────────────────┘
                         │ JWT
                         │
┌────────────────────────┼────────────────────────────────────────┐
│                    API LAYER                                     │
│                                                                 │
│   Option A/B/D: API Gateway (HTTP API) → AWS Lambda             │
│   Option C:     Cloud Functions for Firebase                    │
│                                                                 │
│   Core Operations:                                              │
│   - CRUD receipts (metadata + file reference)                   │
│   - CRUD user profiles & income sources                         │
│   - Tax year summaries (aggregate queries)                      │
│   - Generate presigned URL for direct S3/Storage upload         │
│   - Trigger OCR on uploaded receipts                            │
└──────────┬──────────────────────────┬───────────────────────────┘
           │                          │
           ▼                          ▼
┌──────────────────┐      ┌───────────────────────────────┐
│    DATABASE       │      │   FILE STORAGE                 │
│                   │      │                               │
│  Option A/D:      │      │  Option A/B/D: Amazon S3      │
│   DynamoDB        │      │  Option C:     Firebase Stor. │
│   (NoSQL)         │      │                               │
│                   │      │  Structure:                   │
│  Option B:        │      │  /{tenant_id}/                │
│   RDS PostgreSQL  │      │    /{user_id}/                │
│   (Serverless v2) │      │      /{tax_year}/             │
│                   │      │        /{income_src}/         │
│  Option C:        │      │          /receipt_*           │
│   Firestore       │      │                               │
│                   │      │  S3 Lifecycle (Option A/B/D): │
│  Entities:        │      │  Yr 0-1: Standard             │
│  - users          │      │  Yr 1-3: IA ($0.0125/GB)     │
│  - profiles       │      │  Yr 3-7: Glacier Instant      │
│  - income_sources │      │          ($0.004/GB)          │
│  - receipts       │      │  Yr 7+:  Auto-delete          │
│  - app_config     │      │                               │
│  - tax_summaries  │      │                               │
│    (DynamoDB only)│      │                               │
└──────────────────┘      └───────────────────────────────┘

                         │
                         ▼ (Future / Nice-to-have)
┌─────────────────────────────────────────────────────────────────┐
│                    OCR / AI LAYER                                │
│                                                                 │
│   Option A/B/D: Amazon Textract (AnalyzeExpense API)            │
│   Option C:     Google Cloud Vision API                         │
│                                                                 │
│   Flow:                                                         │
│   1. File upload event triggers function (Lambda/Cloud Func)    │
│   2. Function calls OCR API                                     │
│   3. Structured data returned: vendor, date, amounts, tax       │
│   4. Write extracted fields to database                         │
│   5. User reviews/confirms auto-filled fields in app            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  SUPPORTING SERVICES                             │
│                                                                 │
│  AWS Options (A/B/D):                                           │
│  - Amazon SES: Transactional emails                             │
│  - Amazon SNS: Push notifications                               │
│  - AWS Secrets Manager: API keys, DB credentials                │
│  - Amazon CloudWatch: Logging, monitoring, alarms               │
│  - AWS KMS: Encryption at rest (S3, DynamoDB, RDS)              │
│  - AWS WAF: API rate limiting, bot protection                   │
│  - Amazon Route 53: DNS                                         │
│                                                                 │
│  Firebase Option (C):                                           │
│  - FCM: Push notifications                                      │
│  - Firebase Hosting: CDN + SSL                                  │
│  - Security Rules: Per-user data isolation                      │
│  - Firebase Extensions: Email (SendGrid)                        │
└─────────────────────────────────────────────────────────────────┘
```

### S3 Lifecycle Policy for 7-Year Retention (Options A, B, D)

```
Year 0–1:   S3 Standard              ($0.023/GB/mo)  — frequent access, recent receipts
Year 1–3:   S3 Infrequent Access     ($0.0125/GB/mo) — occasional lookups
Year 3–7:   S3 Glacier Instant       ($0.004/GB/mo)  — rare access, ms retrieval for audits
Year 7+:    Auto-delete via lifecycle rule
```

---

## Data Model

### DynamoDB Single-Table Design (Options A, D)

```
Table: TaxApp
Partition Key: PK (String)    Sort Key: SK (String)

─────────────────────────────────────────────────────────────────
PK                          SK                          Attributes
─────────────────────────────────────────────────────────────────
USER#<userId>               PROFILE                     firstName, lastName, dob, email, phone, city, province, spouseEmail, tenantId
USER#<userId>               INCOME#<sourceId>           type, name, isDefault, fyStartDate, fyEndDate
USER#<userId>               RECEIPT#<taxYear>#<date>#<id> incomeSourceId, craExpenseType, netAmt, taxAmt, grossAmt, s3Key, fileSize, ocrStatus, ocrData
USER#<userId>               SUMMARY#<taxYear>           totalNet, totalTax, totalGross, byCategory: {office: $X, travel: $Y, ...}, byIncome: {...}
TENANT#<tenantId>           CONFIG                      appName, logoPath, labelsJson, themeJson, disclaimerText, tutorialConfig
EMAIL#<email>               USER#<userId>               (sparse: for spouse lookup)

GSI1 — Query receipts by income source:
  GSI1PK: USER#<userId>#INCOME#<sourceId>
  GSI1SK: RECEIPT#<taxYear>#<date>#<id>

GSI2 — Query by CRA expense type within a year:
  GSI2PK: USER#<userId>#YEAR#<taxYear>
  GSI2SK: CRA#<craType>#<date>#<id>

Summary Pattern:
  On every receipt write (create/update/delete), a Lambda function
  recalculates and upserts the SUMMARY#<taxYear> item for that user.
  This pre-computed summary powers the dashboard without scan operations.
```

### PostgreSQL Schema (Option B)

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cognito_sub     VARCHAR UNIQUE NOT NULL,
    email           VARCHAR UNIQUE NOT NULL,
    phone           VARCHAR NOT NULL,
    tenant_id       UUID REFERENCES app_config(tenant_id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(email, phone)
);

CREATE TABLE profiles (
    user_id         UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    first_name      VARCHAR NOT NULL,
    last_name       VARCHAR NOT NULL,
    dob             DATE NOT NULL,
    city            VARCHAR NOT NULL,
    province        VARCHAR NOT NULL,
    spouse_email    VARCHAR REFERENCES users(email)
);

CREATE TABLE income_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            VARCHAR NOT NULL CHECK (type IN (
                        'full_time','student','self_employed',
                        'sole_prop','corp_director','rental_owner')),
    name            VARCHAR NOT NULL,
    is_default      BOOLEAN DEFAULT FALSE,
    fy_start_date   DATE,
    fy_end_date     DATE
);

CREATE TABLE receipts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    income_source_id    UUID NOT NULL REFERENCES income_sources(id),
    tax_year            SMALLINT NOT NULL,
    receipt_date        DATE NOT NULL CHECK (receipt_date <= CURRENT_DATE),
    cra_expense_type    VARCHAR NOT NULL,
    net_amount          NUMERIC(12,2) NOT NULL,
    tax_amount          NUMERIC(12,2) NOT NULL,
    gross_amount        NUMERIC(12,2) NOT NULL,
    s3_key              VARCHAR NOT NULL,
    file_size_bytes     INTEGER,
    thumbnail_s3_key    VARCHAR,
    ocr_status          VARCHAR DEFAULT 'none' CHECK (ocr_status IN ('none','pending','complete','failed')),
    ocr_data            JSONB,
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_receipts_user_year ON receipts(user_id, tax_year);
CREATE INDEX idx_receipts_user_source ON receipts(user_id, income_source_id);
CREATE INDEX idx_receipts_user_cra ON receipts(user_id, cra_expense_type);

CREATE TABLE app_config (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_name        VARCHAR NOT NULL,
    logo_s3_key     VARCHAR,
    favicon_s3_key  VARCHAR,
    labels_json     JSONB NOT NULL,
    theme_json      JSONB NOT NULL,
    disclaimer_text TEXT,
    tutorial_config JSONB
);
```

### Firestore Schema (Option C)

```
users/{userId}
├── email, phone, tenantId, createdAt
├── profile: { firstName, lastName, dob, city, province, spouseEmail }
├── incomeSources/{sourceId}: { type, name, isDefault, fyStartDate, fyEndDate }
├── receipts/{receiptId}: {
│       incomeSourceId, taxYear, date, craExpenseType,
│       netAmount, taxAmount, grossAmount,
│       filePath, fileSizeBytes, thumbnailPath,
│       ocrStatus, ocrData, createdAt
│   }
└── summaries/{taxYear}: {
        totalNet, totalTax, totalGross,
        byCategory: { office: $X, travel: $Y, ... },
        byIncome: { ... }
    }

appConfig/{tenantId}
├── appName, logoPath, faviconPath
├── labelsJson, themeJson, disclaimerText, tutorialConfig

Note: summaries are pre-computed by Cloud Functions on receipt writes,
same pattern as DynamoDB. Firestore doesn't support GROUP BY or SUM.
```

---

## Cost Analysis by Scale

All costs are **monthly estimates in USD**.

### Small Scale: 100–500 Users, ~5,000 receipts/month

| Component | A: AWS DynamoDB | B: AWS RDS PostgreSQL | C: Firebase |
|---|---|---|---|
| **Auth** | Cognito: Free (<50K MAU) | Cognito: Free | Firebase Auth: Free |
| **Database** | DynamoDB on-demand: ~$0.50 | RDS Serverless v2: ~$15/mo (0.5 ACU min) | Firestore: ~$0 (free tier) |
| **File Storage** (50 GB) | S3: ~$1.15 | S3: ~$1.15 | Firebase Storage: ~$1.30 |
| **Compute / API** | Lambda + API GW: Free tier | Lambda + API GW: Free tier | Cloud Functions: Free tier |
| **CDN / Hosting** | CloudFront: Free tier | CloudFront: Free tier | Firebase Hosting: Free |
| **Textract / Vision** (1K pages) | ~$1.50 | ~$1.50 | ~$1.50 (Cloud Vision) |
| **Push** | SNS: ~$0.50 | SNS: ~$0.50 | FCM: Free |
| **Monitoring** | CloudWatch: ~$0 | CloudWatch: ~$0 | Included |
| **Total** | **~$4/mo** | **~$19/mo** | **~$3/mo** |

---

### Medium Scale: 1,000–10,000 Users, ~50,000 receipts/month

| Component | A: AWS DynamoDB | B: AWS RDS PostgreSQL | C: Firebase |
|---|---|---|---|
| **Auth** | Cognito: Free | Cognito: Free | Firebase Auth: Free |
| **Database** | DynamoDB on-demand: ~$8/mo | RDS Serverless v2 (1-4 ACU): ~$50/mo | Firestore: ~$25/mo |
| **RDS Proxy** | N/A | ~$22/mo | N/A |
| **File Storage** (500 GB) | S3: ~$11.50 | S3: ~$11.50 | Firebase Storage: ~$13 |
| **S3 requests** | ~$3 | ~$3 | Included in Firestore pricing |
| **Compute / API** | Lambda: ~$5 + API GW: ~$5 | Lambda: ~$5 + API GW: ~$5 | Cloud Functions: ~$10 |
| **CDN / Hosting** | CloudFront: ~$10 | CloudFront: ~$10 | Firebase Hosting: ~$5 |
| **OCR** (10K pages) | Textract: ~$15 | Textract: ~$15 | Cloud Vision: ~$15 |
| **CloudWatch** | ~$10 | ~$10 | Included |
| **SES / Email** | ~$1 | ~$1 | ~$1 (SendGrid) |
| **Total** | **~$69/mo** | **~$133/mo** | **~$69/mo** |

---

### Large Scale: 50,000–100,000 Users, ~500,000 receipts/month

| Component | A: AWS DynamoDB | B: AWS RDS PostgreSQL | C: Firebase |
|---|---|---|---|
| **Auth** | Cognito: ~$28/mo | Cognito: ~$28/mo | Firebase Auth: Free |
| **Database** | DynamoDB on-demand: ~$70/mo | RDS Serverless v2 (4-16 ACU): ~$200/mo | Firestore: ~$300/mo |
| **RDS Proxy** | N/A | ~$22/mo | N/A |
| **File Storage** (5 TB) | S3 (with tiering): ~$80 | S3 (with tiering): ~$80 | Firebase Storage: ~$130 |
| **S3 requests** | ~$25 | ~$25 | Included |
| **Compute / API** | Lambda: ~$60 + API GW: ~$20 | Lambda: ~$60 + API GW: ~$20 | Cloud Functions: ~$120 |
| **CDN / Hosting** | CloudFront: ~$85 | CloudFront: ~$85 | Firebase Hosting: ~$30 |
| **OCR** (100K pages) | Textract: ~$150 | Textract: ~$150 | Cloud Vision: ~$150 |
| **CloudWatch** | ~$40 | ~$40 | Included |
| **SES** | ~$5 | ~$5 | ~$5 |
| **WAF** | ~$11 | ~$11 | N/A |
| **Push** | SNS: ~$5 | SNS: ~$5 | FCM: Free |
| **Total** | **~$579/mo** | **~$731/mo** | **~$735/mo** |

---

### Cost Summary Across Scales

| Scale | A: AWS DynamoDB | B: AWS RDS PG | C: Firebase |
|---|---|---|---|
| Small (100-500 users) | **~$4/mo** | ~$19/mo | ~$3/mo |
| Medium (1K-10K users) | **~$69/mo** | ~$133/mo | ~$69/mo |
| Large (50K-100K users) | **~$579/mo** | ~$731/mo | ~$735/mo |

**Key observations:**
- **DynamoDB is cheapest at every scale** — pay-per-request with no minimum, no connection pooling overhead
- **RDS has the highest small-scale cost** (~$19/mo minimum) but provides the simplest development experience for tax reporting queries
- **Firebase matches DynamoDB at small/medium scale** but becomes the most expensive at large scale due to read/write pricing
- **The ~$15/mo RDS minimum** is the price of full SQL convenience — worth it if developer time savings outweigh the cost
- **S3 lifecycle tiering** (AWS options) saves 40-68% on storage over 7 years vs Firebase Storage
- Frontend choice (Flutter vs CMP) has zero impact on infrastructure cost — Options A and D are identical in cost

---

### One-Time / Fixed Recurring Costs (All Options)

| Item | Cost |
|---|---|
| Apple Developer Account | $99 USD/year |
| Google Play Developer Account | $25 USD one-time |
| Domain (Route 53 or external) | ~$12-15/year |
| SSL Certificate | Free (ACM for AWS, auto for Firebase) |
| PIPEDA compliance review (legal) | $2,000–$5,000 one-time (recommended) |

---

### Storage Growth: 7-Year Retention Cost Projection

Assuming 1 MB average per receipt (compressed), 10,000 users, 20 receipts/user/year:

| Year | Storage | S3 Standard/mo | S3 with Lifecycle/mo | Firebase Storage/mo | Savings (S3 lifecycle) |
|---|---|---|---|---|---|
| 1 | 200 GB | $4.60 | $4.60 | $5.20 | — |
| 3 | 600 GB | $13.80 | $7.50 | $15.60 | 46% |
| 5 | 1 TB | $23.00 | $8.80 | $26.00 | 62% |
| 7 | 1.4 TB | $32.20 | $10.20 | $36.40 | 68% |

---

## Receipt File Handling Strategy

**Recommended approach: Client-side compression with preview confirmation.**

```
Upload Flow:
1. User captures/selects image from phone camera or gallery
2. Client-side compression to configurable max (e.g., 2 MB) maintaining CRA-acceptable quality
3. Show preview: "This is what will be stored. Proceed or re-upload?"
4. On confirm:
   - AWS: Request presigned S3 URL from Lambda → upload directly to S3
   - Firebase: Upload via FirebaseStorage.instance.ref().putFile(...)
5. Storage event triggers function → generate thumbnail + save metadata to DB
6. For PDFs → store as-is (typically already compact)

Why presigned URLs (AWS):
- Lambda payload limit is 6 MB (API Gateway is 10 MB)
- Receipt photos from phones can be 5-15 MB before compression
- Direct-to-S3 upload has no size limit and better performance

CRA Compliance Note:
- CRA accepts digital copies if "legible and complete"
- Recommended minimum: 300 DPI equivalent, ~1-2 MB per image
- Compression quality is configurable per white-label tenant in app_config
```

---

## Recommendation

### Primary: **Option A — Flutter + AWS DynamoDB**

**Why this combination wins:**

1. **Flutter is the safest frontend choice** — Most mature cross-platform framework. Single Dart codebase for iOS, Android, and Web. Excellent white-label theming via ThemeData.

2. **Lowest cost at every scale** — $4/mo at small scale, $579/mo at 100K users. DynamoDB on-demand has no minimum cost. No RDS Proxy, no connection pooling overhead.

3. **Zero operational overhead** — DynamoDB is fully managed: no patching, no connection limits, no vacuuming, no scaling knobs. Lambda + DynamoDB is the most hands-off serverless architecture on AWS.

4. **Millisecond performance at any scale** — DynamoDB delivers single-digit ms latency whether you have 100 or 100,000 users. No cold database connections.

5. **S3 lifecycle tiering** — Native 7-year retention with automatic tiering saves up to 68% on storage over time. This is a significant advantage over Firebase Storage.

6. **Textract for OCR** — Best-in-class receipt OCR, triggered directly from S3 events. Structured output (vendor, date, line items, amounts).

7. **Canadian data residency** — All services available in ca-central-1.

8. **Tax summaries via pre-computation** — Write a Lambda that recalculates summary items on every receipt change. The dashboard reads a single pre-computed item — fast and cheap. This adds application complexity but eliminates the need for RDS.

**The trade-off:** You must design your access patterns upfront and pre-compute aggregates. This is more work than writing SQL queries, but the cost and operational simplicity rewards are significant for a startup.

### Choose Option B (AWS RDS PostgreSQL) instead if:
- Developer simplicity is worth ~$15-65/mo more per month
- You want ad-hoc tax reporting queries without pre-computation
- Your team is more comfortable with SQL than NoSQL patterns
- You anticipate complex, evolving reporting requirements beyond basic summaries

### Choose Option C (Firebase) instead if:
- Absolute fastest MVP is the priority (Firebase setup is the simplest)
- Real-time sync across devices matters from day one (Firestore excels here)
- Your team already knows Firebase/FlutterFire
- Offline-first support is important (Firestore client SDK caches locally)

### Choose Option D (CMP + AWS DynamoDB) instead if:
- Your team is Kotlin-first and strongly prefers Kotlin over Dart
- You're willing to accept CMP's less mature iOS/Web targets
- Same backend cost as Option A

---

## Next Steps

1. **Set up AWS CDK project** — Define Cognito User Pool, DynamoDB table + GSIs, S3 bucket (with lifecycle), Lambda functions, API Gateway
2. **Design DynamoDB access patterns** — Finalize PK/SK design, GSIs, and summary pre-computation Lambda
3. **Scaffold Flutter project** — Initialize with `flutter create`, add `amplify_flutter` for Cognito auth, configure deep linking for OAuth
4. **Build white-label system** — Tenant config in DynamoDB + Flutter ThemeData provider loading config on app start
5. **Implement auth flow** — Google/Apple sign-in → Cognito → JWT → profile creation in DynamoDB
6. **Build receipt upload** — Camera/gallery → compress → preview → presigned S3 URL → direct upload → Lambda processes metadata
7. **Build core screens** — Receipt list (grouped by FY/income source), receipt detail, add/edit receipt
8. **Build tax summary dashboard** — Read pre-computed SUMMARY items, display totals by CRA category and income source
9. **Deploy web** — Flutter Web build hosted on S3 + CloudFront
10. **First-launch tutorial** — Interactive walkthrough (disclaimer, PIPEDA message, security, data persistence)
11. **App store submission** — iOS App Store + Google Play Store
12. **Add OCR (post-MVP)** — S3 trigger → Lambda → Textract AnalyzeExpense → auto-fill receipt fields
