# Implementation Plan — AI Execution Phases

> Each phase is a single, self-contained task that takes under 1 minute for an AI to implement.
> Phases are sequential within each milestone. Complete all phases before moving to the next milestone.

---

## Milestone 01 — Project Skeleton & Config

### Phase 1.1 — Initialize Node.js project
- Run `npm init -y` in project root
- Set `"type": "module"` in package.json
- Set `"main": "src/index.js"`
- Add scripts: `"dev": "nodemon src/index.js"`, `"start": "node src/index.js"`

### Phase 1.2 — Install core dependencies
- Run: `npm install express pg knex dotenv cors helmet express-rate-limit winston uuid zod`

### Phase 1.3 — Install queue & cache dependencies
- Run: `npm install bullmq ioredis`

### Phase 1.4 — Install AI & PDF dependencies
- Run: `npm install openai puppeteer handlebars`

### Phase 1.5 — Install storage & auth dependencies
- Run: `npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner jsonwebtoken bcrypt multer`

### Phase 1.6 — Install dev dependencies
- Run: `npm install -D nodemon jest supertest eslint prettier`

### Phase 1.7 — Create folder structure
```
src/
├── config/          # App config, env loader, constants
├── routes/          # Express route definitions
│   ├── webhook/     # Webhook routes (assessment, payment)
│   └── admin/       # Admin portal API routes
├── middleware/      # Auth, validation, error handling middleware
├── services/        # Business logic services
│   ├── scoring/     # Scoring engine
│   ├── ai/          # OpenAI narrative engine
│   ├── pdf/         # Puppeteer PDF generation
│   ├── storage/     # DigitalOcean Spaces
│   ├── crm/         # FluentCRM integration
│   └── queue/       # BullMQ job queue
├── models/          # Database models / Knex queries
├── templates/       # HTML report templates per brand
│   └── saferelate/  # Default brand
├── charts/          # Modular chart renderer modules
├── utils/           # Shared utilities (logger, helpers)
└── index.js         # Entry point
```
- Create all directories (empty, with .gitkeep files)

### Phase 1.8 — Create config loader
- Create `src/config/index.js`
- Load dotenv
- Export config object with all env vars grouped by category: `db`, `redis`, `openai`, `crm`, `spaces`, `auth`, `app`, `queue`, `report`
- Validate required env vars on startup — throw if missing

### Phase 1.9 — Create logger utility
- Create `src/utils/logger.js`
- Use winston with JSON format
- Log levels: error, warn, info, debug
- Console transport for dev, file transport for production
- Export named logger instance

### Phase 1.10 — Create Express app setup
- Create `src/index.js`
- Import express, config, logger
- Apply middleware: `helmet()`, `cors()`, `express.json()`, `express-rate-limit`
- Add health check route: `GET /api/health` → returns `{ status: 'ok', timestamp }`
- Start server on `config.app.port`
- Log startup message with port

### Phase 1.11 — Create .gitignore
- Add: `node_modules/`, `.env`, `*.pdf`, `dist/`, `coverage/`, `.DS_Store`

### Phase 1.12 — Create .env.example
- Copy from `.env` but with placeholder values only (no real secrets)
- Add comments explaining each variable

---

## Milestone 02 — Database Schema & Migrations

### Phase 2.1 — Initialize Knex
- Create `knexfile.js` at project root
- Configure development and production environments using config loader
- Set migrations directory: `src/db/migrations`
- Set seeds directory: `src/db/seeds`

### Phase 2.2 — Migration: create `submissions` table
- Columns: `id` (UUID, PK), `assessment_key` (varchar), `user_email` (varchar), `first_name` (varchar), `last_name` (varchar), `consent_status` (boolean), `raw_payload` (jsonb), `status` (enum: submitted/processing/generated/failed/sent), `created_at`, `updated_at`
- Index on `user_email`, `assessment_key`, `status`, `created_at`

### Phase 2.3 — Migration: create `assessment_configs` table
- Columns: `id` (UUID, PK), `assessment_key` (varchar, unique), `name` (varchar), `dimensions` (jsonb), `question_map` (jsonb), `weights` (jsonb), `reverse_scored` (jsonb), `score_bands` (jsonb), `recommendation_rules` (jsonb), `prompt_version_id` (UUID, FK), `template_id` (varchar), `brand_id` (UUID, FK), `chart_config` (jsonb), `clone_origin` (UUID, nullable), `cloned_by` (varchar, nullable), `is_active` (boolean, default true), `created_at`, `updated_at`

