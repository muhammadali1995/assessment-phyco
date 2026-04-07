
Meetings
Notion AI
Inbox
Library
Assessment Generation Platform — Full Technical Spec
📋 NRM / SafeRelate Platform — Version 1.1 | Stack: Node.js, PostgreSQL, Redis, OpenAI, Puppeteer | Infra: DigitalOcean, Docker, Nginx, SSL | Updated April 2026 — admin features, prompt management, report versioning, security, chart modularity
01 — Project Overview
The Assessment Generation Platform is a server-side Report Engine that receives clinical assessment submissions from Fluent Forms via webhook, applies a configurable scoring engine, generates personalised AI-powered narrative reports in two tiers (Free and Full), stores all results with full audit trails, and syncs data back into FluentCRM for automated email sequences and CRM tagging.
The platform is built to be multi-brand — each brand (e.g. SafeRelate) has its own HTML report template, logo, colour scheme, and CTA configuration. The same engine powers all brands from a single codebase.
The Admin Portal is a first-class feature of this platform. It enables non-technical operators to clone assessments, manage prompts, track clients and reports, trigger report regeneration, switch brands, and review system health — all without developer intervention.
System Purpose
Users complete a structured NRM-based assessment via Fluent Forms on a WordPress site. The system receives their submission, scores it according to a clinically defined framework, generates a personalised AI narrative report, and delivers it via email. Higher-tier full reports are available as a paid upsell at $5.00.
Core Pipeline
Fluent Forms submission (WordPress frontend)
Webhook fired to the Report API
Payload validated and stored
Scoring engine applies config-based rules (loaded from database)
AI narrative generated via OpenAI Structured Outputs (using versioned, admin-editable prompts)
HTML report rendered via modular chart blocks and converted to PDF via Puppeteer
PDF stored and secure pre-signed expiring URL generated (not permanently public)
FluentCRM synced with scores, tags, and report URL
FluentCRM triggers email automation (free report delivery, upsell sequence)
02 — Tech Stack
Layer
Technology
Backend Framework
Node.js + Express
Database
PostgreSQL
Queue & Caching
Redis + BullMQ (background job processing)
AI API
OpenAI (Structured Outputs — JSON schema enforcement)
PDF Generation
HTML templates + Puppeteer (headless Chrome)
CRM Integration
FluentCRM REST API (/wp-json/fluent-crm/v2)
Form Integration
Fluent Forms webhooks (WordPress)
Infrastructure
DigitalOcean Droplet + Docker + Nginx + SSL (Let's Encrypt)
OS
Ubuntu Linux
Secret Management
Environment variables + secure secrets store
File Storage
DigitalOcean Spaces (PDFs via pre-signed expiring URLs)
03 — System Architecture
Overview
The system is a private Node.js API deployed on DigitalOcean, containerised with Docker, and served through Nginx with SSL. It operates independently of the WordPress/Fluent Forms frontend — all communication is server-to-server via authenticated webhooks and REST API calls.
Key Architectural Decisions
Assessment Config Stored in Database (not files)
All assessment configurations — question mappings, scoring weights, dimensions, score bands, recommendation rules, prompt version references, template references, and brand references — are stored in PostgreSQL. Configs are fully editable from the Admin Portal without developer intervention or code deployments. This supports rapid onboarding of new programs and assessment variants as they arrive from clients.
Assessment Clone Workflow
The Admin Portal exposes a full Clone Assessment workflow designed for rapid implementation of new program variants:
Select any existing assessment as the base
Duplicate its entire config (questions, scoring rules, dimension mappings, prompt version mapping, template mapping) into a new named assessment
Edit any field of the cloned config within the admin UI: question weights, score bands, recommendation rules, linked prompt version, linked template, linked brand
Save and activate the new assessment — no code changes, no deployments required
Full audit trail: clone origin, who cloned it, and when
Two-Layer Report Generation
Layer 1 (Deterministic): The system computes total scores, subscale scores, score bands, and recommendation categories using pure logic — no AI involved. This ensures the clinical framework always controls the outcome.
Layer 2 (AI Narrative): OpenAI receives the structured scoring output and generates the personalised written narrative using a versioned prompt loaded from the Prompt Manager. The AI only controls language — never logic or clinical decisions.
Modular Chart Blocks in Report Renderer
The PDF report renderer uses a modular chart block system — each section in a report template specifies its chart type via the assessment's chart_config field. Supported chart types: Donut, Pie, Bar, Column, Radar, Nightingale. Each chart type is a self-contained renderer module. New chart types can be added as new modules without changing report or scoring logic. Different assessments can use entirely different chart configurations.
HTML-to-PDF via Puppeteer
Reports are generated as branded HTML templates, injected with dynamic data (name, scores, narrative text, modular chart blocks), and rendered to PDF by Puppeteer (headless Chrome). The rendered PDF is stored in DigitalOcean Spaces and accessed via pre-signed expiring URLs — never permanently public links.
Background Job Processing
All report generation happens asynchronously via BullMQ queues backed by Redis. The webhook endpoint immediately acknowledges receipt and queues the job. Failed jobs are retried automatically with exponential backoff. All job outcomes are logged and surfaced in the Admin Review Queue.
04 — Assessment Intake & Webhook
Fluent Forms fires a POST webhook to the API upon form submission. The endpoint validates the payload, maps fields to a structured schema, and queues the job for processing.
Method + Path
POST /api/webhook/assessment
Auth
Bearer token (server-to-server, configured in Fluent Forms webhook settings)
Validation
Rejects incomplete or malformed payloads with 400 error. All rejected payloads are logged and surfaced in the Admin Review Queue.
Payload Schema
Every submission is mapped into a structured schema before processing:
assessment_key — identifies which assessment config to load from the database
user_email, first_name, last_name — user identity
consent_status — must be confirmed before processing
question_1 through question_N — all answer values
05 — Scoring Engine
The scoring engine is rule-based and fully configurable. Each assessment has a config record in PostgreSQL defining all scoring logic. The core engine code never changes — configs are added, edited, or cloned via the Admin Portal without code deployments.
Config Structure (per assessment — stored in DB, editable in Admin)
Config Key
Description
dimensions
Named subscales (e.g. Emotional Awareness, Regulation, Behavioural Response)
question_map
Which question maps to which dimension
weights
Scoring weight per question (default 1.0)
reverse_scored
List of question IDs that are reverse-scored
score_bands
Score ranges mapped to band labels (e.g. Low, Moderate, High)
recommendation_rules
Conditional logic mapping score profiles to recommendation pathways
prompt_version_id
Which prompt version to use for AI narrative generation
template_id
Which brand HTML template to use for PDF generation
brand_id
Which brand config to apply (logo, colours, CTAs, tone)
chart_config
Chart type per report section (Donut, Bar, Radar, Nightingale, etc.)
Scoring Output (Structured JSON)
total_score — overall weighted score
subscale_scores — score per dimension
score_bands — band label per dimension (e.g. Moderate, Low)
recommendation_category — pathway selected (e.g. group-program, individual-therapy)
pattern_flags — notable patterns detected across dimensions
06 — AI Narrative Engine
The AI narrative engine receives the structured scoring output and generates personalised written content using OpenAI Structured Outputs — enforcing a strict JSON schema to prevent missing sections or hallucinated fields. All prompts are loaded from the Prompt Manager at runtime by prompt_version_id — never hardcoded.
Input Sent to OpenAI
Assessment type and name
User first name
All dimension scores and band labels
Recommendation category and pathway
Dimension labels and interpretation intent
Active prompt (loaded at runtime from Prompt Manager by prompt_version_id)
Output Schema
Field
Description
summary
2-paragraph profile narrative (non-diagnostic, supportive tone)
strengths
Array of strength statements based on score profile
challenges
Array of challenge statements based on score profile
regulation_style
One-paragraph regulation style description
behavioural_tendencies
Array of identified behavioural patterns
emotional_triggers
Array of likely trigger scenarios
relational_impact
Narrative paragraph on relational effects
risk_indicators
Array of risk flags if unaddressed
growth_areas
Array of development opportunities
recommendations
Array of actionable recommendation statements
program_matching
Matched program(s) based on recommendation_category
disclaimer
Standard non-diagnostic disclaimer text
Audit Trail (Stored per Report)
prompt_version_id — exact prompt version used for this generation
model_name — OpenAI model (e.g. gpt-4o)
generation_timestamp — exact time of generation
report_version — version of report template used
regeneration_count — total number of regenerations for this submission
regeneration_reason — reason for most recent regeneration (prompt edit / template edit / scoring change / brand change)
07 — Prompt Manager (Admin Portal)
All AI prompts are stored in PostgreSQL and editable from the Admin Portal without developer intervention. Every prompt type has full version history. No prompt is ever hardcoded in the codebase.
Prompt Types
Prompt Type
Purpose
Free Report Prompt
System prompt for generating the free (1-page) report narrative
Full Report Prompt
System prompt for generating the full (3-page) report narrative
Recommendation Prompt
Governs recommendation and program matching language
Brand Tone / CTA Prompt
Brand-specific tone of voice and CTA language injected per brand
Prompt Version Control
Every edit creates a new version — prior versions are never deleted
Each version stores: version number, created timestamp, created by (admin user), and change notes
Active version per prompt is set explicitly by admin — not auto-promoted on save
Assessment configs reference a specific prompt_version_id — the assessment uses exactly that version until explicitly updated
Prompt history is viewable in the Admin Portal
Admin Prompt Editor
Plain text / rich text editor per prompt type
Save as draft before promoting to active
Activate a specific version with one click
Roll back to any prior version
View which assessments are currently linked to which prompt version
08 — Report Types
Report Type
Scope
Delivery
Free Report
1 page / 150–250 words
Auto-delivered to all users after completion
Full Report
3 pages / 12 sections
Paid upsell at $5.00 — triggered only after confirmed payment webhook
Full Report — 12 Sections
Profile Summary
Dimensions Breakdown (modular chart block — chart type configured per assessment)
Core Patterns Identified
Behavioural Tendencies
Emotional Triggers
Relational Impact
Risk Indicators (modular chart block — chart type configured per assessment)
Strengths
Growth Areas
Recommendations
Program Matching
Next Steps (CTA buttons + Important Note)
Modular Chart Blocks
Each chart section in a report template specifies its chart type via chart_config in the assessment config. Supported types: Donut, Pie, Bar, Column, Radar, Nightingale. Chart blocks are self-contained renderer modules registered at startup. A new chart type is added by writing a new renderer module — no changes to report, scoring, or PDF pipeline logic required.
09 — Full Report Purchase Flow
The paid full-report flow is strictly sequenced to prevent any delivery before payment is confirmed.
End-to-End Flow
User receives upsell email triggered by paid-report-ready tag in FluentCRM
User clicks payment link and completes $5.00 payment
Payment provider fires a webhook to POST /api/webhook/payment
API validates webhook signature — rejects and logs any invalid/unsigned payloads
API confirms payment_status === "succeeded" — only on this condition does processing continue
Full report generation job is queued for this submission
Full report generated: AI narrative (using full-report prompt version) + PDF via Puppeteer
PDF stored in DigitalOcean Spaces; pre-signed expiring URL generated
FluentCRM contact updated: full_report_url set, full-report-delivered tag applied
FluentCRM triggers "Full Report Ready" email with secure signed URL
Payment Safety Rules
Full report generation is only triggered by a confirmed, signature-verified payment webhook — never by a CRM tag, form event, or admin action without payment confirmation
paid-report-ready tag is applied before payment to enable the upsell email sequence — it does not indicate payment has occurred
full-report-delivered tag is only applied after confirmed payment + successful PDF generation + CRM URL update — in that order
Duplicate payment webhooks are detected and deduplicated
Failed payment webhooks (invalid signature, wrong status) are logged and surfaced in the Admin Review Queue
If payment is confirmed but report generation fails, the failure is logged in the Review Queue and admin is alerted
10 — PDF Generation (HTML → Puppeteer)
Reports are generated by injecting dynamic data into branded HTML templates and rendered to PDF via Puppeteer (headless Chrome).
Generation Pipeline
Node.js loads assessment config from DB to determine brand, template, and chart config
Brand template (HTML/CSS) retrieved for the assessment's brand_id
Dynamic data injected: user name, scores, narrative sections, chart data
Modular chart blocks rendered per section based on chart_config
Puppeteer loads the HTML in headless Chrome and waits for all chart blocks to render
Puppeteer prints to PDF with correct page dimensions and margins
PDF saved to DigitalOcean Spaces
Pre-signed expiring URL generated and stored in PostgreSQL with full report metadata
Secure Report Access
Report PDFs are not served via permanently public URLs. All report links are pre-signed URLs with a configurable expiry (default: 7 days). Links can be refreshed on demand from the Client & Reports Backend in the Admin Portal. This prevents unauthorised access to private clinical reports after a link has expired.
Multi-Brand Template System
Each brand has its own directory:
template_free.html — 1-page free report template
template_full.html — 3-page full report template
brand.css — brand-specific colours, fonts, and decorative elements
logo.png — brand logo file
assets/ — geometric/decorative images specific to the brand
11 — Brand Layer (Admin Portal — Fully Swappable)
The brand layer is fully swappable from the Admin Portal. The same assessment engine and scoring logic can produce a completely different branded report with no code changes.
Per-Brand Configuration (Admin-Editable)
Setting
Description
Logo
Upload brand logo (PNG/SVG)
Primary & secondary colours
Hex values applied to template CSS variables
CTA button labels
Per-button text labels
CTA URLs
Per-brand destination URLs for all CTA buttons
Footer / disclaimer text
Brand-specific legal and contact footer
Contact details
Brand-specific contact info displayed in reports
Template selection
Choose HTML template for free and full reports
Brand tone / CTA prompt
Optional brand-specific tone prompt injected into AI generation
Brand Switching per Assessment
Each assessment config references a brand_id
Switching brand is a single field update in the Admin Portal — no code change
After a brand switch, existing submissions can be regenerated under the new brand via the Report Regeneration feature
12 — FluentCRM Integration
After a report is generated, the system pushes contact data, scores, tags, and the report URL into FluentCRM via its REST API. FluentCRM handles all email automations.
Custom Fields Synced
Field
Value
assessment_name
Name of the completed assessment
last_assessment_date
Timestamp of submission
total_score
Overall weighted score
primary_band
Primary score band label (e.g. Moderate)
report_type
free or full
recommendation
Recommended pathway
free_report_url
Pre-signed expiring URL to free report PDF
full_report_url
Pre-signed expiring URL to full report PDF (set after confirmed payment)
Tags Applied
Tag
Trigger
assessment-completed
Every successful submission
free-report-ready
Free report PDF generated and URL stored
paid-report-ready
Upsell sequence trigger — does NOT indicate payment
full-report-delivered
Only after confirmed payment + PDF generation + CRM URL update
free-talk-invite
Based on recommendation rules
program-recommended
Group program matched
therapy-recommended
Individual therapy matched
FluentCRM Email Automations
Thank-you email — immediate on assessment-completed
Free report delivery — triggered by free-report-ready
Full report upsell ($5.00) — triggered by paid-report-ready
Full report delivery — triggered by full-report-delivered
Free talk invitation — triggered by free-talk-invite
Program nurture sequence — triggered by program-recommended
13 — Report Regeneration & Version Control (Admin Portal)
Admins can regenerate any free or full report directly from the Admin Portal after changes to prompts, templates, scoring, or branding.
Supported Regeneration Triggers
Prompt version updated (free, full, recommendation, or brand tone prompt)
HTML template updated
Brand changed (logo, colours, CTA URLs)
Scoring config changed (weights, bands, recommendation rules)
Regeneration Flow (Admin-Initiated)
Admin opens a client record in the Client & Reports Backend
Selects the specific report to regenerate
Chooses scope: free report, full report, or both
Selects which config to apply: current active config, or a specific prior version
Confirms — a new report generation job is queued
New PDF generated and stored; prior version is retained (not overwritten)
Report version number incremented; new pre-signed URL generated
CRM updated with new URL if admin selects this option
Report Version Storage
Each report record stores:
report_version — incrementing version number per submission
prompt_version_id — exact prompt version used for this generation
template_version — template version used
brand_id — brand active at time of generation
scoring_config_snapshot — point-in-time snapshot of scoring config (for auditability)
regenerated_at — timestamp of most recent regeneration
regeneration_reason — admin-entered reason for regeneration
regeneration_count — total regenerations for this submission
All prior report versions are retained in a report_versions table — nothing is permanently deleted.
14 — Client & Reports Backend (Admin Portal)
A dedicated backend dashboard for tracking all clients and all generated reports across the platform — not just CRM data.
Client Search & Filtering
Admins can search and filter by:
Name (first, last, or full name)
Email address
Assessment taken
Submission date range
Brand
Report status: submitted / processing / generated / failed / sent / upsell purchased
Client Record View
Each client record shows:
All submissions across all assessments and brands
All reports per submission, with status and version history
Per-report status: submitted → processing → generated → failed → sent → upsell purchased
Prompt version and template version used for each generation
CRM sync status per record
Ability to re-open any past submission and view the raw incoming payload
Ability to view the current report PDF and all prior versions
Link refresh: generate a new pre-signed URL for any expired report link on demand
Report Status Lifecycle
​
15 — Admin Logs & Review Queue (Admin Portal)
A dedicated Review Queue screen for monitoring, debugging, and reprocessing problem cases — beyond standard success/failure logs.
Review Queue — Items Surfaced
Item Type
Description
Failed jobs
BullMQ jobs that exhausted all retries
Rejected webhook payloads
Fluent Forms or payment webhooks that failed validation
Regeneration attempts
All admin-initiated regeneration events with outcome
AI generation errors
OpenAI API errors, timeouts, schema validation failures, empty responses
Delivery failures
CRM sync failures, email trigger failures
Payment webhook issues
Signature verification failures, duplicate payments, payment status anomalies
Review Queue Features
Filter by item type, date range, assessment, brand, or status
View full error detail per item: raw payload, error message, stack trace
Reprocess a failed job directly from the Review Queue
Mark an item as reviewed / resolved with optional note
Export logs for external debugging sessions
Standard Logs (Always Stored)
Every webhook received (accepted and rejected)
Every job: queued, started, completed, failed
Every OpenAI API call: model, prompt version ID, response time, token count
Every PDF generation attempt and outcome
Every CRM sync attempt and outcome
Every report delivery (email trigger fired)
Every report regeneration event
16 — Infrastructure & Security
Component
Detail
Hosting
DigitalOcean Droplet or App Platform
OS
Ubuntu Linux
Containerisation
Docker (all services containerised)
Reverse Proxy
Nginx
SSL/TLS
Let's Encrypt (auto-renewing)
Database
PostgreSQL (containerised)
Cache/Queue
Redis (containerised)
Storage
DigitalOcean Spaces for PDFs
Report URLs
Pre-signed expiring URLs only — never permanently public
Security
Server-to-server authentication via bearer tokens
Payment webhook signature verification (HMAC validation per payment provider)
API keys stored as environment variables — never in code
All webhook payloads validated and sanitised before processing
Audit logging for every report generation, failure, regeneration, and retry
Admin portal protected by authentication
PostgreSQL access restricted to application layer only
All report PDFs accessed via pre-signed expiring URLs — default 7-day expiry, configurable, refreshable from Admin Portal
17 — Implementation Milestones
#
Milestone
Key Deliverables
Hours
01
Server & Infrastructure
DigitalOcean setup, Docker, Nginx, SSL, PostgreSQL, Redis, Node.js skeleton, API auth
15 hrs
02
Webhook Intake & Validation
Fluent Forms + payment webhook endpoints, payload schema validation, rejection logging, BullMQ job queuing
15 hrs
03
Assessment Config in DB
DB schema for assessment configs, scoring rules, prompt version refs, template refs, brand refs; admin CRUD; clone workflow
18 hrs
04
Scoring Engine
Config-based scoring loaded from DB; dimension mapping, weights, reverse scoring, score bands, recommendation rules
18 hrs
05
Prompt Manager
DB schema for prompts + versions; admin editor for all prompt types; version history; active version selection; assessment linkage
15 hrs
06
AI Narrative Engine
OpenAI Structured Outputs, runtime prompt loading by version ID, JSON schema enforcement, audit trail per report
18 hrs
07
Modular Chart Blocks + PDF Generation
Chart renderer modules (Donut, Pie, Bar, Column, Radar, Nightingale); HTML brand templates; Puppeteer; Spaces storage; pre-signed URL generation
25 hrs
08
Full Report Purchase Flow
Payment webhook endpoint, signature verification, payment-gated generation, tag sequencing, duplicate payment deduplication
15 hrs
09
FluentCRM Integration
REST API, contact sync, custom fields, tag application, automation triggers
12 hrs
10
Admin — Brand Layer Management
Per-brand logo, colours, CTA URLs, footer/disclaimer, template selection; brand switching per assessment
10 hrs
11
Admin — Client & Reports Backend
Client search + filtering, client record view, report status lifecycle, re-open submissions, signed URL refresh on demand
18 hrs
12
Admin — Report Regeneration & Version Control
Admin-initiated regeneration, version storage, config snapshot, prior version retention, regeneration history
15 hrs
13
Admin — Logs & Review Queue
Review Queue screen, all log types, reprocess from queue, filter/search, export
12 hrs
14
End-to-End Testing & Hardening
Full pipeline tests, edge cases, retry validation, payment flow testing, security review, deployment
15 hrs
Total: 221 hours
18 — Pending Items from Client
Item
Blocks
Assessment questions & scoring logic
Milestone 04 — scoring engine config
SafeRelate brand assets (logo, hex colours, fonts)
Milestone 07 — PDF templates
Number of brands at launch
Scope of multi-brand template system
FluentCRM WordPress URL + REST API credentials
Milestone 09 — CRM integration
Payment provider choice + webhook signing details
Milestone 08 — purchase flow
Full report template (pages 2–3 confirmed ✅)
Received and confirmed
CTA button URLs per brand (confirmed ✅)
Milestone 10 — brand config
Desired signed URL expiry duration
Milestone 07 — security config