### Phase 2.4 — Migration: create `prompts` table
- Columns: `id` (UUID, PK), `prompt_type` (enum: free_report/full_report/recommendation/brand_tone), `version_number` (integer), `content` (text), `status` (enum: draft/active/archived), `change_notes` (text), `created_by` (varchar), `created_at`
- Unique constraint on `(prompt_type, version_number)`
- Index on `prompt_type`, `status`

### Phase 2.5 — Migration: create `brands` table
- Columns: `id` (UUID, PK), `name` (varchar), `logo_url` (varchar), `primary_color` (varchar), `secondary_color` (varchar), `cta_labels` (jsonb), `cta_urls` (jsonb), `footer_text` (text), `contact_details` (text), `tone_prompt` (text, nullable), `is_active` (boolean), `created_at`, `updated_at`

### Phase 2.6 — Migration: create `scoring_results` table
- Columns: `id` (UUID, PK), `submission_id` (UUID, FK → submissions), `total_score` (decimal), `subscale_scores` (jsonb), `score_bands` (jsonb), `recommendation_category` (varchar), `pattern_flags` (jsonb), `scoring_config_snapshot` (jsonb), `created_at`

### Phase 2.7 — Migration: create `ai_generations` table
- Columns: `id` (UUID, PK), `submission_id` (UUID, FK), `report_type` (enum: free/full), `prompt_version_id` (UUID, FK → prompts), `model_name` (varchar), `narrative_output` (jsonb), `token_count` (integer), `response_time_ms` (integer), `created_at`

### Phase 2.8 — Migration: create `reports` table
- Columns: `id` (UUID, PK), `submission_id` (UUID, FK), `report_type` (enum: free/full), `report_version` (integer, default 1), `storage_key` (varchar), `signed_url` (text), `signed_url_expires_at` (timestamp), `prompt_version_id` (UUID, FK), `template_version` (varchar), `brand_id` (UUID, FK), `scoring_config_snapshot` (jsonb), `regenerated_at` (timestamp, nullable), `regeneration_reason` (text, nullable), `regeneration_count` (integer, default 0), `created_at`

### Phase 2.9 — Migration: create `report_versions` table
- Columns: `id` (UUID, PK), `report_id` (UUID, FK → reports), `report_version` (integer), `storage_key` (varchar), `signed_url` (text), `prompt_version_id` (UUID), `template_version` (varchar), `brand_id` (UUID), `scoring_config_snapshot` (jsonb), `created_at`
- This table stores every prior version — nothing deleted

### Phase 2.10 — Migration: create `payments` table
- Columns: `id` (UUID, PK), `submission_id` (UUID, FK), `provider` (varchar), `provider_payment_id` (varchar, unique), `amount` (decimal), `currency` (varchar, default 'USD'), `status` (enum: succeeded/failed/pending), `raw_payload` (jsonb), `created_at`

### Phase 2.11 — Migration: create `webhook_logs` table
- Columns: `id` (UUID, PK), `webhook_type` (enum: assessment/payment), `status` (enum: accepted/rejected), `raw_payload` (jsonb), `error_message` (text, nullable), `ip_address` (varchar), `created_at`

### Phase 2.12 — Migration: create `crm_sync_logs` table
- Columns: `id` (UUID, PK), `submission_id` (UUID, FK), `action` (varchar), `request_payload` (jsonb), `response_status` (integer), `response_body` (jsonb), `success` (boolean), `error_message` (text, nullable), `created_at`

### Phase 2.13 — Migration: create `review_queue` table
- Columns: `id` (UUID, PK), `item_type` (enum: failed_job/rejected_webhook/regeneration/ai_error/delivery_failure/payment_issue), `reference_id` (UUID, nullable), `assessment_key` (varchar, nullable), `brand_id` (UUID, nullable), `error_message` (text), `stack_trace` (text, nullable), `raw_data` (jsonb), `status` (enum: new/reviewed/resolved), `resolved_by` (varchar, nullable), `resolution_note` (text, nullable), `created_at`, `resolved_at` (timestamp, nullable)

### Phase 2.14 — Migration: create `admin_users` table
- Columns: `id` (UUID, PK), `email` (varchar, unique), `password_hash` (varchar), `name` (varchar), `role` (enum: admin/viewer), `last_login` (timestamp, nullable), `created_at`, `updated_at`

### Phase 2.15 — Migration: create `system_logs` table
- Columns: `id` (UUID, PK), `log_type` (varchar), `submission_id` (UUID, nullable), `action` (varchar), `details` (jsonb), `created_at`
- Index on `log_type`, `submission_id`, `created_at`

### Phase 2.16 — Seed: default admin user
- Insert admin user with email from `ADMIN_DEFAULT_EMAIL`, bcrypt-hashed password from `ADMIN_DEFAULT_PASSWORD`

### Phase 2.17 — Seed: default brand (SafeRelate)
- Insert brand with name "SafeRelate", placeholder colours (#1a1a2e, #e94560), empty CTA arrays

### Phase 2.18 — Create database model helpers
- Create `src/models/index.js` — exports Knex instance
- Create individual model files: `src/models/submissions.js`, `src/models/assessmentConfigs.js`, `src/models/prompts.js`, `src/models/brands.js`, `src/models/reports.js`, `src/models/payments.js`, `src/models/reviewQueue.js`, `src/models/adminUsers.js`
- Each model file exports CRUD functions: `findById`, `findAll`, `create`, `update`, `findByField`

---

## Milestone 03 — Webhook Intake & Validation

### Phase 3.1 — Create webhook auth middleware
- Create `src/middleware/webhookAuth.js`
- Extract `Authorization: Bearer <token>` from header
- Compare against `config.webhook.bearerToken`
- Reject with 401 if invalid, log attempt

### Phase 3.2 — Create assessment webhook schema
- Create `src/routes/webhook/schemas.js`
- Define Zod schema for assessment payload: `assessment_key` (string, required), `user_email` (email), `first_name` (string), `last_name` (string), `consent_status` (boolean, must be true), `question_*` (dynamic)

### Phase 3.3 — Create assessment webhook route
- Create `src/routes/webhook/assessment.js`
- `POST /api/webhook/assessment`
- Apply webhookAuth middleware
- Validate payload with Zod schema
- On validation fail: log to `webhook_logs` (rejected), add to `review_queue`, return 400
- On success: store in `submissions` (status: submitted), queue BullMQ job, return 202

### Phase 3.4 — Create payment webhook route
- Create `src/routes/webhook/payment.js`
- `POST /api/webhook/payment`
- Verify HMAC signature (Stripe: `stripe.webhooks.constructEvent`)
- Check `payment_status === "succeeded"`
- Check for duplicate via `provider_payment_id`
- Store in `payments` table
- Queue full report generation job
- On any failure: log + surface in review queue

### Phase 3.5 — Create webhook router
- Create `src/routes/webhook/index.js`
- Mount assessment and payment routes
- Mount in main app: `app.use('/api/webhook', webhookRouter)`

### Phase 3.6 — Create error handling middleware
- Create `src/middleware/errorHandler.js`
- Global Express error handler
- Log error with winston
- Return standardized error response: `{ error: message, code, timestamp }`
- Don't leak stack traces in production

---

## Milestone 04 — BullMQ Queue Setup

### Phase 4.1 — Create Redis connection
- Create `src/services/queue/connection.js`
- Create ioredis connection using config
- Handle connection errors with logging
- Export connection instance

### Phase 4.2 — Create report generation queue
- Create `src/services/queue/reportQueue.js`
- Define `report-generation` queue using BullMQ
- Export `addReportJob(submissionId, reportType)` function
- Set default job options: attempts, backoff (exponential)

### Phase 4.3 — Create report generation worker
- Create `src/services/queue/reportWorker.js`
- Define BullMQ worker for `report-generation` queue
- Worker function skeleton: load submission → score → AI → render → upload → CRM sync
- Add event handlers: `completed`, `failed`, `error`
- On failure after all retries: update submission status to 'failed', add to review queue

### Phase 4.4 — Start worker on app boot
- In `src/index.js`, import and start the worker
- Log worker started message
- Graceful shutdown: close worker on SIGTERM/SIGINT

---

## Milestone 05 — Scoring Engine

### Phase 5.1 — Create scoring engine service
- Create `src/services/scoring/engine.js`
- Export `scoreSubmission(submission, assessmentConfig)` function
- Implement: iterate questions, map to dimensions, apply weights, handle reverse scoring
- Return: `{ total_score, subscale_scores, score_bands, recommendation_category, pattern_flags }`

### Phase 5.2 — Create score band resolver
- Create `src/services/scoring/bands.js`
- Export `resolveBands(subscaleScores, scoreBandsConfig)` function
- Map each subscale score to its band label based on configured ranges

### Phase 5.3 — Create recommendation resolver
- Create `src/services/scoring/recommendations.js`
- Export `resolveRecommendation(subscaleScores, scoreBands, rules)` function
- Apply conditional logic from recommendation_rules config
- Return recommendation category string

### Phase 5.4 — Create pattern detector
- Create `src/services/scoring/patterns.js`
- Export `detectPatterns(subscaleScores, scoreBands)` function
- Detect notable cross-dimension patterns (e.g., all-high, all-low, mixed extremes)
- Return array of pattern flag strings

### Phase 5.5 — Create scoring integration in worker
- In `src/services/queue/reportWorker.js`, import scoring engine
- After loading submission + config: call `scoreSubmission()`
- Store result in `scoring_results` table
- Pass scoring output to next step (AI generation)

---

## Milestone 06 — AI Narrative Engine

### Phase 6.1 — Create OpenAI client wrapper
- Create `src/services/ai/client.js`
- Initialize OpenAI SDK with API key from config
- Export client instance

### Phase 6.2 — Create free report JSON schema
- Create `src/services/ai/schemas/freeReport.js`
- Define JSON schema for free report output: `summary` (string), `strengths` (array), `key_insight` (string), `next_step` (string)
- Export schema object

### Phase 6.3 — Create full report JSON schema
- Create `src/services/ai/schemas/fullReport.js`
- Define JSON schema for all 12 sections: `summary`, `strengths`, `challenges`, `regulation_style`, `behavioural_tendencies`, `emotional_triggers`, `relational_impact`, `risk_indicators`, `growth_areas`, `recommendations`, `program_matching`, `disclaimer`
- Each field typed (string or array of strings)
- Export schema object

### Phase 6.4 — Create narrative generator service
- Create `src/services/ai/narrativeGenerator.js`
- Export `generateNarrative(scoringOutput, assessmentConfig, reportType)` function
- Load prompt from DB by `prompt_version_id`
- Build messages array: system (prompt), user (scoring data + user name)
- Call OpenAI with `response_format: { type: "json_schema", schema }` (Structured Outputs)
- Parse and validate response
- Store generation record in `ai_generations` table
- Return structured narrative JSON

### Phase 6.5 — Create AI error handler
- Create `src/services/ai/errorHandler.js`
- Handle: timeout, rate limit, schema validation failure, empty response
- Classify errors as retryable vs non-retryable
- Log full error details for review queue

### Phase 6.6 — Integrate AI into worker
- In report worker, after scoring: call `generateNarrative()`
- Pass result to next step (HTML rendering)

---

## Milestone 07 — Modular Chart Blocks

### Phase 7.1 — Create chart registry
- Create `src/charts/registry.js`
- Export `registerChart(type, rendererFn)` and `getChartRenderer(type)` functions
- Store renderers in a Map

### Phase 7.2 — Create Donut chart renderer
- Create `src/charts/donut.js`
- Export function that takes `{ label, value, max, color }` and returns SVG/HTML string
- Register in registry

### Phase 7.3 — Create Bar chart renderer
- Create `src/charts/bar.js`
- Same pattern — returns horizontal bar SVG/HTML
- Register in registry

### Phase 7.4 — Create Radar chart renderer
- Create `src/charts/radar.js`
- Returns radar/spider chart SVG/HTML
- Register in registry

### Phase 7.5 — Create Pie, Column, Nightingale renderers
- Create `src/charts/pie.js`, `src/charts/column.js`, `src/charts/nightingale.js`
- Each follows same pattern
- Register all in registry

### Phase 7.6 — Create chart block resolver
- Create `src/charts/blockResolver.js`
- Export `renderChartBlock(sectionId, chartConfig, data)` function
- Look up chart type from `chartConfig` for the given section
- Call the registered renderer
- Return rendered HTML string

### Phase 7.7 — Auto-register all charts on startup
- Create `src/charts/index.js`
- Import and register all chart modules
- Export registry and blockResolver

---

## Milestone 08 — PDF Generation Pipeline

### Phase 8.1 — Create SafeRelate brand template (free)
- Create `src/templates/saferelate/template_free.html`
- Single-page layout: logo, user name, summary narrative, strengths, key insight, CTA
- Use Handlebars variables: `{{logo_url}}`, `{{user_name}}`, `{{summary}}`, etc.
- Use CSS variables for brand colours: `var(--primary)`, `var(--secondary)`

### Phase 8.2 — Create SafeRelate brand template (full)
- Create `src/templates/saferelate/template_full.html`
- 3-page layout with all 12 sections
- Include chart block placeholders: `{{{chart_dimensions}}}`, `{{{chart_risk}}}`
- Page break markers for Puppeteer
- Header/footer with logo + page number

### Phase 8.3 — Create SafeRelate brand CSS
- Create `src/templates/saferelate/brand.css`
- Define CSS variables from brand config
- Typography, spacing, section headers, chart containers, CTA button styles

### Phase 8.4 — Create HTML renderer service
- Create `src/services/pdf/htmlRenderer.js`
- Export `renderReportHtml(reportType, brandConfig, narrative, scoringOutput, chartConfig)` function
- Load correct template (free or full) for the brand
- Compile with Handlebars: inject user data, narrative sections, brand values
- Render chart blocks via chartBlockResolver
- Return complete HTML string

### Phase 8.5 — Create Puppeteer PDF service
- Create `src/services/pdf/pdfGenerator.js`
- Export `generatePdf(htmlString, options)` function
- Launch Puppeteer (headless Chromium)
- Set page content to rendered HTML
- Wait for charts to render (networkidle0 or manual wait)
- Print to PDF with A4 dimensions, margins, print background
- Return PDF buffer
- Close browser instance

### Phase 8.6 — Create DigitalOcean Spaces upload service
- Create `src/services/storage/spaces.js`
- Export `uploadPdf(buffer, key)` — uploads to Spaces bucket
- Export `generateSignedUrl(key, expiryDays)` — returns pre-signed URL
- Export `refreshSignedUrl(key)` — generates new URL for expired links
- Use `@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`

### Phase 8.7 — Create report storage service
- Create `src/services/pdf/reportStorage.js`
- Export `storeReport(submissionId, reportType, pdfBuffer, metadata)` function
- Upload PDF to Spaces
- Generate pre-signed URL
- Insert record into `reports` table
- Return report record with signed URL

### Phase 8.8 — Integrate PDF pipeline into worker
- In report worker, after AI generation: call `renderReportHtml()` → `generatePdf()` → `storeReport()`
- Update submission status to 'generated'

---

## Milestone 09 — FluentCRM Integration

### Phase 9.1 — Create CRM API client
- Create `src/services/crm/client.js`
- Export axios/fetch wrapper preconfigured with FluentCRM base URL + auth headers
- Methods: `get(path)`, `post(path, data)`, `put(path, data)`

### Phase 9.2 — Create contact sync service
- Create `src/services/crm/contactSync.js`
- Export `syncContact(submission, scoringOutput, reportUrl, reportType)` function
- Create or update FluentCRM contact by email
- Set custom fields: assessment_name, last_assessment_date, total_score, primary_band, report_type, recommendation, free_report_url / full_report_url

### Phase 9.3 — Create tag service
- Create `src/services/crm/tags.js`
- Export `applyTags(contactId, tags)` function
- Export `getTagsForSubmission(scoringOutput, reportType)` — returns array of tag names based on report type + recommendation category
- Tags: assessment-completed, free-report-ready, paid-report-ready, full-report-delivered, free-talk-invite, program-recommended, therapy-recommended

### Phase 9.4 — Create CRM sync logger
- Create `src/services/crm/syncLogger.js`
- Log every CRM API call to `crm_sync_logs` table
- Record: request payload, response status, success/failure, error message

### Phase 9.5 — Integrate CRM sync into worker
- In report worker, after PDF stored: call `syncContact()` + `applyTags()`
- Update submission status to 'sent'
- If CRM sync fails: log error, add to review queue (report is still generated)

---

## Milestone 10 — Admin Portal API — Authentication

### Phase 10.1 — Create admin auth middleware
- Create `src/middleware/adminAuth.js`
- Extract JWT from Authorization header or cookie
- Verify with `jsonwebtoken`
- Attach `req.adminUser` with decoded payload
- Reject 401 if missing/invalid/expired

### Phase 10.2 — Create login route
- Create `src/routes/admin/auth.js`
- `POST /api/admin/auth/login` — validate email + bcrypt compare password → issue JWT
- `POST /api/admin/auth/logout` — clear cookie if using cookies
- `GET /api/admin/auth/me` — return current admin user info (protected)

### Phase 10.3 — Create admin router
- Create `src/routes/admin/index.js`
- Apply `adminAuth` middleware to all admin routes
- Mount in main app: `app.use('/api/admin', adminRouter)`

---

## Milestone 11 — Admin Portal API — Assessment Config CRUD

### Phase 11.1 — Create assessment config routes
- Create `src/routes/admin/assessments.js`
- `GET /api/admin/assessments` — list all configs with pagination + filters
- `GET /api/admin/assessments/:id` — get single config with all fields
- `PUT /api/admin/assessments/:id` — update config fields
- `POST /api/admin/assessments` — create new assessment config

### Phase 11.2 — Create clone assessment route
- Add `POST /api/admin/assessments/:id/clone` to assessments router
- Duplicate full config into new record
- Set `clone_origin`, `cloned_by`
- Return new assessment config

---

## Milestone 12 — Admin Portal API — Prompt Manager

### Phase 12.1 — Create prompt routes
- Create `src/routes/admin/prompts.js`
- `GET /api/admin/prompts` — list all prompt types with active version info
- `GET /api/admin/prompts/:type` — get prompt type with all versions
- `POST /api/admin/prompts/:type` — save new version (status: draft)
- `POST /api/admin/prompts/:type/activate` — set a version as active
- `POST /api/admin/prompts/:type/rollback` — activate a prior version

---

## Milestone 13 — Admin Portal API — Brand Management

### Phase 13.1 — Create brand routes
- Create `src/routes/admin/brands.js`
- `GET /api/admin/brands` — list all brands
- `GET /api/admin/brands/:id` — get brand config
- `POST /api/admin/brands` — create new brand
- `PUT /api/admin/brands/:id` — update brand config

### Phase 13.2 — Create logo upload endpoint
- Add `POST /api/admin/brands/:id/logo` to brands router
- Use multer for file upload
- Upload to Spaces: `brands/{brand_id}/logo.png`
- Update brand record with logo URL

---

## Milestone 14 — Admin Portal API — Client & Reports Backend

### Phase 14.1 — Create client routes
- Create `src/routes/admin/clients.js`
- `GET /api/admin/clients` — search + filter (name, email, assessment, brand, date range, status)
- `GET /api/admin/clients/:id` — full client record with all submissions, reports, versions

### Phase 14.2 — Create report management routes
- Create `src/routes/admin/reports.js`
- `GET /api/admin/reports/:id` — get report detail with version history
- `POST /api/admin/reports/:id/regenerate` — queue regeneration job with scope + config version + reason
- `POST /api/admin/reports/:id/refresh-url` — generate new pre-signed URL for expired report

### Phase 14.3 — Create regeneration worker logic
- Add regeneration handling to report worker (or create separate worker)
- Re-score → re-generate AI → re-render HTML → new PDF → upload → store new version
- Retain prior version in `report_versions` table
- Increment `report_version`, update `regeneration_count`

---

## Milestone 15 — Admin Portal API — Review Queue & Logs

### Phase 15.1 — Create review queue routes
- Create `src/routes/admin/reviewQueue.js`
- `GET /api/admin/review-queue` — list items with filters (type, status, date range, assessment, brand)
- `GET /api/admin/review-queue/:id` — item detail with full error info + raw payload
- `POST /api/admin/review-queue/:id/reprocess` — re-queue the failed job
- `PUT /api/admin/review-queue/:id/resolve` — mark as reviewed/resolved with note

### Phase 15.2 — Create logs routes
- Create `src/routes/admin/logs.js`
- `GET /api/admin/logs` — paginated system logs with filters
- `GET /api/admin/logs/export` — export as CSV or JSON

### Phase 15.3 — Create dashboard stats route
- Create `src/routes/admin/dashboard.js`
- `GET /api/admin/dashboard` — return aggregate stats:
  - Total submissions (today / this week / all time)
  - Reports generated (free / full)
  - Failed jobs count
  - Review queue items (new)
  - Revenue (total full report purchases)

---

## Milestone 16 — Admin Portal Frontend

### Phase 16.1 — Create admin frontend skeleton
- Create `src/admin/` directory
- Set up a basic single-page app (vanilla HTML/JS or React — depending on preference)
- Create layout: sidebar navigation + main content area
- Pages: Dashboard, Assessments, Prompts, Brands, Clients, Review Queue, Logs

### Phase 16.2 — Build Login page
- Email + password form
- Call `POST /api/admin/auth/login`
- Store JWT, redirect to dashboard

### Phase 16.3 — Build Dashboard page
- Cards: total submissions, reports generated, failed jobs, revenue
- Call `GET /api/admin/dashboard`

### Phase 16.4 — Build Assessments list page
- Table: name, brand, status, last modified
- Search + filter controls
- "Clone" and "New" buttons

### Phase 16.5 — Build Assessment config editor page
- Form with all config fields (dimensions, question_map, weights, etc.)
- Dropdowns for prompt version, template, brand
- Save + Clone buttons

### Phase 16.6 — Build Prompt Manager page
- Tabs for each prompt type
- Text editor for prompt content
- Version history sidebar
- "Save Draft", "Activate", "Rollback" buttons
- "Linked Assessments" display

### Phase 16.7 — Build Brands page
- Card grid of brands with logo + colour swatch
- Brand editor form: logo upload, colour pickers, CTA fields, footer text

### Phase 16.8 — Build Clients page
- Search bar + filter controls
- Results table with pagination
- Click row → client detail view

### Phase 16.9 — Build Client detail page
- Submission history table
- Per-submission: report list with version history
- Buttons: "Refresh Link", "Regenerate Report", "View PDF"
- Expandable raw payload viewer

### Phase 16.10 — Build Review Queue page
- Filterable table of queue items
- Click row → detail panel with error info + raw data
- Action buttons: "Reprocess", "Mark Resolved"

### Phase 16.11 — Build Logs page
- Paginated log table with filters
- Export button

---

## Milestone 17 — Docker & Deployment

### Phase 17.1 — Create Dockerfile
- Node.js 20 base image
- Install Chromium dependencies for Puppeteer
- Copy package.json, install deps
- Copy src/
- Expose port
- CMD: `node src/index.js`

### Phase 17.2 — Create docker-compose.yml
- Services: `api` (Node.js app), `db` (PostgreSQL), `redis` (Redis)
- Volumes for persistent data (postgres, redis)
- Environment variables from .env
- Health checks for all services
- Network: internal bridge

### Phase 17.3 — Create Nginx config
- Create `nginx/nginx.conf`
- Reverse proxy to Node.js app
- SSL configuration placeholder (Let's Encrypt)
- Security headers
- Rate limiting at proxy level

### Phase 17.4 — Create deployment script
- Create `scripts/deploy.sh`
- Pull latest code
- Run `docker-compose build`
- Run migrations: `docker exec api npx knex migrate:latest`
- Restart services: `docker-compose up -d`
- Health check verification

### Phase 17.5 — Create SSL setup script
- Create `scripts/ssl-setup.sh`
- Install Certbot
- Obtain SSL certificate for domain
- Configure Nginx with SSL
- Set up auto-renewal cron

---

## Milestone 18 — End-to-End Testing

### Phase 18.1 — Create webhook endpoint tests
- Test valid assessment payload → 202 + job queued
- Test missing fields → 400
- Test invalid auth → 401
- Test invalid consent → 400

### Phase 18.2 — Create scoring engine tests
- Test basic scoring with known inputs → expected outputs
- Test reverse scoring
- Test weighted scoring
- Test score band resolution
- Test recommendation rules

### Phase 18.3 — Create AI narrative tests
- Test prompt loading by version ID
- Test OpenAI call with mocked response
- Test schema validation of AI output

### Phase 18.4 — Create PDF pipeline tests
- Test HTML rendering with template + data
- Test Puppeteer PDF generation (integration test)
- Test Spaces upload + signed URL generation (mocked)

### Phase 18.5 — Create payment flow tests
- Test valid payment webhook → full report queued
- Test invalid signature → rejected
- Test duplicate payment → deduplicated
- Test failed payment status → rejected

### Phase 18.6 — Create admin API tests
- Test login flow
- Test CRUD for assessments, prompts, brands
- Test clone assessment
- Test regeneration endpoint
- Test review queue operations

### Phase 18.7 — Create CRM sync tests
- Test contact creation/update
- Test tag application
- Test custom field setting
- Test sync failure handling

### Phase 18.8 — Full pipeline integration test
- Submit mock assessment → verify: submission stored → scored → AI generated → PDF created → CRM synced → status = 'sent'
- Verify all database records created correctly
- Verify review queue empty (no errors)
