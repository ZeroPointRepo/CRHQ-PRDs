# Lightweight CRM — Product Requirements Document

> **Version:** 3.0 (Final)  
> **Type:** AI-Agent-Executable PRD  
> **Status:** Ready for Execution  
> **Last Updated:** 2026-03-25  

---

## Table of Contents

1. [Before You Start — User Customization](#1-before-you-start--user-customization)
2. [Project Overview & Vision](#2-project-overview--vision)
3. [Technology Stack & Architecture](#3-technology-stack--architecture)
4. [Coding Best Practices](#4-coding-best-practices)
5. [Database Schema & Migrations](#5-database-schema--migrations)
6. [Core Features](#6-core-features)
   - 6.1 [Authentication & Session Management](#61-authentication--session-management)
   - 6.2 [User Management](#62-user-management)
   - 6.3 [Contacts Module](#63-contacts-module)
   - 6.4 [Companies Module](#64-companies-module)
   - 6.5 [Deals & Pipeline Module](#65-deals--pipeline-module)
   - 6.6 [Activities Module](#66-activities-module)
   - 6.7 [Meetings Module](#67-meetings-module)
   - 6.8 [Notes Module](#68-notes-module)
   - 6.9 [Attachments Module (Cloudflare R2)](#69-attachments-module-cloudflare-r2)
   - 6.10 [Email Integration (Brevo)](#610-email-integration-brevo)
   - 6.11 [API Keys & Integrations](#611-api-keys--integrations)
   - 6.12 [Settings](#612-settings)
   - 6.13 [Dashboard](#613-dashboard)
7. [API Design](#7-api-design)
8. [Security](#8-security)
9. [Milestones — Execution Plan](#9-milestones--execution-plan)
10. [AI Agent Execution Instructions](#10-ai-agent-execution-instructions)

---

# Lightweight CRM — Product Requirements Document (Draft 2)

## Sections 1–4

---

# SECTION 1: Before You Start (User Customization Q&A)

## Purpose

**STOP. Do NOT begin implementation until this section is completed.**

Before writing a single line of code, the executing agent MUST present the following questions to the user and collect answers. Each question includes a default value. **All defaults listed below are FINAL decisions — if the user skips a question, the default is used as-is. The agent MUST NOT ask questions whose answers are already specified elsewhere in this PRD.** The defaults exist so the agent can proceed without blocking; they are not placeholders.

After all answers are collected (or defaults accepted), the agent MUST echo back a summary of all chosen values and get explicit confirmation before proceeding.

---

### Q1: Company Name

> What is the name of your company? This will appear in the application header, page titles, email templates, and all branding surfaces.

- **Default:** `"Acme Corp"`
- **Used in:** Application title bar, login page, email footers, PDF exports, browser tab title, `<meta>` tags.
- **Validation:** Non-empty string, max 100 characters. No HTML or script tags. Unicode-safe (must support international characters).
- **Stored in:** `settings` table as `company_name`, and in frontend config as `APP_NAME`.

---

### Q2: Logo URL

> Do you have a logo URL (publicly accessible image)? Provide the full URL to a PNG or JPEG file. Recommended dimensions: 200x50px (landscape) or 50x50px (square).

- **Default:** `null` (system renders company name as text in a styled `<span>` with the primary brand color when no logo is provided)
- **Used in:** Sidebar header, login page, email templates, favicon derivation.
- **Validation:** Must be a valid URL ending in `.png`, `.jpg`, `.jpeg`, or `.webp` if provided. Agent should verify URL is reachable (HTTP 200) before accepting. If unreachable, warn user and fall back to default.
- **Stored in:** `settings` table as `company_logo_url`.
- **Edge case:** If the image fails to load at runtime, the frontend MUST gracefully fall back to text-based company name rendering. Use an `<img>` tag with `onError` handler.

---

### Q3: Brand Colors

> What are your primary and secondary brand colors? Provide hex codes.

- **Default:** Primary `#2563EB` (blue-600), Secondary `#1E40AF` (blue-800)
- **Used in:** Sidebar background/accent, buttons, links, active states, focus rings, email template header, loading spinners.
- **Validation:** Must be valid 6-digit hex color codes with `#` prefix. Regex: `/^#[0-9A-Fa-f]{6}$/`. Agent must verify sufficient contrast ratio (WCAG AA minimum 4.5:1) between primary color and white text — if contrast is insufficient, warn the user.
- **Stored in:** `settings` table as `brand_color_primary` and `brand_color_secondary`. Also injected into Tailwind config as CSS custom properties (`--color-primary`, `--color-secondary`).
- **Implementation note:** Colors are applied via CSS custom properties at the `:root` level. The Tailwind config extends the color palette to reference these custom properties, allowing dynamic theming without rebuild.

---

### Q4: Industry

> What industry is your company in? This helps configure default labels, activity types, and deal stage terminology.

- **Default:** `"Technology / SaaS"`
- **Options (suggestions, not exhaustive):** Technology/SaaS, Professional Services, Real Estate, Financial Services, Healthcare, Manufacturing, Retail/E-commerce, Education, Non-Profit, Consulting, Marketing/Agency, Construction, Legal, Other.
- **Used in:** Seeding default pipeline stages, default custom field suggestions, help text phrasing, demo/seed data generation.
- **Stored in:** `settings` table as `industry`.
- **Behavior:** The industry selection triggers industry-specific defaults for Q6 (deal stages) and Q8/Q9/Q10 (custom fields). For example:
  - **Real Estate:** Deal stages become "Lead, Viewing Scheduled, Offer Made, Under Contract, Closed"
  - **SaaS:** Deal stages become "Lead, Qualified, Proposal, Negotiation, Closed Won, Closed Lost"
  - **Professional Services:** Deal stages become "Inquiry, Scoping, Proposal Sent, Engagement, Delivered, Closed"

---

### Q5: Contact & Company Types

> What types of contacts and companies do you manage? This determines default tags, categories, and filter options.

- **Default:** Contact types: `["Lead", "Customer", "Partner", "Vendor", "Other"]`. Company types: `["Prospect", "Customer", "Partner", "Vendor", "Other"]`.
- **Used in:** Contact and company creation forms as dropdown/select options. Filter sidebar. Bulk operations.
- **Validation:** Each type must be a non-empty string, max 50 characters. Minimum 1 type for each. Maximum 20 types for each. No duplicates (case-insensitive comparison). No special characters except spaces, hyphens, and ampersands.
- **Stored in:** `settings` table as JSON arrays `contact_types` and `company_types`.
- **Edge case:** If user later removes a type that is already assigned to existing records, those records retain their type value but it displays with a "(Deprecated)" badge. The type is hidden from new record creation dropdowns but remains filterable.

---

### Q6: Deal/Pipeline Tracking

> Do you need deal/pipeline tracking? If yes, what are your pipeline stages? Deals track potential revenue through a pipeline from first contact to close.

- **Default:** Yes, with stages: `["Lead", "Qualified", "Proposal", "Negotiation", "Closed Won", "Closed Lost"]`
- **Used in:** Deals module (Kanban board view and list view), pipeline reports, dashboard metrics, forecasting.
- **Validation:** If enabled, minimum 2 stages required, maximum 15. Each stage name max 50 characters. Must include at least one "closed" stage. The last two stages MUST be designated as "closed-won" and "closed-lost" (agent should ask user to confirm which stages are win/loss). Stages are ordered — the order provided is the display/progression order. No duplicate stage names.
- **Stored in:** `deal_stages` table with columns: `id`, `name`, `display_order`, `is_closed`, `is_won`, `created_at`.
- **Behavior:** If user says "No" to deal tracking, the entire Deals module is hidden from the sidebar, dashboard, and all navigation. The database tables are still created (for future enablement) but routes return 403 with message "Deals module is not enabled." The setting `deals_enabled` in the `settings` table controls this.
- **Edge case:** If user provides stages without explicitly marking closed-won/closed-lost, agent MUST ask: "Which of these stages represents a won deal? Which represents a lost deal?" Both are required.

---

### Q7: Currency

> What currency should be used for deal values and financial reporting?

- **Default:** `"USD"`
- **Options:** Any valid ISO 4217 currency code. Common: USD, EUR, GBP, CAD, AUD, JPY, INR, BRL, CHF, CNY, MXN, NZD, SEK, NOK, DKK, SGD, HKD, KRW, ZAR, AED.
- **Used in:** Deal value display, deal creation forms, dashboard revenue metrics, reports, CSV exports.
- **Validation:** Must be a valid 3-letter ISO 4217 currency code (uppercase).
- **Stored in:** `settings` table as `currency`. Also stored: `currency_symbol` (derived, e.g., `$`), `currency_locale` (e.g., `en-US`), `currency_decimal_places` (e.g., `2` for USD, `0` for JPY).
- **Implementation note:** All monetary values are stored in the database as `DECIMAL(15,2)`. Display formatting uses `Intl.NumberFormat` with the configured locale and currency code. For zero-decimal currencies (e.g., JPY), the application layer handles rounding before display.

---

### Q8: Custom Fields on Contacts

> Do you need any custom fields on contacts beyond the defaults? Default fields: First Name, Last Name, Email, Phone, Mobile, Company, Job Title, Address (Street, City, State/Province, Postal Code, Country), Website, LinkedIn URL, Date of Birth, Source, Tags, Notes, Owner.

- **Default:** No additional custom fields.
- **Format:** Each custom field requires: `name` (string), `type` (one of: `text`, `textarea`, `number`, `date`, `datetime`, `select`, `multi-select`, `checkbox`, `url`, `email`, `phone`, `currency`), `required` (boolean), `options` (array of strings, only for select/multi-select types), `default_value` (matching the field type, optional), `help_text` (string, optional, shown as tooltip).
- **Validation per type:**
  - `text`: max 255 characters
  - `textarea`: max 5000 characters
  - `number`: must be valid numeric, optional min/max
  - `date`: ISO 8601 date string
  - `datetime`: ISO 8601 datetime string
  - `select`: value must be one of defined options
  - `multi-select`: each value must be one of defined options, stored as JSON array
  - `checkbox`: boolean
  - `url`: valid URL format
  - `email`: valid email format
  - `phone`: E.164 format preferred but flexible (stored normalized)
  - `currency`: `DECIMAL(15,2)`, displayed with configured currency
- **Stored in:** Inline `custom_fields` JSONB column on the contacts table (NOT in separate `custom_field_definitions`/`custom_field_values` tables — those tables are NOT used; custom fields are stored as inline JSONB).
- **Maximum:** 50 custom fields per entity type.

---

### Q9: Custom Fields on Companies

> Do you need any custom fields on companies beyond the defaults? Default fields: Name, Industry, Website, Phone, Address (Street, City, State/Province, Postal Code, Country), Size (employee count range), Annual Revenue, Tags, Notes, Owner.

- **Default:** No additional custom fields.
- **Format:** Same as Q8.
- **Stored in:** Inline `custom_fields` JSONB column on the companies table.

---

### Q10: Custom Fields on Deals

> Do you need any custom fields on deals beyond the defaults? Default fields: Name, Value, Currency, Stage, Expected Close Date, Probability (%), Contact, Company, Owner, Source, Tags, Notes.

- **Default:** No additional custom fields.
- **Format:** Same as Q8.
- **Stored in:** Inline `custom_fields` JSONB column on the deals table.

---

### Q11: Email Provider Credentials (Brevo)

> Provide your Brevo (formerly Sendinblue) API key for transactional emails. This is used for password reset emails, invitation emails, activity notifications, and meeting reminders. You can get this from https://app.brevo.com/settings/keys/api.

- **Default:** `null` (emails are disabled; system logs email payloads to console instead of sending. A warning banner appears in admin settings: "Email sending is disabled. Configure Brevo API key in settings.")
- **Validation:** Brevo API keys follow the format `xkeysib-XXXX...` (starts with `xkeysib-`). Agent should validate format before accepting. Agent should test the key with a GET request to `https://api.brevo.com/v3/account` using the key as `api-key` header. If the request fails, warn user but allow proceeding (key might work in production environment with different network rules).
- **Also required if providing Brevo key:**
  - **Sender email address:** The verified "from" address in Brevo. Default: `noreply@<user-domain>`.
  - **Sender name:** Display name for sent emails. Default: Company name from Q1.
- **Stored in:** Environment variable `BREVO_API_KEY`. Sender info in `settings` table as `email_sender_address` and `email_sender_name`.
- **Security:** This value must NEVER appear in logs, error messages, API responses, or frontend code. Stored only in `.env` file and environment variables.

---

### Q12: Cloudflare R2 Credentials

> Provide your Cloudflare R2 credentials for file storage (attachments, profile photos, imports). You can get these from the Cloudflare dashboard under R2 > Manage R2 API Tokens.

- **Required values:**
  - `R2_ACCOUNT_ID` — Cloudflare account ID (32-char hex string)
  - `R2_ACCESS_KEY_ID` — R2 access key ID
  - `R2_SECRET_ACCESS_KEY` — R2 secret access key
  - `R2_BUCKET_NAME` — Name of the R2 bucket to use
  - `R2_PUBLIC_URL` — (Optional) Custom domain or R2.dev public URL for the bucket
- **Default:** `null` for all (file uploads are disabled; upload buttons are hidden in UI. A warning banner appears in admin settings: "File storage is not configured. Configure R2 credentials in settings.")
- **Validation:** Account ID must be 32 hex characters. Bucket name must be 3-63 characters, lowercase alphanumeric and hyphens only. Agent should attempt a `HeadBucket` operation to verify credentials and bucket exist. If verification fails, warn user but allow proceeding.
- **Stored in:** Environment variables only. NEVER in database or frontend code.
- **Implementation note:** The backend uses AWS SDK v3 with S3-compatible endpoint `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`. All file uploads go through the backend (never direct browser-to-R2 uploads) to maintain access control.

---

### Q13: Domain / Subdomain for Deployment

> What domain or subdomain will this CRM be deployed to? This is needed for CORS configuration, cookie domain settings, and email links.

- **Default:** `"localhost:5173"` (development mode)
- **Validation:** Must be a valid domain name or `localhost` with optional port. No protocol prefix (no `http://` or `https://`). Examples: `crm.yourcompany.com`, `app.example.io`, `localhost:5173`.
- **Used in:** CORS `origin` whitelist, `cookie.domain` in session config, links in email templates, CSP headers.
- **Stored in:** Environment variables `APP_DOMAIN` and `APP_URL` (full URL with protocol, derived: `https://<domain>` for production, `http://<domain>` for development).
- **Edge case:** If domain includes a port number, cookie domain must be set to just the hostname portion. If domain is `localhost`, secure cookies must be disabled (SameSite=Lax, Secure=false).

---

### Q14: Initial Admin User

> Provide the details for the first admin user account. This account will have full administrative privileges.

- **Required:**
  - **Email:** Valid email address. Default: `admin@<domain-from-Q13>`.
  - **Full Name:** Non-empty string, max 100 characters. Default: `"Admin User"`.
  - **Password:** Minimum 10 characters, must contain at least 1 uppercase, 1 lowercase, 1 number, and 1 special character. Default: Agent generates a secure random 16-character password and displays it to the user ONCE.
- **Stored in:** `users` table with `role = 'admin'`. Password is bcrypt-hashed with cost factor 12.
- **Behavior:** This user is created during the seed step. This user cannot be deleted (enforced at application level — at least one admin must always exist). The email is also set as the `system_admin_email` in settings for critical system notifications.
- **Edge case:** If the user provides an email that does not match the deployment domain, that is allowed but the agent should note it. The admin user's email is independent of the company domain.

---

### Q15: Custom Activity Types

> Do you need any activity types beyond the defaults? Default types: Call, Email, Meeting, Note, Task. Activity types categorize interactions logged against contacts, companies, and deals.

- **Default:** `["Call", "Email", "Meeting", "Note", "Task"]`
- **Used in:** Activity creation forms, activity timeline filtering, activity reports.
- **Validation:** Each type must be a non-empty string, max 50 characters. Minimum 1 type. Maximum 20 types. No duplicates. Each type also requires an icon identifier (from the Lucide icon set). Defaults: Call=`Phone`, Email=`Mail`, Meeting=`Calendar`, Note=`FileText`, Task=`CheckSquare`.
- **Stored in:** `activity_types` table with columns: `id`, `name`, `label`, `icon`, `color` (hex), `is_system` (boolean, true for system types), `display_order`, `created_at`, `updated_at`.
- **Behavior:** Default types cannot be deleted, only deactivated. Deactivated types are hidden from creation forms but remain visible on existing activities. Custom types can be fully deleted only if no activities reference them; otherwise they can only be deactivated.

---

### Q16: Timezone

> What timezone should be used as the system default? Individual users can override this in their profile settings.

- **Default:** `"America/New_York"` (ET)
- **Validation:** Must be a valid IANA timezone identifier. Agent should validate against the full IANA tz database list. Examples: `America/New_York`, `Europe/London`, `Asia/Tokyo`, `America/Los_Angeles`, `Australia/Sydney`.
- **Used in:** Default display timezone for dates/times in the UI, email template timestamps, activity logging, meeting scheduling, "created at" display values. All dates are STORED in UTC in the database — timezone is only applied at the presentation layer.
- **Stored in:** `settings` table as `default_timezone`. Each user also has a `timezone` column in the `users` table that overrides this default (nullable; if null, system default applies).
- **Implementation note:** The backend always works in UTC. The frontend converts UTC to the user's timezone (or system default) for display using `date-fns-tz`. When the user inputs a date/time, the frontend converts from user timezone to UTC before sending to the API.

---

### Q17: Password Policy

> What is the minimum password length for user accounts?

- **Default:** `10` characters minimum
- **Used in:** User registration, password change, password reset.
- **Validation:** Minimum value is 8, maximum value is 128. Must be a positive integer.
- **Additional requirements (non-configurable):** At least 1 uppercase letter, 1 lowercase letter, 1 number, and 1 special character.
- **Stored in:** `settings` table as `password_min_length`.

---

### Q18: Compliance Requirements

> Do you have specific compliance needs? This affects data handling, retention, and user-facing features.

- **Default:** None (basic best practices still apply).
- **Options:**
  - **GDPR:** Enables consent tracking fields on contacts, right-to-erasure endpoint (`DELETE /api/v1/contacts/:id/gdpr-erase` which anonymizes the contact record), data export endpoint (`GET /api/v1/contacts/:id/export` returning all data for a contact in JSON), cookie consent banner, privacy policy link in footer, consent audit log.
  - **CCPA:** Similar to GDPR plus "Do Not Sell" flag on contacts, opt-out tracking.
  - **HIPAA:** Adds additional encryption requirements warning (agent should warn that full HIPAA compliance requires infrastructure-level controls beyond application scope), audit logging for all data access, automatic session timeout after 15 minutes of inactivity.
  - **SOC 2:** Enables comprehensive audit logging for all create/update/delete operations, login attempt logging, API key usage logging.
  - **None:** Standard security best practices without additional compliance features.
- **Stored in:** `settings` table as JSON array `compliance_frameworks`.
- **Behavior:** Compliance features are additive. Selecting multiple frameworks enables the union of all their features. Compliance settings affect database schema (additional columns/tables), API endpoints, and UI elements. Agent must apply all relevant schema changes during migration generation.

---

### Q19: Expected Number of Users

> How many users do you expect to have? This helps size the database, session store, and rate limiting configuration.

- **Default:** `10`
- **Validation:** Positive integer, minimum 1, maximum 10000.
- **Used in:** Rate limiting configuration (higher user counts get more generous limits), session store pool sizing, database connection pool sizing, suggested pagination defaults.
- **Stored in:** `settings` table as `expected_user_count`. Also influences environment variables:
  - `DB_POOL_MIN`: `Math.max(2, Math.floor(users / 5))`
  - `DB_POOL_MAX`: `Math.max(10, Math.floor(users / 2))`
  - `RATE_LIMIT_WINDOW_MS`: `60000` (1 minute)
  - `RATE_LIMIT_MAX_REQUESTS`: `Math.max(100, users * 20)` per window
- **Edge case:** If user says "just me" or "1", set `expected_user_count = 1` and use minimal pool sizes (min 2, max 5).

---

### Q20: Email Templates

> Do you need pre-built email templates for common CRM actions? These are HTML email templates sent via Brevo for system actions.

- **Default:** Yes, include all default templates.
- **Default templates:**
  - **Welcome Email:** Sent to newly invited users with their login credentials/link.
  - **Password Reset:** Sent when a user requests a password reset. Includes a time-limited token link (expires in 1 hour).
  - **Deal Stage Change Notification:** Sent to deal owner when a deal moves stages.
  - **Meeting Reminder:** Sent 24 hours and 1 hour before a scheduled meeting to all participants.
  - **Activity Assignment:** Sent when an activity (task) is assigned to a user.
  - **Weekly Summary Digest:** Sent every Monday at 9 AM (in system timezone) to all users with their weekly activity summary.
- **Customization:** User can request modifications to template content, colors, layout. All templates use the brand colors from Q3 and company name from Q1.
- **Stored in:** `email_templates` table with columns: `id`, `slug` (unique identifier, e.g., `welcome-email`), `name`, `subject` (supports `{{variable}}` interpolation), `html_body` (full HTML, supports `{{variable}}` interpolation), `text_body` (plain text fallback), `variables` (JSON array of variable names the template expects), `is_active` (boolean), `created_at`, `updated_at`.
- **Behavior:** Templates can be edited by admins in the Settings > Email Templates UI. Changes take effect immediately. A "Reset to Default" button restores the original template. Templates support preview mode (renders with sample data).

---

### Q21: Data Import

> Do you have existing data to import? If yes, what format is it in?

- **Default:** No import needed.
- **Supported formats:** CSV, JSON, XLSX (Excel). Agent should ask for a sample file or at minimum a column mapping.
- **Behavior if yes:**
  - Agent asks for the file path or sample data.
  - Agent creates a one-time import script in `server/scripts/import-data.ts`.
  - The script reads the file, maps columns to CRM fields (agent asks user for column mapping if ambiguous), validates each row, and inserts via the service layer (not raw SQL) to trigger all business logic.
  - Import logs are written to `server/logs/import-YYYY-MM-DD.log` with per-row success/failure details.
  - A dry-run mode is included: `npm run import -- --dry-run` validates without inserting.
  - Duplicate detection: If a contact with the same email already exists, the record is skipped and logged as duplicate. User can opt for "upsert" mode where duplicates are updated instead.
- **Validation per entity:**
  - Contacts: `email` is the dedup key. `first_name` is required.
  - Companies: `name` is the dedup key. `name` is required.
  - Deals: `name` + `company` is the composite dedup key. `name`, `value`, `stage` are required.
- **Stored in:** Import creates records in the normal tables. No separate import table. An `import_batch_id` (UUID) is stored on each imported record in a `meta` JSON column for traceability.
- **Edge case:** If the import file has columns that do not map to any standard field, agent should suggest creating custom fields (per Q8/Q9/Q10) for those columns.

---

### Post-Q&A Agent Behavior

After all questions are answered:

1. **Echo back a complete summary** of all values (substituting defaults where the user skipped) in a clean table format.
2. **Ask for explicit confirmation:** "Please confirm these settings are correct. Type 'confirmed' to proceed or tell me what to change."
3. **Do NOT proceed until the user confirms.**
4. **Store all answers** in a configuration manifest file at `server/config/setup-manifest.json` for reference during build. This file is gitignored.
5. **Generate the `.env` file** from the answers, with all secrets populated and all non-secret config set.

---

---

# SECTION 2: Project Overview & Vision

## 2.1 Executive Summary

This CRM (Customer Relationship Management) system is a lightweight, self-hosted, API-first application designed for small to mid-size teams that need a fast, extensible, and privacy-respecting way to manage their customer relationships.

**API-first: all UI operations go through the REST API. No server-side rendering.**

**Single-tenant deployment. One instance = one company.**

The system is **AI-first**: it is designed to be built, maintained, and extended by AI agents. Every design decision prioritizes clarity, predictability, and machine-readability. The API surface is consistent and documented. The codebase follows rigid conventions. The architecture is modular so individual features can be added, modified, or replaced without cascading changes.

## 2.2 Core Principles

1. **API-First:** Every piece of functionality is exposed through a RESTful JSON API. The frontend is merely one consumer of this API. External integrations, scripts, and AI agents can perform any action via the API that a human can perform via the UI. All UI operations go through the REST API — there is no server-side rendering.

2. **AI-First:** The codebase is designed for AI agents to read, understand, modify, and extend. This means: consistent patterns, no magic, explicit over implicit, comprehensive type definitions, predictable file locations, and thorough inline documentation of non-obvious business logic.

3. **Speed:** The application must feel instant. Target metrics:
   - API response time: < 100ms for simple CRUD, < 500ms for complex queries/aggregations.
   - Frontend initial load: < 2 seconds on 3G connection.
   - Time to interactive: < 3 seconds.
   - Client-side navigation: < 100ms (pre-fetched data via TanStack Query).

4. **Simplicity:** No over-engineering. No premature abstraction. No enterprise patterns for non-enterprise problems. If a feature can be implemented in 50 lines of clear code, it should not be 200 lines of "clean architecture."

5. **Extensibility:** The module-based architecture allows adding new feature modules without modifying existing ones. Custom fields allow per-deployment flexibility without schema changes. The API token system allows third-party integrations.

6. **Privacy:** Data stays on the deployment owner's infrastructure. No telemetry. No third-party analytics. No data leaves the system except through explicit integrations (Brevo for email, R2 for storage) that the owner configures and controls.

## 2.3 Authorization Model

**All authenticated users have full read and write access to all CRM entities. Delete operations require ownership or admin role.**

Specifically:
- **Admin role:** Full access to all features, settings, user management, data management. Can delete any entity.
- **Member role:** Full read and write access to all CRM entities (contacts, companies, deals, activities, meetings, notes, attachments, dashboard). Can only delete entities they own. Cannot access settings, user management, or API key management.

## 2.4 Deletion Policy

**Users are soft-deactivated (is_active flag). All other entities (contacts, companies, deals, activities, meetings, notes, attachments) are hard-deleted with appropriate CASCADE or SET NULL behavior.**

- When a user is deactivated (`is_active = false`), they cannot log in but their historical data (created records, activities, ownership) is preserved.
- When a contact, company, deal, activity, meeting, note, or attachment is deleted, it is permanently removed from the database.
- Foreign key constraints define CASCADE or SET NULL behavior so that deleting a parent entity properly handles dependent records (e.g., deleting a contact CASCADE-deletes its notes and activities, while deal.contact_id is SET NULL).

## 2.5 Accessibility

**WCAG 2.1 AA compliance target.** All interactive elements are keyboard-navigable. All form inputs have associated labels. ARIA roles are used for dynamic content. Color contrast ratios meet 4.5:1 minimum for text.

### Responsive Design
The application must be fully responsive and usable on mobile devices (320px+), tablets (768px+), and desktop (1024px+). All pages must adapt gracefully:
- **Tables** become stacked card layouts on mobile (< 768px)
- **Sidebar navigation** collapses to a hamburger menu on mobile
- **Modals** become full-screen sheets on viewports < 640px
- **Forms** stack to single-column layout on mobile
- **Dashboard** cards and charts stack vertically on mobile
- **Kanban board** becomes a single-column scrollable list on mobile
- **Calendar view** becomes a list view on mobile
- Touch targets minimum 44×44px on all interactive elements

## 2.6 Browser Support

| Browser | Minimum Version |
|---------|----------------|
| Chrome  | 100+           |
| Firefox | 100+           |
| Safari  | 16+            |
| Edge    | 100+           |

## 2.7 Performance Targets

| Operation               | Target     |
|-------------------------|------------|
| Single entity GET       | < 50ms     |
| List page (paginated)   | < 200ms    |
| Dashboard aggregation   | < 1s       |
| CSV export              | < 10s      |
| CSV import              | < 30s      |

These are server-side response times measured at the API layer (excluding network latency).

## 2.8 Core Modules

### 2.8.1 Contacts

The foundational entity. Represents individual people the company interacts with.

- **Relationships:** A contact can belong to zero or one company. A contact can be associated with zero or many deals. A contact can have zero or many activities, notes, attachments, and meetings.
- **Key operations:** Create, read, update, delete (hard-delete), bulk update, bulk delete, import, export, merge duplicates, search, filter, sort, paginate.
- **Views:** List view (table with configurable columns), detail view (profile page with tabs for activities, deals, notes, attachments, meetings).

### 2.8.2 Companies

Represents organizations. Serves as a grouping entity for contacts and deals.

- **Relationships:** A company has zero or many contacts. A company has zero or many deals. A company can have zero or many activities, notes, and attachments.
- **Key operations:** Same as Contacts plus: view all associated contacts, view all associated deals, revenue rollup.
- **Views:** List view, detail view with tabs.

### 2.8.3 Deals (Pipeline)

Represents potential revenue opportunities tracked through pipeline stages.

- **Relationships:** A deal belongs to one deal stage. A deal can be associated with one contact and one company. A deal has zero or many activities, notes, and attachments. A deal has one owner (user).
- **Key operations:** Create, read, update, delete (hard-delete), move between stages (drag-and-drop on Kanban), close (won/lost with close reason), reopen, bulk update.
- **Views:** Kanban board (grouped by stage), list view (table), detail view.
- **Metrics:** Total value per stage, weighted pipeline value (value * probability), average deal cycle time, win rate, conversion rate between stages.

### 2.8.4 Activities

Time-stamped interactions logged against contacts, companies, or deals. Activities are the audit trail of relationship-building.

- **Types:** Call, Email, Meeting, Note, Task (plus custom types from Q15).
- **Relationships:** An activity belongs to one user (creator). An activity can be linked to one contact, one company, and/or one deal (any combination, at least one required). Tasks can be assigned to a different user than the creator.
- **Key operations:** Create, read, update, delete (hard-delete), mark task complete, filter by type, filter by date range, filter by associated entity.
- **Views:** Timeline view (chronological, reverse-chronological), list view with filters, activity feed on entity detail pages.

### 2.8.5 Meetings

Structured calendar events with participants, location, and agenda.

- **Relationships:** A meeting has one organizer (user). A meeting has one or many participants (users and/or contacts). A meeting can be linked to a deal and/or company.
- **Key operations:** Create, read, update, cancel, reschedule, add/remove participants.
- **Views:** List view (upcoming/past), detail view, timeline integration on entity detail pages.
- **Notifications:** Email reminders at 24h and 1h before start time (if Brevo is configured).

### 2.8.6 Notes

Rich-text notes attached to contacts, companies, or deals.

- **Content:** Stored as plain text (Markdown supported, rendered on display). Max 50,000 characters.
- **Relationships:** A note belongs to one user (author). A note is linked to one contact, one company, and/or one deal.
- **Key operations:** Create, read, update, delete (hard-delete), pin/unpin (pinned notes appear first).
- **Views:** Inline on entity detail pages, standalone notes list.

### 2.8.7 Attachments

Files uploaded and stored in Cloudflare R2, associated with any entity.

- **Supported types:** PDF, DOCX, XLSX, CSV, PNG, JPG, JPEG, GIF, ZIP. Max file size: 25MB.
- **Relationships:** An attachment belongs to one user (uploader). An attachment is linked to one contact, one company, one deal, or one note.
- **Key operations:** Upload, download, delete (hard-delete), preview (images and PDFs in browser).
- **Views:** Attachment list on entity detail pages, global attachment list in settings (admin only, for storage management).
- **Storage:** Files are stored in R2 with the key pattern: `attachments/<entity_type>/<entity_id>/<uuid>-<original_filename>`. Metadata (filename, size, MIME type, uploader, entity association) is stored in the `attachments` database table.

### 2.8.8 Dashboard

The landing page after login. Provides at-a-glance metrics and action items.

- **Widgets:**
  - Total contacts / new this week/month
  - Total companies / new this week/month
  - Open deals count and total value
  - Pipeline funnel visualization
  - Upcoming meetings (next 7 days)
  - Overdue tasks
  - Recent activities (last 10)
  - Deal close rate (last 30 days)
- **Behavior:** Dashboard data is loaded via a single aggregation endpoint `GET /api/v1/dashboard` that returns all widget data in one response for performance. Data is cached on the server for 60 seconds (cache key includes user ID for permission-scoped data).

### 2.8.9 Settings (Admin Only)

Administrative configuration panel.

- **Sections:**
  - **General:** Company name, logo, colors, timezone, currency.
  - **Users:** Invite/manage users, assign roles.
  - **Pipeline:** Manage deal stages (add, reorder, rename, deactivate).
  - **Custom Fields:** Manage custom field definitions per entity type.
  - **Activity Types:** Manage activity types.
  - **Email Templates:** View/edit email templates.
  - **API Keys:** Generate/revoke API keys for integrations.
  - **Data Management:** Export all data, import data, delete all data (with confirmation).
  - **Compliance:** Manage compliance settings (if any frameworks are enabled).

### 2.8.10 User Management

- **Roles:**
  - **Admin:** Full access to all features, settings, user management, data management.
  - **Member:** Access to all CRM features (contacts, companies, deals, activities, meetings, notes, attachments, dashboard). Cannot access settings, user management, or API key management.
- **Authentication:** Session-based with HTTP-only, secure, SameSite cookies. Sessions stored in PostgreSQL via `connect-pg-simple`.
- **Invitation flow:** Admin enters email + name + role -> system creates user with random temp password -> welcome email sent with login link and temp password -> user is forced to change password on first login.
- **Password policy:** Minimum 10 characters (configurable via Q17), must contain uppercase, lowercase, number, special character. Passwords are hashed with bcrypt (cost factor 12). Password reset via email token (1-hour expiry).
- **User deactivation:** Users are never hard-deleted. Setting `is_active = false` prevents login while preserving all historical data.

### 2.8.11 API Keys

Token-based authentication for external integrations (scripts, Zapier, AI agents, etc.).

- **Behavior:** Admin generates an API key with a name/description. The key is displayed once and cannot be retrieved again. Keys are stored as HMAC-SHA256 hashes in the database. API key auth is provided via `Authorization: Bearer crm_...` header. API keys have the same permissions as the admin who created them. Keys can be revoked at any time.
- **Rate limiting:** API key requests have a separate, more generous rate limit than session-based requests (configurable).

## 2.9 What This CRM Is NOT

- **Not a marketing automation platform.** No email campaigns, no drip sequences, no A/B testing.
- **Not a helpdesk.** No ticket management, no SLAs, no customer portals.
- **Not a project management tool.** Tasks exist only as activity types, not as a full task management system.
- **Not multi-tenant.** One deployment = one company. For multiple companies, deploy multiple instances.
- **Not a real-time collaboration tool.** No WebSocket-based live updates, no presence indicators, no collaborative editing.

---

---

# SECTION 3: Technology Stack & Architecture

## 3.1 Frontend Stack

### 3.1.1 Build Tool: Vite

- **Version:** 5.x+
- **Why:** Fastest dev server startup, instant HMR, optimized production builds with Rollup under the hood.
- **Configuration:** `client/vite.config.ts` with the following:
  - `react()` plugin
  - Proxy `/api/v1` requests to backend in development: `server.proxy['/api/v1'] = { target: 'http://localhost:3000', changeOrigin: true }`
  - Build output to `client/dist/`
  - Source maps enabled in development, disabled in production
  - Chunk splitting strategy: vendor chunk for `node_modules`, per-feature lazy chunks

### 3.1.2 Framework: React 18+ with TypeScript

- **Version:** React 18.2+, TypeScript 5.x+
- **Strict mode:** Enabled (`<React.StrictMode>` wrapper in `main.tsx`)
- **TypeScript config:** `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true` in `tsconfig.json`
- **Component style:** Functional components only. No class components. Arrow function syntax for components: `const MyComponent: React.FC<Props> = ({ prop1, prop2 }) => { ... }`. Avoid `React.FC` if not needed — prefer explicit return types.

### 3.1.3 Routing: React Router v6

- **Version:** 6.x
- **Pattern:** File-based-like organization within `features/` but configured in a central `router.tsx` file.
- **Route structure:**
  ```
  /login                      — Public, LoginPage
  /forgot-password            — Public, ForgotPasswordPage
  /reset-password/:token      — Public, ResetPasswordPage
  /                           — Protected, redirect to /dashboard
  /dashboard                  — Protected, DashboardPage
  /contacts                   — Protected, ContactsListPage
  /contacts/new               — Protected, ContactCreatePage
  /contacts/:id               — Protected, ContactDetailPage
  /contacts/:id/edit          — Protected, ContactEditPage
  /companies                  — Protected, CompaniesListPage
  /companies/new              — Protected, CompanyCreatePage
  /companies/:id              — Protected, CompanyDetailPage
  /companies/:id/edit         — Protected, CompanyEditPage
  /deals                      — Protected (if enabled), DealsListPage
  /deals/board                — Protected (if enabled), DealsBoardPage (Kanban)
  /deals/new                  — Protected (if enabled), DealCreatePage
  /deals/:id                  — Protected (if enabled), DealDetailPage
  /deals/:id/edit             — Protected (if enabled), DealEditPage
  /activities                 — Protected, ActivitiesListPage
  /meetings                   — Protected, MeetingsListPage
  /meetings/new               — Protected, MeetingCreatePage
  /meetings/:id               — Protected, MeetingDetailPage
  /settings                   — Protected (Admin only), SettingsPage (with nested routes)
  /settings/general           — Admin, GeneralSettingsPage
  /settings/users             — Admin, UsersManagementPage
  /settings/pipeline          — Admin, PipelineSettingsPage
  /settings/custom-fields     — Admin, CustomFieldsSettingsPage
  /settings/activity-types    — Admin, ActivityTypesSettingsPage
  /settings/email-templates   — Admin, EmailTemplatesPage
  /settings/api-keys          — Admin, ApiKeysPage
  /settings/data              — Admin, DataManagementPage
  ```
- **Protected routes:** Wrapped in an `AuthGuard` component that checks session state (via Zustand store). If not authenticated, redirect to `/login` with `returnUrl` query param. If authenticated but insufficient role (e.g., member accessing `/settings`), redirect to `/dashboard` with a toast error.
- **Lazy loading:** All page-level components are lazy-loaded via `React.lazy()` + `Suspense` with a loading spinner fallback. Shared components are NOT lazy-loaded.

### 3.1.4 Server State: TanStack Query (React Query)

- **Version:** 5.x
- **Configuration:** `QueryClientProvider` wrapping the app in `main.tsx` with default options:
  ```typescript
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 30_000,        // 30 seconds
        gcTime: 5 * 60_000,       // 5 minutes (formerly cacheTime)
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  });
  ```
- **Query key conventions:** Feature-scoped arrays. Examples:
  - `['contacts']` — all contacts list
  - `['contacts', { page, limit, sort, filters }]` — filtered/paginated contacts list
  - `['contacts', id]` — single contact by ID
  - `['contacts', id, 'activities']` — activities for a contact
  - `['contacts', id, 'deals']` — deals for a contact
  - `['dashboard']` — dashboard data
- **Mutations:** All create/update/delete operations use `useMutation` with `onSuccess` callbacks that `queryClient.invalidateQueries` the relevant query keys. Optimistic updates are used for simple state changes (e.g., toggling a task complete).
- **Custom hooks:** Each feature module exports custom hooks:
  - `useContacts(params)` — returns paginated contacts query
  - `useContact(id)` — returns single contact query
  - `useCreateContact()` — returns create mutation
  - `useUpdateContact()` — returns update mutation
  - `useDeleteContact()` — returns delete mutation
  - (Same pattern for every entity)

### 3.1.5 Client State: Zustand

- **Version:** 4.x+
- **Stores:**
  - `useAuthStore` — `{ user: User | null, isAuthenticated: boolean, login, logout, checkSession }`. Persisted to `sessionStorage` (not `localStorage` to avoid stale sessions across tabs).
  - `useUIStore` — `{ sidebarCollapsed: boolean, theme: 'light' | 'dark', toggleSidebar, setTheme }`. Persisted to `localStorage`.
  - `useFilterStore` — Per-entity filter state for list pages. Not persisted (resets on page navigation). Structure: `{ contacts: { search, sortBy, sortOrder, filters, page, limit }, companies: {...}, deals: {...} }`.
- **Rules:** No business logic in stores. Stores hold state only. Business logic lives in custom hooks or utility functions.

### 3.1.6 Styling: Tailwind CSS + shadcn/ui

- **Tailwind version:** 3.4+
- **shadcn/ui:** Not installed as a package — components are copied into `client/src/components/ui/` (this is the shadcn/ui convention). Use the shadcn CLI to add components: `npx shadcn-ui@latest add button`. Components used:
  - `Button`, `Input`, `Label`, `Select`, `Textarea`, `Checkbox`, `RadioGroup`, `Switch`
  - `Dialog`, `Sheet`, `Popover`, `Tooltip`, `DropdownMenu`, `Command` (for combobox/search)
  - `Table`, `Card`, `Badge`, `Avatar`, `Separator`
  - `Toast` (via `Sonner` integration), `Alert`, `AlertDialog`
  - `Tabs`, `Accordion`, `Collapsible`
  - `Form` (integrated with React Hook Form)
  - `Calendar`, `DatePicker`
  - `Skeleton` (for loading states)
- **Custom Tailwind config extensions:**
  ```
  colors: {
    primary: 'hsl(var(--primary))',
    secondary: 'hsl(var(--secondary))',
    // (shadcn/ui default color system using CSS custom properties)
  }
  ```
- **Responsive design:** Mobile-first. Sidebar collapses to hamburger menu below `md` breakpoint (768px). Tables switch to card view below `sm` breakpoint (640px). All form layouts are single-column on mobile, multi-column on `lg`+.

Tailwind is configured with default breakpoints (`sm: 640px`, `md: 768px`, `lg: 1024px`, `xl: 1280px`). All components are built mobile-first — base styles target mobile, responsive prefixes add tablet/desktop enhancements.

### 3.1.7 Forms: React Hook Form + Zod

- **React Hook Form version:** 7.x
- **Zod version:** 3.x
- **Pattern:** Every form defines a Zod schema. The schema is used as the React Hook Form resolver via `@hookform/resolvers/zod`. The same Zod schemas are shared with the backend validation logic via the `shared/` package.
- **Error display:** Field-level errors appear below each input in red text. Form-level errors (e.g., server-side validation failures) appear in a toast and/or an alert banner at the top of the form.
- **Submission handling:** Forms disable the submit button and show a loading spinner during submission. On success: toast notification + redirect (for create) or toast notification (for edit). On failure: re-enable form, show errors.

### 3.1.8 Date Handling: date-fns

- **Version:** 3.x+
- **Usage:** All date formatting, parsing, and manipulation. Never use native `Date` methods for formatting (inconsistent across browsers).
- **Common formatters:** `format(date, 'MMM d, yyyy')` for display, `formatDistanceToNow(date)` for relative times ("5 minutes ago"), `parseISO()` for parsing API responses.
- **Timezone:** Use `date-fns-tz` for timezone conversions. All dates from the API are in UTC. The frontend converts to the user's timezone for display.

### 3.1.9 Icons: Lucide React

- **Version:** Latest
- **Usage:** Import individual icons: `import { Phone, Mail, Calendar } from 'lucide-react'`. Default size 20px, stroke-width 1.5. Consistent sizing via a wrapper component or utility class.

### 3.1.10 HTTP Client: Axios

- **Version:** 1.x
- **Configuration:** Centralized Axios instance in `client/src/lib/api-client.ts`:
  ```typescript
  const apiClient = axios.create({
    baseURL: '/api/v1',
    withCredentials: true,    // send cookies for session auth
    headers: { 'Content-Type': 'application/json' },
    timeout: 30000,           // 30 second timeout
  });
  ```
- **Interceptors:**
  - **Request interceptor:** Adds `X-Request-ID` header (UUID) for request tracing.
  - **Response interceptor:** On 401 response, clears auth store and redirects to `/login`. On 403, shows toast "You don't have permission to perform this action." On 5xx, shows toast "An unexpected error occurred. Please try again."
- **API layer:** Each feature module has an API file (e.g., `features/contacts/contacts.api.ts`) that exports typed functions:
  ```typescript
  export const contactsApi = {
    getAll: (params: ContactsQueryParams) => apiClient.get<ApiResponse<Contact[]>>('/contacts', { params }),
    getById: (id: string) => apiClient.get<ApiResponse<Contact>>(`/contacts/${id}`),
    create: (data: CreateContactDto) => apiClient.post<ApiResponse<Contact>>('/contacts', data),
    update: (id: string, data: UpdateContactDto) => apiClient.patch<ApiResponse<Contact>>(`/contacts/${id}`, data),
    delete: (id: string) => apiClient.delete<ApiResponse<void>>(`/contacts/${id}`),
  };
  ```

### 3.1.11 Client Dependencies (Complete List)

**Production dependencies (`client/package.json`):**
react, react-dom, react-router-dom, @tanstack/react-query, zustand, axios, zod, react-hook-form, @hookform/resolvers, date-fns, lucide-react, tailwindcss, @dnd-kit/core, @dnd-kit/sortable, marked, sanitize-html

## 3.2 Backend Stack

### 3.2.1 Runtime: Node.js 20+

- **Version:** 20 LTS or later (for native fetch, stable test runner, performance improvements).
- **TypeScript:** Compiled via `tsc`. `tsconfig.json` with `strict: true`, `target: ES2022`, `module: NodeNext`, `moduleResolution: NodeNext`.
- **Dev runner:** `tsx watch` (or `ts-node-dev`) for development with auto-reload.
- **Production:** Compiled to JavaScript in `server/dist/`, run with `node dist/index.js`.

### 3.2.2 Framework: Express.js

- **Version:** 4.x (not v5 — v5 is not yet stable enough for production)
- **Configuration:** `server/src/app.ts` sets up middleware in this exact order:
  1. `helmet()` — Security headers
  2. `cors(corsOptions)` — CORS with configured origins
  3. `express.json({ limit: '10mb' })` — JSON body parsing with size limit
  4. `express.urlencoded({ extended: true, limit: '10mb' })` — URL-encoded body parsing
  5. `cookieParser()` — Cookie parsing
  6. `sessionMiddleware` — Session management
  7. `requestIdMiddleware` — Generates and attaches UUID to each request
  8. `requestLoggerMiddleware` — Logs each request (method, URL, status, duration)
  9. `rateLimitMiddleware` — Global rate limiting
  10. Route mounting (`/api/v1/auth`, `/api/v1/contacts`, `/api/v1/companies`, etc.)
  11. `notFoundHandler` — 404 handler for unmatched routes
  12. `errorHandler` — Centralized error handler (MUST be last)

**ALL API routes use the `/api/v1/` prefix. No exceptions. No `/api/` without version.**

### 3.2.3 Database: PostgreSQL 15+

- **Why PostgreSQL:** JSONB support for custom fields, excellent performance, rock-solid reliability, rich ecosystem.
- **Connection:** Via Knex.js connection pool. Connection string from `DATABASE_URL` environment variable.
- **Naming:** All table names are `snake_case` plural (e.g., `contacts`, `companies`, `deals`). All column names are `snake_case`. Primary keys are `id` (UUID v4, generated database-side via `gen_random_uuid()` as the column default). Timestamps are `created_at` and `updated_at` (both `timestamptz`, both set automatically).
- **No soft deletes on entities:** Users are soft-deactivated via `is_active` flag. All other entities are hard-deleted. See Section 2.4 for the full deletion policy.
- **Search:** Use `ILIKE` for text search on contacts, companies, and other entities. Keep it simple — no `tsvector` or full-text search infrastructure.

### 3.2.4 Query Builder & Migration Tool: Knex.js

- **Version:** 3.x
- **Knex.js is the ONLY migration and query tool.** No other migration libraries (no node-pg-migrate, no db-migrate, etc.).
- **Configuration:** `server/knexfile.ts` with environments for development, production, and test.
- **Connection pool:** Min/max sized based on expected users (from Q19).
- **Migrations:** Located in `server/migrations/`. Named with timestamp prefix: `20240101120000_create_contacts_table.ts`. Each migration exports `up` and `down` functions. Migrations run automatically on application startup (in production, this can be disabled via `RUN_MIGRATIONS=false` env var; in that case, run `npx knex migrate:latest` manually).
- **Seeds:** Located in `server/seeds/`. Used for initial setup (admin user, default settings, default deal stages, default activity types, default email templates). Seeds are idempotent — running them twice does not create duplicates (use `INSERT ... ON CONFLICT DO NOTHING` or check-before-insert logic).
- **Query patterns:**
  ```typescript
  // ALWAYS use parameterized queries
  knex('contacts').where({ id }).first();                    // simple lookup
  knex('contacts').where('email', 'ilike', `%${search}%`);  // search (Knex parameterizes automatically)
  knex('contacts').insert(data).returning('*');              // insert with return
  knex('contacts').where({ id }).update(data).returning('*'); // update with return

  // NEVER do this:
  knex.raw(`SELECT * FROM contacts WHERE email = '${email}'`); // SQL INJECTION!

  // If raw SQL is needed, ALWAYS parameterize:
  knex.raw('SELECT * FROM contacts WHERE email = ?', [email]); // Safe
  ```

### 3.2.5 Session Management: express-session + connect-pg-simple

- **Session store:** PostgreSQL (via `connect-pg-simple`). Session table is created by the library automatically.
- **Session config:**
  ```typescript
  {
    store: new PgSession({ pool, tableName: 'sessions' }),
    secret: process.env.SESSION_SECRET,   // min 64-char random string
    resave: false,
    saveUninitialized: false,
    rolling: true,                         // resets expiry on every request
    cookie: {
      maxAge: 24 * 60 * 60 * 1000,       // 24 hours (rolling)
      httpOnly: true,                       // not accessible via JavaScript
      secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
      sameSite: 'lax',                     // CSRF protection
      domain: process.env.COOKIE_DOMAIN,   // from Q13
    },
    name: 'crm.sid',                       // custom session cookie name (not default 'connect.sid')
  }
  ```
- **Session data:** Stores `userId`, `role`, `loginAt`. Nothing else. Full user data is fetched from the database per request (via auth middleware).

### 3.2.6 Password Hashing: bcrypt

- **Version:** 5.x
- **Cost factor:** 12 (balances security and performance; takes ~250ms to hash on modern hardware).
- **Usage:** `bcrypt.hash(password, 12)` to hash, `bcrypt.compare(password, hash)` to verify. NEVER store plaintext passwords. NEVER log passwords (even hashed).

### 3.2.7 Input Validation: Zod (shared with frontend)

- **Package:** `zod` (version 3.x) — used on BOTH frontend and backend.
- **Middleware:** A `zod-express` middleware validates request body, params, and query against Zod schemas.
- **Pattern:** Each feature has a `*.schemas.ts` file in the `shared/` package that exports Zod schemas. Both frontend forms and backend routes use the same schemas:
  ```typescript
  // shared/schemas/contact.schemas.ts
  import { z } from 'zod';

  export const createContactSchema = z.object({
    firstName: z.string().trim().min(1, 'First name is required').max(100),
    lastName: z.string().trim().max(100).optional(),
    email: z.string().trim().email('Invalid email format').optional(),
    phone: z.string().trim().max(20).optional(),
    // ... etc.
  });

  export type CreateContactDto = z.infer<typeof createContactSchema>;
  ```
- **Backend middleware:**
  ```typescript
  // server/src/middleware/validate.middleware.ts
  import { ZodSchema } from 'zod';

  export function validateBody(schema: ZodSchema) {
    return (req: Request, res: Response, next: NextFunction) => {
      const result = schema.safeParse(req.body);
      if (!result.success) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Validation failed',
            details: result.error.issues.map(issue => ({
              field: issue.path.join('.'),
              message: issue.message,
            })),
          },
        });
      }
      req.body = result.data; // use parsed/transformed data
      next();
    };
  }
  ```
- **This replaces express-validator entirely.** Do not use express-validator anywhere in the project.

### 3.2.8 File Uploads: multer + AWS SDK v3 (R2)

- **multer:** Used as middleware to parse `multipart/form-data` requests. Files are buffered in memory (not disk) with a 25MB limit.
  ```typescript
  const upload = multer({
    storage: multer.memoryStorage(),
    limits: { fileSize: 25 * 1024 * 1024 },  // 25MB
  });
  ```
- **MIME type validation:** Do NOT rely on the `Content-Type` header from the client. After multer parses the file into a buffer, use the `file-type` package to detect the actual MIME type from magic bytes:
  ```typescript
  import { fileTypeFromBuffer } from 'file-type';

  const ALLOWED_MIME_TYPES = [
    'image/png', 'image/jpeg', 'image/gif',
    'application/pdf',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    'text/csv', 'application/zip',
  ];
  // NOTE: SVG is intentionally excluded due to XSS risk (SVGs can contain script tags).

  const detected = await fileTypeFromBuffer(buffer);
  if (!detected || !ALLOWED_MIME_TYPES.includes(detected.mime)) {
    throw new ValidationError('File type not allowed');
  }
  ```
- **R2 upload flow:**
  1. multer parses file into memory buffer
  2. Validate MIME type via magic bytes using `file-type`
  3. Controller extracts file metadata (name, size, detected MIME type)
  4. Service generates R2 key: `attachments/<entity_type>/<entity_id>/<uuid>-<sanitized_filename>`
  5. Service uploads buffer to R2 via `PutObjectCommand`
  6. Service stores metadata in `attachments` table (key, filename, size, mimetype, uploader, entity)
  7. Controller returns attachment record with download URL
- **Download flow:** `GET /api/v1/attachments/:id/download` -> Service fetches metadata from DB -> Service generates presigned URL from R2 (expires in 5 minutes) -> Returns 302 redirect to presigned URL. Alternative: stream through backend via `GetObjectCommand` and pipe to response (used when presigned URLs are not configured).
- **Presigned URL expiry: 5 minutes.** Not 15, not 60. This limits the window for URL sharing/leaking.

### 3.2.9 Email Service: Brevo SDK

- **Package:** `@getbrevo/brevo` (official SDK)
- **Configuration:** Centralized in `server/src/services/email.service.ts`.
- **Sending pattern:**
  ```typescript
  class EmailService {
    async send(to: string, templateSlug: string, variables: Record<string, string>): Promise<void> {
      // 1. Fetch template from database by slug
      // 2. Interpolate variables into subject, html_body, text_body
      // 3. If Brevo is configured, send via API
      // 4. If Brevo is NOT configured, log the email payload (to, subject, body) at info level
      // 5. On Brevo API failure, log error but do NOT throw (email failures should not break operations)
    }
  }
  ```
- **Variable interpolation:** Templates use `{{variableName}}` syntax. The service replaces all `{{key}}` occurrences with the corresponding value from the variables object. Unmatched variables are replaced with empty strings and logged as warnings.
- **Retry:** No automatic retry for email sending. Failures are logged. A future enhancement could add a retry queue.

### 3.2.10 Security Middleware

- **helmet:** Default configuration with CSP customized to allow inline styles (required by some shadcn/ui components) and the configured domain.
- **cors:**
  ```typescript
  {
    origin: process.env.CORS_ORIGIN || 'http://localhost:5173',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  }
  ```
- **express-rate-limit:**
  - Global: 100 requests per minute per IP (configurable via env vars).
  - Auth endpoints (`/api/v1/auth/login`, `/api/v1/auth/forgot-password`): 10 requests per 15 minutes per IP.
  - API key endpoints: 300 requests per minute per key.
- **Input sanitization:** All string inputs are trimmed. HTML tags are stripped from text fields (except notes, which allow Markdown). File names sanitized (alphanumeric, hyphens, dots only).

### 3.2.11 Logging: winston

- **Configuration:**
  ```typescript
  const logger = winston.createLogger({
    level: process.env.LOG_LEVEL || 'info',
    format: winston.format.combine(
      winston.format.timestamp(),
      winston.format.errors({ stack: true }),
      process.env.NODE_ENV === 'production'
        ? winston.format.json()
        : winston.format.combine(winston.format.colorize(), winston.format.simple())
    ),
    transports: [
      new winston.transports.Console(),
      // In production, also write to file:
      ...(process.env.NODE_ENV === 'production' ? [
        new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
        new winston.transports.File({ filename: 'logs/combined.log' }),
      ] : []),
    ],
  });
  ```
- **Request logging:** Every HTTP request logs: `{ method, url, statusCode, duration, requestId, userId (if authenticated) }`.
- **Sensitive data:** NEVER log: passwords, API keys, session tokens, email content bodies, file contents. Log redaction middleware strips these fields from log output.

### 3.2.12 Configuration: dotenv + Centralized Config

- **Pattern:** `server/src/config/index.ts` reads all environment variables, validates them, and exports a typed configuration object:
  ```typescript
  export const config = {
    port: requireEnv('PORT', '3000'),
    nodeEnv: requireEnv('NODE_ENV', 'development'),
    database: {
      url: requireEnv('DATABASE_URL'),     // required, no default, fails on missing
      poolMin: parseInt(requireEnv('DB_POOL_MIN', '2')),
      poolMax: parseInt(requireEnv('DB_POOL_MAX', '10')),
    },
    session: {
      secret: requireEnv('SESSION_SECRET'),  // required, no default
    },
    brevo: {
      apiKey: optionalEnv('BREVO_API_KEY'),
      senderEmail: optionalEnv('BREVO_SENDER_EMAIL'),
      senderName: optionalEnv('BREVO_SENDER_NAME'),
    },
    r2: {
      accountId: optionalEnv('R2_ACCOUNT_ID'),
      accessKeyId: optionalEnv('R2_ACCESS_KEY_ID'),
      secretAccessKey: optionalEnv('R2_SECRET_ACCESS_KEY'),
      bucketName: optionalEnv('R2_BUCKET_NAME'),
      publicUrl: optionalEnv('R2_PUBLIC_URL'),
    },
    app: {
      domain: requireEnv('APP_DOMAIN', 'localhost:5173'),
      url: requireEnv('APP_URL', 'http://localhost:5173'),
    },
    rateLimit: {
      windowMs: parseInt(requireEnv('RATE_LIMIT_WINDOW_MS', '60000')),
      maxRequests: parseInt(requireEnv('RATE_LIMIT_MAX_REQUESTS', '100')),
    },
  } as const;
  ```
- **`requireEnv(key, defaultValue?)`:** Returns the env var value. If no value and no default, throws immediately at startup with message `"Missing required environment variable: ${key}"`.
- **`optionalEnv(key)`:** Returns the env var value or `undefined`. No error on missing.
- **Startup validation:** The config module is imported first in `server/src/index.ts`. If any required env var is missing, the process exits with code 1 and a clear error message BEFORE attempting database connections, server startup, or any other initialization.

### 3.2.13 camelCase / snake_case Transformation

**API request/response bodies use camelCase. Database columns use snake_case. The service layer transforms between them.**

- Knex `postProcessResponse` hook converts `snake_case` database results to `camelCase` for the API layer.
- Knex `wrapIdentifier` hook converts `camelCase` identifiers to `snake_case` for queries.
- This mapping is configured ONCE in the Knex setup and applies globally. No manual mapping in service code.

### 3.2.14 Server Dependencies (Complete List)

**Production dependencies (`server/package.json`):**
express, cors, helmet, express-rate-limit, express-session, connect-pg-simple, knex, pg, bcrypt, zod, multer, @aws-sdk/client-s3, @aws-sdk/s3-request-presigner, @getbrevo/brevo, winston, dotenv, uuid, cookie-parser, file-type, csv-parse, csv-stringify, sanitize-html, marked

### 3.2.15 Testing Strategy

- **Framework:** Vitest for both frontend and backend. No other test runner.
- **Test files:** Colocated with source files as `*.test.ts` (e.g., `contacts.service.test.ts` lives next to `contacts.service.ts`).
- **Integration tests:** Use a separate test database (`crm_test`). Created/destroyed per test suite.
- **Coverage target:** Minimum 80% coverage on service layer files.
- **Backend test pattern:** Use `supertest` to test API endpoints end-to-end through Express.
- **Frontend test pattern:** Use `@testing-library/react` and `@testing-library/jest-dom` for component tests.
- **Test commands:**
  ```bash
  npm run test              # Run all tests (vitest)
  npm run test:server       # Backend tests only
  npm run test:client       # Frontend tests only
  npm run test:coverage     # Tests with coverage report
  ```

### 3.2.16 Dev Dependencies (Shared)

**Dev dependencies (root or shared across workspaces):**
typescript, vitest, @testing-library/react, @testing-library/jest-dom, supertest

## 3.3 Project Structure

```
crm/
├── client/                          # Frontend (Vite + React)
│   ├── public/                      # Static assets (favicon, etc.)
│   ├── src/
│   │   ├── components/
│   │   │   ├── ui/                  # shadcn/ui primitives (Button, Input, Dialog, etc.)
│   │   │   ├── forms/               # Reusable form components
│   │   │   │   ├── FormField.tsx    # Generic form field wrapper with label + error
│   │   │   │   ├── SearchableSelect.tsx  # Combobox/searchable dropdown
│   │   │   │   ├── DatePicker.tsx   # Date picker wrapper
│   │   │   │   ├── FileUpload.tsx   # Drag-and-drop file upload
│   │   │   │   ├── TagInput.tsx     # Multi-value tag input
│   │   │   │   ├── RichTextEditor.tsx  # Markdown editor with preview
│   │   │   │   └── EntitySelector.tsx  # Reusable searchable autocomplete for entity references

**Key shared form component — `EntitySelector.tsx`:**
- Reusable searchable autocomplete for entity references. Props: `entityType: 'contact' | 'company' | 'deal' | 'user'`, `value: string | null`, `onChange: (id: string | null, entity?: T) => void`, `filterParams?: Record<string, string>`, `placeholder?: string`, `multiple?: boolean`. Uses TanStack Query with debounced search. Renders selected entity as a chip with name, dismissible.
│   │   │   ├── layout/
│   │   │   │   ├── AppLayout.tsx    # Main layout: sidebar + header + content area
│   │   │   │   ├── Sidebar.tsx      # Navigation sidebar
│   │   │   │   ├── Header.tsx       # Top header with user menu, search
│   │   │   │   ├── PageHeader.tsx   # Page-level header (title, breadcrumbs, actions)
│   │   │   │   └── AuthLayout.tsx   # Layout for login/auth pages (centered card)
│   │   │   ├── data-table/
│   │   │   │   ├── DataTable.tsx    # Generic sortable, filterable, paginated table
│   │   │   │   ├── DataTablePagination.tsx
│   │   │   │   ├── DataTableToolbar.tsx  # Search + filters + bulk actions
│   │   │   │   ├── DataTableColumnHeader.tsx  # Sortable column header
│   │   │   │   └── DataTableFacetedFilter.tsx  # Multi-select column filter
│   │   │   └── common/
│   │   │       ├── LoadingSpinner.tsx
│   │   │       ├── EmptyState.tsx   # "No data" placeholder with illustration
│   │   │       ├── ConfirmDialog.tsx  # Generic confirmation dialog
│   │   │       ├── EntityAvatar.tsx  # Avatar for contacts/companies
│   │   │       ├── StatusBadge.tsx   # Colored badge for statuses
│   │   │       ├── Timeline.tsx      # Activity timeline component
│   │   │       ├── KanbanBoard.tsx   # Drag-and-drop Kanban board
│   │   │       └── ErrorBoundary.tsx # React error boundary
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   │   ├── LoginPage.tsx
│   │   │   │   ├── ForgotPasswordPage.tsx
│   │   │   │   ├── ResetPasswordPage.tsx
│   │   │   │   ├── ChangePasswordPage.tsx
│   │   │   │   ├── auth.api.ts      # API calls for auth
│   │   │   │   ├── auth.hooks.ts    # useLogin, useLogout, useCurrentUser
│   │   │   │   └── auth.types.ts
│   │   │   ├── contacts/
│   │   │   │   ├── ContactsListPage.tsx
│   │   │   │   ├── ContactCreatePage.tsx
│   │   │   │   ├── ContactDetailPage.tsx
│   │   │   │   ├── ContactEditPage.tsx
│   │   │   │   ├── components/       # Contact-specific components
│   │   │   │   │   ├── ContactForm.tsx
│   │   │   │   │   ├── ContactCard.tsx
│   │   │   │   │   └── ContactActivityTab.tsx
│   │   │   │   ├── contacts.api.ts
│   │   │   │   ├── contacts.hooks.ts
│   │   │   │   └── contacts.types.ts
│   │   │   ├── companies/            # Same pattern as contacts
│   │   │   ├── deals/                # Same pattern + KanbanBoard integration
│   │   │   ├── activities/           # Same pattern
│   │   │   ├── meetings/             # Same pattern
│   │   │   ├── settings/             # Admin settings pages
│   │   │   ├── users/                # User management (admin)
│   │   │   └── dashboard/
│   │   │       ├── DashboardPage.tsx
│   │   │       ├── components/
│   │   │       │   ├── MetricCard.tsx
│   │   │       │   ├── PipelineFunnel.tsx
│   │   │       │   ├── UpcomingMeetings.tsx
│   │   │       │   ├── OverdueTasks.tsx
│   │   │       │   └── RecentActivity.tsx
│   │   │       ├── dashboard.api.ts
│   │   │       └── dashboard.hooks.ts
│   │   ├── hooks/
│   │   │   ├── useDebounce.ts       # Debounce hook for search inputs
│   │   │   ├── usePagination.ts     # Pagination state management
│   │   │   ├── useConfirmDialog.ts  # Imperative confirm dialog hook
│   │   │   └── useMediaQuery.ts     # Responsive breakpoint hook
│   │   ├── lib/
│   │   │   ├── api-client.ts        # Axios instance + interceptors
│   │   │   ├── utils.ts             # cn() (clsx + tailwind-merge), formatCurrency, formatDate, etc.
│   │   │   ├── constants.ts         # App-wide constants
│   │   │   └── validations.ts       # Shared Zod schemas (reused across features)
│   │   ├── stores/
│   │   │   ├── auth.store.ts
│   │   │   ├── ui.store.ts
│   │   │   └── filter.store.ts
│   │   ├── types/
│   │   │   ├── api.ts               # ApiResponse<T>, PaginationMeta, etc.
│   │   │   └── index.ts             # Re-exports shared types
│   │   ├── config/
│   │   │   └── index.ts             # Frontend config (API URL, feature flags)
│   │   ├── router.tsx               # React Router configuration
│   │   ├── App.tsx                   # Root component (providers, router outlet)
│   │   ├── main.tsx                  # Entry point (ReactDOM.createRoot)
│   │   └── index.css                # Tailwind directives + custom CSS vars
│   ├── index.html
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   ├── postcss.config.js
│   ├── tsconfig.json
│   ├── tsconfig.node.json
│   └── package.json
├── server/                          # Backend (Express + TypeScript)
│   ├── src/
│   │   ├── config/
│   │   │   ├── index.ts             # Centralized config with env validation
│   │   │   ├── database.ts          # Knex instance creation + pool config
│   │   │   └── r2.ts                # R2 (S3) client initialization
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts    # Session auth check + role check
│   │   │   ├── apiKey.middleware.ts  # API key auth check
│   │   │   ├── validate.middleware.ts  # Zod schema validation middleware
│   │   │   ├── rateLimiter.middleware.ts  # Rate limit configs
│   │   │   ├── requestId.middleware.ts  # UUID generation per request
│   │   │   ├── requestLogger.middleware.ts  # HTTP request logging
│   │   │   ├── errorHandler.middleware.ts  # Centralized error handling
│   │   │   └── upload.middleware.ts  # multer configuration
│   │   ├── features/
│   │   │   ├── auth/
│   │   │   │   ├── auth.routes.ts
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   └── auth.types.ts
│   │   │   ├── contacts/
│   │   │   │   ├── contacts.routes.ts
│   │   │   │   ├── contacts.controller.ts
│   │   │   │   ├── contacts.service.ts
│   │   │   │   ├── contacts.service.test.ts
│   │   │   │   └── contacts.types.ts
│   │   │   ├── companies/            # Same pattern
│   │   │   ├── deals/                # Same pattern
│   │   │   ├── activities/           # Same pattern
│   │   │   ├── meetings/             # Same pattern
│   │   │   ├── users/                # Same pattern
│   │   │   ├── settings/             # Same pattern
│   │   │   ├── attachments/          # Same pattern
│   │   │   ├── notes/                # Same pattern
│   │   │   ├── api-keys/             # Same pattern
│   │   │   └── dashboard/
│   │   │       ├── dashboard.routes.ts
│   │   │       ├── dashboard.controller.ts
│   │   │       └── dashboard.service.ts
│   │   ├── services/
│   │   │   ├── email.service.ts      # Brevo email sending
│   │   │   ├── storage.service.ts    # R2 file storage
│   │   │   └── cache.service.ts      # Simple in-memory cache (Map-based, TTL)
│   │   ├── utils/
│   │   │   ├── errors.ts            # Custom error classes (AppError, NotFoundError, etc.)
│   │   │   ├── pagination.ts        # Pagination helper (parse query params, build meta)
│   │   │   ├── response.ts          # Response builder (success, error, paginated)
│   │   │   ├── filters.ts           # Generic filter/sort query builder
│   │   │   └── crypto.ts            # Token generation, random string helpers
│   │   ├── types/
│   │   │   ├── express.d.ts         # Express type augmentation (req.user, req.requestId)
│   │   │   └── index.ts
│   │   ├── app.ts                   # Express app setup (middleware, routes)
│   │   └── index.ts                 # Entry point (start server, run migrations)
│   ├── migrations/                  # Knex migration files (timestamped)
│   ├── seeds/                       # Seed files (initial data)
│   ├── scripts/
│   │   └── import-data.ts           # Data import script (from Q21)
│   ├── logs/                        # Log files (gitignored)
│   ├── knexfile.ts                  # Knex configuration
│   ├── tsconfig.json
│   └── package.json
├── shared/                          # Shared types and schemas between client and server
│   ├── schemas/
│   │   ├── contact.schemas.ts       # Zod schemas for contacts
│   │   ├── company.schemas.ts       # Zod schemas for companies
│   │   ├── deal.schemas.ts          # Zod schemas for deals
│   │   ├── activity.schemas.ts      # Zod schemas for activities
│   │   ├── meeting.schemas.ts       # Zod schemas for meetings
│   │   ├── note.schemas.ts          # Zod schemas for notes
│   │   ├── auth.schemas.ts          # Zod schemas for auth (login, register, etc.)
│   │   └── common.schemas.ts        # Shared validation schemas (email, URL, UUID, pagination)
│   ├── types/
│   │   ├── contact.ts
│   │   ├── company.ts
│   │   ├── deal.ts
│   │   ├── activity.ts
│   │   ├── meeting.ts
│   │   ├── note.ts
│   │   ├── attachment.ts
│   │   ├── user.ts
│   │   ├── settings.ts
│   │   ├── api.ts                   # ApiResponse<T>, PaginationMeta, ErrorResponse, etc.
│   │   └── index.ts                 # Re-exports all types
│   ├── tsconfig.json
│   └── package.json
├── .env.example                     # Example environment variables (no secrets)
├── .gitignore
├── docker-compose.yml               # PostgreSQL for local development
├── package.json                     # Root package.json (workspaces)
├── tsconfig.base.json               # Shared TypeScript base config
└── vitest.config.ts                 # Root Vitest configuration
```

## 3.4 Migration Strategy

### Naming Convention

All migrations use the format: `YYYYMMDDHHMMSS_descriptive_name.ts`

Example sequence:
```
20240101000001_create_settings_table.ts
20240101000002_create_users_table.ts
20240101000003_create_sessions_table.ts
20240101000004_create_contacts_table.ts
20240101000005_create_companies_table.ts
20240101000006_create_deal_stages_table.ts
20240101000007_create_deals_table.ts
20240101000008_create_activities_table.ts
20240101000009_create_activity_types_table.ts
20240101000010_create_meetings_table.ts
20240101000011_create_meeting_participants_table.ts
20240101000012_create_notes_table.ts
20240101000013_create_attachments_table.ts
20240101000014_create_api_keys_table.ts
20240101000017_create_email_templates_table.ts
20240101000018_create_audit_log_table.ts
20240101000019_add_indexes.ts
```

### Migration Rules

1. Every migration MUST have a working `down()` function that fully reverses the `up()`.
2. Migrations MUST be idempotent — running them on an already-migrated database should not error.
3. Never modify an existing migration after it has been committed. Create a new migration for changes.
4. All foreign keys MUST have explicit `ON DELETE` behavior (CASCADE, SET NULL, or RESTRICT — never leave it to default).
5. All tables MUST have `created_at` and `updated_at` columns with default `CURRENT_TIMESTAMP`.
6. All UUID primary keys use database-side generation via `gen_random_uuid()` as the column default. Do NOT generate UUIDs application-side for primary keys.
7. Indexes are created in a dedicated migration for clarity.

## 3.5 Environment Configuration

### `.env.example`

```env
# Application
NODE_ENV=development
PORT=3000
APP_DOMAIN=localhost:5173
APP_URL=http://localhost:5173

# Database
DATABASE_URL=postgresql://crm_user:crm_password@localhost:5432/crm_db
DB_POOL_MIN=2
DB_POOL_MAX=10
RUN_MIGRATIONS=true

# Session
SESSION_SECRET=your-64-character-minimum-random-string-here-replace-this-immediately

# API Key Hashing
API_KEY_SECRET=

# CORS
CORS_ORIGIN=http://localhost:5173

# Email (Brevo) — optional
BREVO_API_KEY=
BREVO_SENDER_EMAIL=
BREVO_SENDER_NAME=

# File Storage (Cloudflare R2) — optional
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET_NAME=
R2_PUBLIC_URL=

# Rate Limiting
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100

# Logging
LOG_LEVEL=debug

# Cookie
COOKIE_DOMAIN=localhost
```

### Development Setup

```bash
# 1. Clone repo
git clone <repo-url> && cd crm

# 2. Install dependencies (npm workspaces)
npm install

# 3. Start PostgreSQL (via Docker Compose)
docker compose up -d postgres

# 4. Copy env file and fill in values
cp .env.example .env

# 5. Run migrations and seeds
npm run db:migrate
npm run db:seed

# 6. Start development servers (concurrently)
npm run dev   # runs both client (Vite, port 5173) and server (Express, port 3000)
```

### `package.json` (root) Scripts

```json
{
  "scripts": {
    "dev": "concurrently \"npm run dev:server\" \"npm run dev:client\"",
    "dev:server": "npm run dev --workspace=server",
    "dev:client": "npm run dev --workspace=client",
    "build": "npm run build --workspace=shared && npm run build --workspace=server && npm run build --workspace=client",
    "start": "npm run start --workspace=server",
    "db:migrate": "npm run migrate --workspace=server",
    "db:rollback": "npm run migrate:rollback --workspace=server",
    "db:seed": "npm run seed --workspace=server",
    "lint": "npm run lint --workspaces",
    "typecheck": "npm run typecheck --workspaces",
    "test": "vitest run",
    "test:server": "vitest run --project server",
    "test:client": "vitest run --project client",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

---

# SECTION 4: Coding Best Practices

These coding principles are NON-NEGOTIABLE requirements. The executing AI agent MUST follow every single one. Deviation from these practices is a defect.

## 4.1 Separation of Concerns

**Rule:** The request/response lifecycle follows a strict layer chain: **Routes -> Controller -> Service -> Database (via Knex)**. No layer may skip levels.

**Routes (`*.routes.ts`):**
- Define HTTP method, path, middleware chain (auth, validation, rate limiting), and controller method.
- MUST NOT contain any business logic, database calls, or data transformation.
- Example:
  ```typescript
  router.get('/', authenticate, contactsController.getAll);
  router.get('/:id', authenticate, contactsController.getById);
  router.post('/', authenticate, validateBody(createContactSchema), contactsController.create);
  router.patch('/:id', authenticate, validateBody(updateContactSchema), contactsController.update);
  router.delete('/:id', authenticate, contactsController.delete);
  ```

**Controllers (`*.controller.ts`):**
- Extract data from `req` (params, query, body, user).
- Call the appropriate service method with extracted data.
- Format the service response using the standard response builder.
- Send the HTTP response.
- MUST NOT contain business logic, direct database calls, or complex data transformation.
- MUST NOT import Knex or any database module.
- MUST handle errors by either catching service errors and formatting them, or letting them propagate to the error handler middleware.
- Example:
  ```typescript
  async getAll(req: Request, res: Response, next: NextFunction) {
    try {
      const params = parsePaginationParams(req.query);
      const filters = parseContactFilters(req.query);
      const result = await contactsService.getAll(params, filters);
      res.json(successResponse(result.data, result.meta));
    } catch (error) {
      next(error);
    }
  }
  ```

**Services (`*.service.ts`):**
- Contain ALL business logic: validation beyond input format (e.g., checking duplicates), authorization checks, data transformation, cross-entity operations.
- Call the database via Knex queries.
- Return plain data objects (never Express req/res objects).
- Throw typed errors (NotFoundError, ValidationError, etc.) that the error handler middleware translates to HTTP responses.
- MAY call other services (e.g., `contactsService` calling `activitiesService` to log an activity on creation).
- MUST NOT import Express types (Request, Response).
- Example:
  ```typescript
  async getAll(params: PaginationParams, filters: ContactFilters) {
    const query = knex('contacts')
      .modify(applyFilters, filters)
      .modify(applySorting, params)
      .modify(applyPagination, params);
    const [data, countResult] = await Promise.all([
      query.clone().select('*'),
      knex('contacts').modify(applyFilters, filters).count('* as total').first(),
    ]);
    return { data, meta: buildPaginationMeta(params, countResult.total) };
  }
  ```

**Violation examples (NEVER do these):**
- Controller queries the database directly: `const contacts = await knex('contacts').select('*');` in a controller file.
- Route handler contains business logic: `router.get('/', async (req, res) => { const data = await knex('contacts')...; res.json(data); });`
- Service accesses `req.body` or calls `res.json()`.

## 4.2 Componentization

**Rule:** Any UI element that appears in more than one place MUST be extracted into a reusable component. No copy-pasting UI code.

**Mandatory shared components (must exist before any feature page is built):**
- `DataTable` — Generic table with column definitions, sorting, filtering, pagination, row selection, bulk actions. Used by: contacts list, companies list, deals list, activities list, meetings list, users list.
- `PageHeader` — Title, breadcrumbs, action buttons. Used by: every page.
- `EntityForm` — Pattern for create/edit forms with React Hook Form + Zod. Each entity has its own form component that uses this pattern.
- `ConfirmDialog` — "Are you sure?" dialog with customizable title, message, and confirm/cancel buttons. Used by: delete operations, bulk operations, dangerous actions.
- `EmptyState` — "No data yet" placeholder with icon, message, and optional CTA button. Used by: every list page when empty, every tab when empty.
- `LoadingSpinner` — Centered spinner. Used by: Suspense fallbacks, data loading states, button loading states.
- `StatusBadge` — Colored pill/badge. Used by: deal stages, contact types, activity types, user roles.
- `Timeline` — Chronological activity feed. Used by: contact detail, company detail, deal detail.
- `SearchableSelect` — Combobox that searches via API. Used by: contact picker, company picker, user picker (owner assignment).
- `FileUpload` — Drag-and-drop zone with progress bar. Used by: attachment upload on any entity.

**Component API contract:**
- All components accept typed props (TypeScript interface, never `any`).
- All components that render lists accept `isLoading` and `isEmpty` props (or equivalent) and handle both states internally.
- All interactive components support `disabled` prop.
- All components that trigger async actions expose `isLoading`/`isPending` state.

## 4.3 DRY (Don't Repeat Yourself)

**Rule:** If logic appears in more than one place, extract it. Zealously.

**Required shared utilities:**

Backend:
- `pagination.ts` — Parse `page`, `limit`, `sort`, `order` from query params. Build `PaginationMeta` response object. Apply pagination to Knex query. Default page=1, limit=25, max limit=100.
- `filters.ts` — Parse filter query params (e.g., `?type=Lead&created_after=2024-01-01`). Apply to Knex query. Support operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `like`, `ilike`, `in`, `not_in`, `is_null`, `is_not_null`.
- `response.ts` — `successResponse(data, meta?)`, `errorResponse(code, message, details?)`. All API responses MUST use these builders.
- `errors.ts` — `AppError` base class (extends `Error`, adds `statusCode` and `code`). Subclasses: `NotFoundError` (404), `ValidationError` (400), `UnauthorizedError` (401), `ForbiddenError` (403), `ConflictError` (409), `RateLimitError` (429).

Frontend:
- `api-client.ts` — Single Axios instance with interceptors.
- `utils.ts` — `cn()` (class name merger), `formatCurrency()`, `formatDate()`, `formatRelativeTime()`, `truncate()`, `generateInitials()`, `debounce()`.
- `validations.ts` — Shared Zod schemas for email, phone, URL, UUID, date range, pagination params.

## 4.4 Type Safety

**Rule:** Zero `any` types. Every function parameter, return value, and variable MUST be typed.

**Shared types (in `shared/types/`):**
- Every database entity has a corresponding TypeScript interface matching column names (snake_case in DB -> camelCase in TypeScript via the Knex transformation layer).
- DTOs (Data Transfer Objects) for create and update operations: `CreateContactDto`, `UpdateContactDto` (uses `Partial<CreateContactDto>` where applicable). DTOs are inferred from Zod schemas via `z.infer<>`.
- API response types: `ApiResponse<T>`, `PaginatedResponse<T>`, `ErrorResponse`.
- Enum types for all fixed-value fields (roles, activity types, deal stages, etc.).

**Strict rules:**
- No `as any` casts.
- No `@ts-ignore` or `@ts-expect-error` comments (if one is absolutely necessary, it MUST include a detailed comment explaining why, and the agent should flag it for human review).
- No `Function` type (use specific function signatures).
- No implicit `any` (TypeScript strict mode catches this).
- All event handlers are typed: `onChange: (e: React.ChangeEvent<HTMLInputElement>) => void`.
- All API responses are typed with generics: `apiClient.get<ApiResponse<Contact[]>>(...)`.

## 4.5 Error Handling

**Rule:** Errors are handled gracefully at every layer. No unhandled promise rejections. No silent failures.

**Backend error handling architecture:**

1. **Custom error classes** (`server/src/utils/errors.ts`):
   ```typescript
   class AppError extends Error {
     constructor(
       public statusCode: number,
       public message: string,
       public code: string,
       public details?: { field: string; message: string }[] | null,
     ) {
       super(message);
       this.name = this.constructor.name;
       Error.captureStackTrace(this, this.constructor);
     }
   }

   class NotFoundError extends AppError {
     constructor(resource: string, id?: string) {
       super(404, `${resource}${id ? ` with id ${id}` : ''} not found`, 'NOT_FOUND');
     }
   }

   class ValidationError extends AppError {
     constructor(message: string, details?: { field: string; message: string }[]) {
       super(400, message, 'VALIDATION_ERROR', details);
     }
   }

   class UnauthorizedError extends AppError {
     constructor(message = 'Authentication required') {
       super(401, message, 'UNAUTHORIZED');
     }
   }

   class ForbiddenError extends AppError {
     constructor(message = 'Insufficient permissions') {
       super(403, message, 'FORBIDDEN');
     }
   }

   class ConflictError extends AppError {
     constructor(message: string) {
       super(409, message, 'CONFLICT');
     }
   }
   ```

2. **Centralized error handler** (`server/src/middleware/errorHandler.middleware.ts`):
   ```typescript
   function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
     // Log the error
     logger.error({
       message: err.message,
       stack: err.stack,
       requestId: req.requestId,
       method: req.method,
       url: req.originalUrl,
       userId: req.user?.id,
     });

     // Known application error
     if (err instanceof AppError) {
       return res.status(err.statusCode).json({
         success: false,
         error: {
           code: err.code,
           message: err.message,
           details: err.details || null,
         },
       });
     }

     // Knex-specific errors
     if ((err as any).code === '23505') { // unique constraint violation
       return res.status(409).json({
         success: false,
         error: {
           code: 'CONFLICT',
           message: 'A record with this value already exists',
           details: null,
         },
       });
     }
     if ((err as any).code === '23503') { // foreign key violation
       return res.status(400).json({
         success: false,
         error: {
           code: 'FOREIGN_KEY_VIOLATION',
           message: 'Referenced record does not exist',
           details: null,
         },
       });
     }

     // Unknown error — don't leak details in production
     const message = process.env.NODE_ENV === 'production'
       ? 'An unexpected error occurred'
       : err.message;
     res.status(500).json({
       success: false,
       error: {
         code: 'INTERNAL_ERROR',
         message,
         details: null,
       },
     });
   }
   ```

3. **Service layer:** Throws typed errors. NEVER returns error objects — always throws.
   ```typescript
   // CORRECT:
   const contact = await knex('contacts').where({ id }).first();
   if (!contact) throw new NotFoundError('Contact', id);

   // WRONG:
   if (!contact) return { error: 'Not found' };
   ```

4. **Controller layer:** Wraps service calls in try/catch and passes errors to `next()`.
   ```typescript
   // CORRECT:
   try { ... } catch (error) { next(error); }

   // WRONG:
   try { ... } catch (error) { res.status(500).json({ error: error.message }); }
   ```

**Frontend error handling:**

1. **API errors:** Axios interceptor catches 401/403/5xx globally. Feature-specific error handling in mutation `onError` callbacks.
2. **Form errors:** Server validation errors (400 with field-level details) are mapped to React Hook Form `setError()` calls. General errors show as toast notifications.
3. **Error boundary:** Top-level `ErrorBoundary` component catches React rendering errors and shows a "Something went wrong" page with a "Reload" button.
4. **Loading/error states:** Every data-fetching component handles three states: loading (skeleton/spinner), error (error message + retry button), and success (actual content). No component should render nothing — always show something.

## 4.6 Validation

**Rule:** NEVER trust client input. Validate on both frontend and backend. Backend is the source of truth.

**Frontend validation (Zod):**
- Every form has a Zod schema that defines the exact shape and constraints of the data.
- Schemas live in the shared package: `shared/schemas/contact.schemas.ts`.
- Schemas are used as React Hook Form resolvers via `@hookform/resolvers/zod`.
- Validation runs on `onBlur` (per field) and `onSubmit` (full form).
- Error messages are user-friendly (not "Expected string, received undefined" — instead "First name is required").

**Backend validation (Zod):**
- Every mutation endpoint (POST, PUT, PATCH, DELETE with body) uses the `validateBody()` middleware with a Zod schema from the shared package.
- The same schemas used on the frontend are used on the backend. One source of truth.
- Additional business-rule validation (uniqueness, referential integrity, authorization) happens in the service layer.

**Validation layers (what each layer checks):**

| Check                  | Frontend (Zod) | Backend (Zod middleware) | Backend (Service) |
|------------------------|:-:|:-:|:-:|
| Required fields        | X | X |   |
| Type correctness       | X | X |   |
| Format (email, URL)    | X | X |   |
| String length          | X | X |   |
| Number range           | X | X |   |
| Enum values            | X | X |   |
| Uniqueness (email)     |   |   | X |
| Referential integrity  |   |   | X |
| Business rules         |   |   | X |
| Authorization          |   |   | X |

## 4.7 Database Practices

**Rule:** All schema changes via Knex migrations. All queries via Knex. Never raw unparameterized SQL.

**Migration rules:**
1. Each migration creates or modifies exactly one table (with exceptions for related junction tables).
2. Every `up()` has a corresponding `down()` that fully reverses it.
3. Use `knex.schema.hasTable()` checks for idempotency.
4. All foreign keys use `references('id').inTable('table_name')` with explicit `.onDelete('CASCADE' | 'SET NULL' | 'RESTRICT')`.
5. All text columns have explicit max length or use `text` type (not unbounded `varchar`).
6. All monetary values stored as `DECIMAL(15,2)`.
7. All dates stored as `timestamptz` (timestamp with timezone).
8. All boolean columns have explicit `defaultTo(false)` or `defaultTo(true)`.
9. Indexes are created for: all foreign key columns, all columns used in WHERE clauses, all columns used in ORDER BY, all columns used in search.
10. All UUID primary keys use `gen_random_uuid()` as the database default.

**Query rules:**
1. Always use Knex query builder methods. Avoid `knex.raw()` except for complex operations (window functions, etc.).
2. When using `knex.raw()`, ALWAYS use parameterized bindings: `knex.raw('? = ANY(tags)', [tag])`.
3. Never build SQL strings with concatenation or template literals.
4. Always `SELECT` specific columns in production queries (not `SELECT *`), except in development/prototyping.
5. Always use transactions for multi-table operations:
   ```typescript
   await knex.transaction(async (trx) => {
     const contact = await trx('contacts').insert(contactData).returning('*');
     await trx('activities').insert({ ...activityData, contact_id: contact[0].id });
   });
   ```

## 4.8 Security

**Rule:** Security is not optional. Every endpoint, every input, every response is secured.

**Mandatory security measures:**

1. **helmet:** Enabled on all routes. Sets security headers (X-Content-Type-Options, X-Frame-Options, CSP, HSTS, etc.).

2. **CORS:** Strict origin whitelist. Only the configured frontend domain is allowed. Credentials: true (for cookies).

3. **Rate limiting:**
   - Global: Configurable requests per minute per IP.
   - Auth endpoints: Stricter limits (10 per 15 minutes) to prevent brute force.
   - File upload endpoints: 20 per minute to prevent storage abuse.

4. **CSRF protection:** SameSite=Lax cookies + CORS origin check. For API key auth, CSRF is not applicable (no cookies involved).

5. **Input sanitization:** All string inputs trimmed. HTML stripped from non-rich-text fields. File names sanitized (alphanumeric, hyphens, dots only).

6. **SQL injection prevention:** Exclusively using Knex parameterized queries. No string concatenation in queries ever.

7. **XSS prevention:** React's default JSX escaping handles most cases. `dangerouslySetInnerHTML` is NEVER used except for rendering sanitized Markdown (using `sanitize-html`). SVG uploads are blocked (XSS vector). CSP headers restrict inline scripts.

8. **Session security:**
   - HTTP-only cookies (not accessible via JavaScript).
   - Secure flag in production (HTTPS only).
   - SameSite=Lax (prevents CSRF from cross-origin requests).
   - Session regeneration on login (prevent session fixation).
   - Session destruction on logout.
   - 24-hour rolling expiry.

9. **Password security:**
   - bcrypt with cost factor 12.
   - Minimum 10 characters with complexity requirements.
   - Password reset tokens: cryptographically random, single-use, 1-hour expiry.
   - Failed login attempt tracking (lock account after 10 failed attempts for 30 minutes).

10. **File upload security:**
    - MIME type validation via magic bytes (using `file-type` package), not client-provided Content-Type.
    - SVG explicitly excluded (XSS risk).
    - File size limit (25MB).
    - File name sanitization.
    - Files are stored in R2 (not on the application server filesystem).
    - Files are served via presigned URLs with 5-minute expiry (never directly accessible).

11. **API key security:**
    - Keys displayed once on creation, then only the hash is stored.
    - Keys can be revoked instantly.
    - Rate limited separately from session auth.
    - Keys are validated via constant-time comparison (bcrypt.compare, which is inherently constant-time).

12. **Sensitive data:**
    - Passwords NEVER appear in logs, API responses, or error messages.
    - API keys NEVER appear in logs or API responses (after initial creation).
    - Email template content is sanitized before rendering.
    - Database connection strings NEVER appear in frontend code or API responses.

## 4.9 Configuration

**Rule:** All configuration comes from environment variables. No hardcoded values. No magic strings.

**Centralized config module (`server/src/config/index.ts`):**
- Reads ALL environment variables in one place.
- Validates all required variables at startup.
- Exports a single typed, frozen config object.
- Any missing required variable causes immediate process exit with a clear error message.

**Rules:**
1. No `process.env` access outside of the config module. Every other file imports from `config`.
2. All config values have explicit types (string, number, boolean).
3. All optional config values have documented defaults.
4. All secrets (API keys, passwords, session secrets) have no defaults — they MUST be explicitly set.
5. Config object is `as const` / `Object.freeze()` to prevent runtime mutation.
6. Frontend config (`client/src/config/index.ts`) reads from Vite's `import.meta.env` and MUST NOT contain any secrets (only `VITE_` prefixed env vars are exposed to the frontend).

## 4.10 Logging

**Rule:** Structured, leveled, traceable logging. Never log sensitive data.

**Log levels and usage:**
- `error`: Unhandled errors, failed critical operations (database connection failure, email service failure). Always includes stack trace.
- `warn`: Recoverable issues (deprecated API usage, rate limit approaching, missing optional config).
- `info`: Significant business events (user login, user created, deal closed, import completed). Request/response logging.
- `debug`: Detailed operation tracing (query parameters, service method entry/exit, cache hit/miss). Disabled in production by default.

**Request logging format:**
```json
{
  "level": "info",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "method": "GET",
  "url": "/api/v1/contacts",
  "statusCode": 200,
  "duration": 45,
  "userId": "user-uuid-here",
  "ip": "192.168.1.1"
}
```

**Forbidden in logs:**
- Passwords (plaintext or hashed)
- API keys or tokens
- Session IDs
- Credit card numbers or financial data
- Full email bodies
- File contents
- Any PII beyond user ID (no email addresses, names, or phone numbers in logs)

## 4.11 Naming Conventions

**Rule:** Consistent naming across the entire codebase. No exceptions.

| Context                  | Convention       | Example                          |
|--------------------------|------------------|----------------------------------|
| TypeScript variables     | camelCase        | `contactList`, `isLoading`       |
| TypeScript functions     | camelCase        | `getContactById`, `handleSubmit` |
| TypeScript constants     | UPPER_SNAKE_CASE | `MAX_FILE_SIZE`, `API_BASE_URL`  |
| React components         | PascalCase       | `ContactForm`, `DataTable`       |
| React component files    | PascalCase       | `ContactForm.tsx`, `DataTable.tsx`|
| TypeScript interfaces    | PascalCase       | `Contact`, `CreateContactDto`    |
| TypeScript enums         | PascalCase       | `UserRole`, `DealStage`          |
| TypeScript enum values   | UPPER_SNAKE_CASE | `UserRole.ADMIN`, `UserRole.MEMBER` |
| CSS classes              | kebab-case       | Via Tailwind utility classes     |
| Database tables          | snake_case plural| `contacts`, `deal_stages`        |
| Database columns         | snake_case       | `first_name`, `created_at`       |
| API URL paths            | kebab-case       | `/api/v1/contacts`, `/api/v1/api-keys` |
| API query params         | snake_case       | `?sort_by=created_at&page=1`     |
| API request/response body| camelCase         | `{ firstName, lastName }`        |
| Environment variables    | UPPER_SNAKE_CASE | `DATABASE_URL`, `BREVO_API_KEY`  |
| File names (non-component)| kebab-case or dot-separated | `api-client.ts`, `contacts.service.ts` |
| Migration files          | snake_case with timestamp | `20240101000001_create_contacts_table.ts` |

**Body/DB conversion:** The API accepts and returns camelCase JSON. The database uses snake_case. Conversion happens automatically via Knex hooks (see Section 3.2.13).

## 4.12 File Organization

**Rule:** Feature-based, not type-based. Related files live together.

**WRONG (type-based):**
```
src/
├── controllers/
│   ├── contacts.controller.ts
│   ├── companies.controller.ts
├── services/
│   ├── contacts.service.ts
│   ├── companies.service.ts
├── routes/
│   ├── contacts.routes.ts
│   ├── companies.routes.ts
```

**CORRECT (feature-based):**
```
src/
├── features/
│   ├── contacts/
│   │   ├── contacts.routes.ts
│   │   ├── contacts.controller.ts
│   │   ├── contacts.service.ts
│   │   ├── contacts.service.test.ts
│   │   └── contacts.types.ts
│   ├── companies/
│   │   ├── companies.routes.ts
│   │   ├── companies.controller.ts
│   │   ├── companies.service.ts
│   │   ├── companies.service.test.ts
│   │   └── companies.types.ts
```

**Why:** When working on the "contacts" feature, all related files are in one directory. No jumping between 5 different directories. Adding a new feature means adding one directory, not modifying 5 existing directories.

**Exception:** Truly shared code (middleware, utils, config, shared services) lives in its own top-level directories because it belongs to no single feature.

## 4.13 Canonical API Response & Pagination Formats

**These response/error/pagination formats are CANONICAL. Do not redefine them in feature sections. All feature sections reference these formats.**

### Success Response

```json
{
  "success": true,
  "data": T | T[],
  "meta": {
    "page": 1,
    "limit": 25,
    "total": 150,
    "totalPages": 6
  }
}
```

- For single-entity responses, `data` is the object `T`. `meta` is omitted.
- For list responses, `data` is an array `T[]`. `meta` is always present.
- For delete/no-content responses, `data` is `null`. `meta` is omitted.

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human readable message",
    "details": [{ "field": "email", "message": "Invalid format" }]
  }
}
```

- `error.code` is a machine-readable error code string (e.g., `NOT_FOUND`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `CONFLICT`, `INTERNAL_ERROR`).
- `error.message` is a human-readable string.
- `error.details` is an array of field-level errors for validation errors, or `null` for other error types.

### Pagination Parameters

All list endpoints accept these query parameters:
- `page` — Page number, 1-indexed. Default: `1`.
- `limit` — Items per page. Default: `25`. Maximum: `100`.

**These are the ONLY pagination parameter names.** Not `pageSize`, not `per_page`, not `cursor`. Pagination is OFFSET-BASED everywhere.

### HTTP Status Code Usage

| Status | Meaning                    | When to use                                      |
|--------|----------------------------|--------------------------------------------------|
| 200    | OK                         | Successful GET, PUT, PATCH, DELETE                |
| 201    | Created                    | Successful POST that creates a resource           |
| 400    | Bad Request                | Validation error, malformed request               |
| 401    | Unauthorized               | Not authenticated (no session, expired session)   |
| 403    | Forbidden                  | Authenticated but insufficient permissions        |
| 404    | Not Found                  | Resource does not exist                           |
| 409    | Conflict                   | Duplicate record (unique constraint violation)     |
| 429    | Too Many Requests          | Rate limit exceeded                               |
| 500    | Internal Server Error      | Unhandled server error                            |

## 4.14 Git Practices

**Rule:** Atomic, meaningful commits. One logical change per commit.

**Commit message format:**
```
<type>(<scope>): <short description>

<optional body with more detail>
```

**Types:** `feat`, `fix`, `refactor`, `style`, `docs`, `test`, `chore`, `perf`.

**Scope:** Feature name or module (`contacts`, `auth`, `ui`, `config`, `db`).

**Examples:**
```
feat(contacts): add contact creation endpoint and form
fix(deals): correct pipeline stage ordering on drag-and-drop
refactor(auth): extract session validation into middleware
chore(db): add index on contacts.email column
test(contacts): add integration tests for CRUD endpoints
```

**Rules:**
1. Never commit `node_modules/`, `.env`, `dist/`, `logs/`, or any file in `.gitignore`.
2. Never commit with `--no-verify` (always run pre-commit hooks).
3. Each commit should leave the codebase in a working state (no broken builds).
4. Migrations are committed in the same commit as the code that uses them.
5. Feature branches are preferred: `feat/contacts-module`, `fix/login-redirect`.

## 4.15 Testing

**Rule:** Every API endpoint has integration tests. Vitest is the sole test runner.

**Backend integration tests:**
- Framework: Vitest + supertest.
- Each feature module has colocated `*.test.ts` files (e.g., `contacts.service.test.ts`).
- Tests use a dedicated test database (`crm_test`), created/destroyed per test suite.
- Tests cover: successful operations (happy path), validation errors (each required field), authentication (unauthenticated access), authorization (member accessing admin endpoints), edge cases (duplicate email, empty lists).
- Each test is independent — no shared state between tests. Use transactions that rollback after each test, or truncate tables in `beforeEach`.
- **Minimum 80% code coverage on service layer files.**

**Test structure:**
```typescript
describe('POST /api/v1/contacts', () => {
  it('should create a contact with valid data', async () => { ... });
  it('should return 400 if first_name is missing', async () => { ... });
  it('should return 400 if email format is invalid', async () => { ... });
  it('should return 409 if email already exists', async () => { ... });
  it('should return 401 if not authenticated', async () => { ... });
  it('should create an activity log entry on creation', async () => { ... });
});

describe('GET /api/v1/contacts', () => {
  it('should return paginated contacts', async () => { ... });
  it('should filter by type', async () => { ... });
  it('should search by name via ILIKE', async () => { ... });
  it('should sort by created_at descending by default', async () => { ... });
});
```

**Frontend component tests:**
- Framework: Vitest + @testing-library/react + @testing-library/jest-dom.
- Colocated as `*.test.tsx` next to the component files.
- Test user interactions, form validation, loading/error states.

**Test commands:**
```bash
npm run test              # Run all tests (vitest)
npm run test:server       # Backend tests only
npm run test:client       # Frontend tests only
npm run test:coverage     # Tests with coverage report
```

## 4.16 Accessibility

**Rule:** All interactive elements keyboard-navigable. All form inputs have labels. ARIA roles for dynamic content. Color contrast ratios meet WCAG AA (4.5:1 for text).

**Requirements:**
- Every `<button>`, `<a>`, `<input>`, `<select>`, and `<textarea>` must be reachable via Tab key.
- Every form input must have an associated `<label>` element (via `htmlFor` or wrapping).
- Dynamic content updates (toasts, modals, live regions) must use appropriate ARIA attributes (`role="alert"`, `aria-live="polite"`, `aria-modal="true"`).
- Focus management: when a modal opens, focus moves into the modal. When it closes, focus returns to the trigger element.
- Skip-to-content link as the first focusable element on every page.
- Images must have descriptive `alt` text (or `alt=""` for decorative images).
- Color must not be the sole means of conveying information (always pair with text/icons).

## 4.17 Error Recovery

**Rule:** Graceful degradation when external services (Brevo, R2) are unavailable. Log errors, show user-friendly messages, never crash.

**Requirements:**
- If Brevo is unreachable, email-sending operations log the error and return success to the caller. The user sees a warning that the email could not be sent, but the primary operation (e.g., user creation, password reset) still completes.
- If R2 is unreachable, file upload operations return a clear error message ("File storage is temporarily unavailable. Please try again later.") without crashing the server.
- Database connection failures trigger graceful shutdown with clear log messages.
- Any unhandled promise rejection or uncaught exception is caught by process-level handlers, logged, and results in a controlled restart — never a silent death.

## 4.18 Regression Testing

**Rule:** After completing each milestone, run ALL previous milestone test suites. No milestone is complete until all prior tests still pass.

This ensures that new feature work does not break existing functionality. The CI pipeline (or manual test run) must execute the full test suite, not just tests for the current milestone.


### 4.19 Responsive Design
Every page and component must be built mobile-first using Tailwind's responsive prefixes (`sm:`, `md:`, `lg:`). Breakpoints: `sm: 640px`, `md: 768px`, `lg: 1024px`, `xl: 1280px`.

- **DataTable component**: Below 768px, render as stacked cards instead of table rows. Each card shows key fields with expand/collapse for full details.
- **Sidebar/Navigation**: Below 768px, collapse to hamburger menu with slide-out drawer.
- **Modals**: Below 640px, render as full-screen sheets.
- **Forms**: Always single-column below 768px. Two-column layout available at `lg:` and above.
- **Kanban board**: Below 768px, render as a single vertical column with stage headers as collapsible sections.
- **Calendar**: Below 768px, switch from grid to list view.
- **Dashboard**: All widgets stack vertically on mobile. Charts resize to fill container width.
- **Touch targets**: All buttons, links, and interactive elements must be minimum 44×44px.
- **Testing**: Every page must be verified at 320px, 768px, and 1024px widths.
---

### 4.20 Entity Selectors — No Raw ID Inputs
Whenever a form field references another entity (company on a contact, contact on a deal, etc.), NEVER use a plain text/UUID input. Always use a reusable `EntitySelector` component — a searchable autocomplete dropdown.

**`EntitySelector` behavior:**
- Debounced text input (300ms) that calls the entity's list API with `?search=` parameter
- Dropdown shows matching results with name + key identifying info (e.g., company name + domain, contact name + email)
- Single-select by default, multi-select variant for attendees
- Clear button to remove selection
- Selected value shows entity name (not UUID)
- `filterParams` prop allows cascading filters (e.g., only show contacts at a specific company)

**Cascading entity selection rules:**
- **Deal form:** If company is selected, contact selector filters to `?company_id={selectedCompanyId}`. If contact is selected first and they belong to a company, auto-populate the company field.
- **Activity/Meeting form:** If contact is selected and has a company, auto-populate company. If company is selected, deal selector filters to `?company_id=`. If deal is selected, auto-populate company and contact from the deal.
- **Contact form:** Company selector searches all companies. Owner selector searches all users.
- **General rule:** When a parent entity is selected, child selectors filter to that parent. When a child is selected and has a parent, auto-populate the parent. Never require the user to know or type a UUID.

---

*End of Sections 1-4. Subsequent sections will cover Database Schema, API Endpoints, Frontend Pages, and Implementation Order.*
# SECTION 5: Database Schema & Migrations

This section is the CANONICAL schema definition for the CRM system's PostgreSQL database. All other sections reference these definitions. Every table, column, type, constraint, index, and relationship is specified here. Implementations MUST follow these definitions exactly.

---

## 5.1 General Conventions

- **Primary Keys**: All tables use `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` unless otherwise noted (e.g., `sessions.sid`). UUIDs are generated at the database level.
- **Timestamps**: All `created_at` columns default to `NOW()`. All `updated_at` columns default to `NOW()` and are automatically updated via the `trigger_set_updated_at` database trigger (see Section 5.3).
- **Column Naming**: All column names use `snake_case`. No exceptions.
- **No Soft Deletes**: No entity table uses a `deleted_at` column. Users are deactivated via `is_active = false`. All other entities are hard-deleted with appropriate CASCADE or SET NULL behavior as defined per foreign key.
- **Foreign Key Naming**: Foreign key constraints follow the pattern `fk_{table}_{column}` (e.g., `fk_contacts_company_id`).
- **Index Naming**: Indexes follow the pattern `idx_{table}_{column(s)}` (e.g., `idx_contacts_email`).
- **JSONB Defaults**: All `JSONB` columns that default to an empty object use `DEFAULT '{}'::jsonb`. All `JSONB` columns that default to an array use the specified default cast to `jsonb`.
- **Array Defaults**: All `TEXT[]` columns default to `DEFAULT '{}'` (empty array literal).
- **String Lengths**: All `VARCHAR` lengths are maximum lengths, not fixed lengths.
- **Collation**: All string comparisons use the database default collation (typically `en_US.UTF-8`). Email comparisons MUST be case-insensitive at the application layer (store lowercase).
- **Migration Tool**: Knex.js. All migrations use Knex migration syntax with TypeScript files. Migration naming: `YYYYMMDDHHMMSS_description.ts`.

---

## 5.2 Table Definitions

### 5.2.1 `users`

Stores all CRM user accounts. Users are never hard-deleted; they are deactivated via `is_active = false`.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique user identifier |
| `email` | `VARCHAR(255)` | `UNIQUE, NOT NULL` | — | User's email address. Stored lowercase. Must be a valid email format. |
| `password_hash` | `VARCHAR(255)` | `NOT NULL` | — | bcrypt hash of the user's password. Minimum password length: 10 characters (enforced at application level). NEVER exposed in API responses. |
| `first_name` | `VARCHAR(100)` | `NOT NULL` | — | User's first name. Must be 1-100 characters. Trimmed of leading/trailing whitespace. |
| `last_name` | `VARCHAR(100)` | `NOT NULL` | — | User's last name. Must be 1-100 characters. Trimmed of leading/trailing whitespace. |
| `role` | `VARCHAR(20)` | `NOT NULL, CHECK (role IN ('admin', 'member'))` | `'member'` | User's system role. Only two values permitted. |
| `avatar_url` | `TEXT` | `NULLABLE` | `NULL` | URL to user's avatar image. If provided, must be a valid URL. |
| `is_active` | `BOOLEAN` | `NOT NULL` | `true` | Whether the user account is active. Inactive users cannot log in. Used for soft-deactivation instead of deletion. |
| `last_login_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | Timestamp of the user's most recent successful login. Updated on every login. |
| `password_reset_token` | `VARCHAR(255)` | `NULLABLE` | `NULL` | SHA-256 hash of the password reset token. Cleared after successful reset or expiration. |
| `password_reset_expires` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | Expiration timestamp for the password reset token. Token is invalid after this time. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the user record was created. Immutable after creation. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the user record was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `users_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_users_email` | `email` | UNIQUE | Automatic from UNIQUE constraint |

**Constraints:**

- `chk_users_role`: `CHECK (role IN ('admin', 'member'))`

**SQL:**

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'member' CHECK (role IN ('admin', 'member')),
    avatar_url TEXT,
    is_active BOOLEAN NOT NULL DEFAULT true,
    last_login_at TIMESTAMPTZ,
    password_reset_token VARCHAR(255),
    password_reset_expires TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### 5.2.2 `sessions`

Managed by `connect-pg-simple`. This table is created automatically by the session middleware, but the schema is defined here for completeness and for the migration to create it upfront.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `sid` | `VARCHAR` | `PRIMARY KEY` | — | Session identifier string |
| `sess` | `JSON` | `NOT NULL` | — | Serialized session data (JSON object containing userId, role, createdAt, cookie metadata) |
| `expire` | `TIMESTAMP(6)` | `NOT NULL` | — | When this session expires. Sessions are cleaned up by `connect-pg-simple` pruning. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `session_pkey` | `sid` | PRIMARY KEY | Automatic |
| `idx_sessions_expire` | `expire` | BTREE | Used by session pruning to efficiently find and delete expired sessions |

**SQL:**

```sql
CREATE TABLE sessions (
    sid VARCHAR PRIMARY KEY,
    sess JSON NOT NULL,
    expire TIMESTAMP(6) NOT NULL
);

CREATE INDEX idx_sessions_expire ON sessions (expire);
```

---

### 5.2.3 `contacts`

Stores all contact (person) records in the CRM. A contact may optionally belong to a company and may optionally be assigned to an owner (user).

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique contact identifier |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE SET NULL` | `NULL` | The company this contact belongs to. Set to NULL if the company is deleted. |
| `owner_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who owns/manages this contact. Set to NULL if the user is deleted. |
| `first_name` | `VARCHAR(100)` | `NOT NULL` | — | Contact's first name. Must be 1-100 characters. Trimmed. |
| `last_name` | `VARCHAR(100)` | `NOT NULL` | — | Contact's last name. Must be 1-100 characters. Trimmed. |
| `email` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Contact's email address. If provided, must be valid email format. Stored lowercase. NOT required to be unique (multiple contacts at same company may share a group email, or duplicates may exist). |
| `phone` | `VARCHAR(50)` | `NULLABLE` | `NULL` | Contact's primary phone number. Stored as-is (no format enforcement at DB level). |
| `mobile` | `VARCHAR(50)` | `NULLABLE` | `NULL` | Contact's mobile phone number. |
| `job_title` | `VARCHAR(150)` | `NULLABLE` | `NULL` | Contact's job title (e.g., "VP of Engineering"). |
| `department` | `VARCHAR(100)` | `NULLABLE` | `NULL` | Contact's department within their company. |
| `address_line1` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Street address line 1. |
| `address_line2` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Street address line 2 (suite, apt, etc.). |
| `city` | `VARCHAR(100)` | `NULLABLE` | `NULL` | City. |
| `state` | `VARCHAR(100)` | `NULLABLE` | `NULL` | State, province, or region. |
| `postal_code` | `VARCHAR(20)` | `NULLABLE` | `NULL` | Postal/ZIP code. |
| `country` | `VARCHAR(100)` | `NULLABLE` | `NULL` | Country name. |
| `source` | `VARCHAR(50)` | `NULLABLE` | `NULL` | How this contact was acquired. Application enforces values: `'website'`, `'referral'`, `'linkedin'`, `'cold_call'`, `'trade_show'`, `'other'`. No DB-level CHECK constraint (to allow future extension without migration), but application MUST validate against the allowed list. |
| `status` | `VARCHAR(20)` | `NOT NULL, CHECK` | `'active'` | Contact's lifecycle status. |
| `tags` | `TEXT[]` | `NOT NULL` | `'{}'` | Array of tag strings. Each tag must be 1-50 characters. No duplicates within the array (enforced at application level). |
| `custom_fields` | `JSONB` | `NOT NULL` | `'{}'::jsonb` | Arbitrary key-value pairs for custom data. Keys must be strings. Values can be strings, numbers, booleans, or null. Nested objects are NOT permitted. |
| `notes_text` | `TEXT` | `NULLABLE` | `NULL` | Quick notes field for short annotations. For longer, structured notes, use the `notes` table. |
| `last_contacted_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | Timestamp of the last activity (call, email, meeting) with this contact. Updated automatically when activities are created. |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this contact. This is NOT affected by user deletion (no ON DELETE action — user deletion is soft-delete only). |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the contact was created. Immutable. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the contact was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `contacts_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_contacts_email` | `email` | BTREE | For lookup by email. Not unique — contacts may share emails. |
| `idx_contacts_company_id` | `company_id` | BTREE | For finding all contacts at a company. |
| `idx_contacts_owner_id` | `owner_id` | BTREE | For finding all contacts owned by a user. |
| `idx_contacts_status` | `status` | BTREE | For filtering contacts by status. |
| `idx_contacts_created_at` | `created_at` | BTREE | For sorting by creation date. |
| `idx_contacts_name` | `(first_name, last_name)` | BTREE | Composite index for name-based searching and sorting. |
| `idx_contacts_custom_fields` | `custom_fields` | GIN | For basic JSONB searching on custom fields. |
| `idx_contacts_tags` | `tags` | GIN | For searching contacts by tag values in the array. |

**Constraints:**

- `chk_contacts_status`: `CHECK (status IN ('active', 'inactive', 'lead', 'prospect', 'customer', 'churned'))`
- `fk_contacts_company_id`: `FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE SET NULL`
- `fk_contacts_owner_id`: `FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE SET NULL`
- `fk_contacts_created_by`: `FOREIGN KEY (created_by) REFERENCES users(id)`

**SQL:**

```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id) ON DELETE SET NULL,
    owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    mobile VARCHAR(50),
    job_title VARCHAR(150),
    department VARCHAR(100),
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100),
    source VARCHAR(50),
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'lead', 'prospect', 'customer', 'churned')),
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}'::jsonb,
    notes_text TEXT,
    last_contacted_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_contacts_email ON contacts (email);
CREATE INDEX idx_contacts_company_id ON contacts (company_id);
CREATE INDEX idx_contacts_owner_id ON contacts (owner_id);
CREATE INDEX idx_contacts_status ON contacts (status);
CREATE INDEX idx_contacts_created_at ON contacts (created_at);
CREATE INDEX idx_contacts_name ON contacts (first_name, last_name);
CREATE INDEX idx_contacts_custom_fields ON contacts USING GIN (custom_fields);
CREATE INDEX idx_contacts_tags ON contacts USING GIN (tags);
```

---

### 5.2.4 `companies`

Stores all company/organization records. Companies can have many contacts and many deals.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique company identifier |
| `name` | `VARCHAR(255)` | `NOT NULL` | — | Company name. Must be 1-255 characters. Trimmed. |
| `domain` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Company's primary web domain (e.g., "acme.com"). Stored without protocol. Used for deduplication hints. |
| `industry` | `VARCHAR(100)` | `NULLABLE` | `NULL` | Industry category (e.g., "Technology", "Healthcare"). Free-text, no enumerated constraint. |
| `size` | `VARCHAR(50)` | `NULLABLE` | `NULL` | Company size range. Application enforces values: `'1-10'`, `'11-50'`, `'51-200'`, `'201-500'`, `'501-1000'`, `'1001-5000'`, `'5000+'`. No DB-level CHECK (to allow future extension). |
| `owner_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who owns/manages this company account. |
| `phone` | `VARCHAR(50)` | `NULLABLE` | `NULL` | Company's main phone number. |
| `email` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Company's general email address. |
| `website` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Company's website URL. Must include protocol (e.g., "https://acme.com"). |
| `address_line1` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Street address line 1. |
| `address_line2` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Street address line 2. |
| `city` | `VARCHAR(100)` | `NULLABLE` | `NULL` | City. |
| `state` | `VARCHAR(100)` | `NULLABLE` | `NULL` | State, province, or region. |
| `postal_code` | `VARCHAR(20)` | `NULLABLE` | `NULL` | Postal/ZIP code. |
| `country` | `VARCHAR(100)` | `NULLABLE` | `NULL` | Country name. |
| `description` | `TEXT` | `NULLABLE` | `NULL` | Free-text description of the company. |
| `tags` | `TEXT[]` | `NOT NULL` | `'{}'` | Array of tag strings. Same constraints as contacts.tags. |
| `custom_fields` | `JSONB` | `NOT NULL` | `'{}'::jsonb` | Arbitrary key-value pairs. Same structure rules as contacts.custom_fields. |
| `annual_revenue` | `DECIMAL(15,2)` | `NULLABLE` | `NULL` | Estimated annual revenue. Must be >= 0 if provided (enforced at application level). |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this company record. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the company was created. Immutable. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the company was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `companies_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_companies_name` | `name` | BTREE | For searching by company name. |
| `idx_companies_domain` | `domain` | BTREE | For lookup and deduplication by domain. |
| `idx_companies_industry` | `industry` | BTREE | For filtering by industry. |
| `idx_companies_owner_id` | `owner_id` | BTREE | For finding all companies owned by a user. |
| `idx_companies_created_at` | `created_at` | BTREE | For sorting by creation date. |

**Constraints:**

- `fk_companies_owner_id`: `FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE SET NULL`
- `fk_companies_created_by`: `FOREIGN KEY (created_by) REFERENCES users(id)`

**SQL:**

```sql
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    domain VARCHAR(255),
    industry VARCHAR(100),
    size VARCHAR(50),
    owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
    phone VARCHAR(50),
    email VARCHAR(255),
    website VARCHAR(255),
    address_line1 VARCHAR(255),
    address_line2 VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(100),
    description TEXT,
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}'::jsonb,
    annual_revenue DECIMAL(15,2),
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_companies_name ON companies (name);
CREATE INDEX idx_companies_domain ON companies (domain);
CREATE INDEX idx_companies_industry ON companies (industry);
CREATE INDEX idx_companies_owner_id ON companies (owner_id);
CREATE INDEX idx_companies_created_at ON companies (created_at);
```

---

### 5.2.5 `deals`

Stores all deal/opportunity records. Deals progress through configurable pipeline stages defined in the `deal_stages` table.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique deal identifier |
| `title` | `VARCHAR(255)` | `NOT NULL` | — | Deal title/name. Must be 1-255 characters. Trimmed. |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE SET NULL` | `NULL` | The company associated with this deal. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE SET NULL` | `NULL` | The primary contact for this deal. |
| `owner_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who owns this deal. |
| `stage` | `VARCHAR(50)` | `NOT NULL` | `'Lead'` | Current pipeline stage name. Validated at application level against the `deal_stages` table. No DB-level CHECK constraint because stages are configurable. |
| `amount` | `DECIMAL(15,2)` | `NULLABLE` | `NULL` | Deal monetary value. Must be >= 0 if provided (application-enforced). |
| `currency` | `VARCHAR(3)` | `NOT NULL` | `'USD'` | ISO 4217 currency code. Application validates against a known list of currency codes. |
| `probability` | `INTEGER` | `NULLABLE, CHECK (probability >= 0 AND probability <= 100)` | `NULL` | Probability of closing as a percentage (0-100). |
| `expected_close_date` | `DATE` | `NULLABLE` | `NULL` | When the deal is expected to close. |
| `actual_close_date` | `DATE` | `NULLABLE` | `NULL` | When the deal actually closed. Set automatically when stage moves to a won or lost stage. Can be overridden manually. |
| `description` | `TEXT` | `NULLABLE` | `NULL` | Free-text description of the deal. |
| `tags` | `TEXT[]` | `NOT NULL` | `'{}'` | Array of tag strings. Same constraints as contacts.tags. |
| `custom_fields` | `JSONB` | `NOT NULL` | `'{}'::jsonb` | Arbitrary key-value pairs. Same rules as contacts.custom_fields. |
| `lost_reason` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Reason the deal was lost. Only meaningful when stage is a lost stage. Application should require this when moving to a lost stage. |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this deal. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the deal was created. Immutable. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the deal was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `deals_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_deals_stage` | `stage` | BTREE | For filtering and grouping deals by pipeline stage. |
| `idx_deals_company_id` | `company_id` | BTREE | For finding all deals with a company. |
| `idx_deals_contact_id` | `contact_id` | BTREE | For finding all deals with a contact. |
| `idx_deals_owner_id` | `owner_id` | BTREE | For finding all deals owned by a user. |
| `idx_deals_expected_close_date` | `expected_close_date` | BTREE | For date-range queries on pipeline. |
| `idx_deals_created_at` | `created_at` | BTREE | For sorting by creation date. |

**Constraints:**

- `chk_deals_probability`: `CHECK (probability >= 0 AND probability <= 100)`
- `fk_deals_company_id`: `FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE SET NULL`
- `fk_deals_contact_id`: `FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE SET NULL`
- `fk_deals_owner_id`: `FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE SET NULL`
- `fk_deals_created_by`: `FOREIGN KEY (created_by) REFERENCES users(id)`

**SQL:**

```sql
CREATE TABLE deals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    company_id UUID REFERENCES companies(id) ON DELETE SET NULL,
    contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,
    owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
    stage VARCHAR(50) NOT NULL DEFAULT 'Lead',
    amount DECIMAL(15,2),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    probability INTEGER CHECK (probability >= 0 AND probability <= 100),
    expected_close_date DATE,
    actual_close_date DATE,
    description TEXT,
    tags TEXT[] NOT NULL DEFAULT '{}',
    custom_fields JSONB NOT NULL DEFAULT '{}'::jsonb,
    lost_reason VARCHAR(255),
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deals_stage ON deals (stage);
CREATE INDEX idx_deals_company_id ON deals (company_id);
CREATE INDEX idx_deals_contact_id ON deals (contact_id);
CREATE INDEX idx_deals_owner_id ON deals (owner_id);
CREATE INDEX idx_deals_expected_close_date ON deals (expected_close_date);
CREATE INDEX idx_deals_created_at ON deals (created_at);
```

---

### 5.2.6 `deal_stages`

Configurable pipeline stages for deals. System stages (where `is_system = true`) cannot be deleted by users. The `stage` column on the `deals` table is validated at application level against the `name` values in this table.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique stage identifier |
| `name` | `VARCHAR(100)` | `NOT NULL, UNIQUE` | — | Stage name. Must be unique. Used as the value stored in `deals.stage`. |
| `display_order` | `INTEGER` | `NOT NULL` | — | Order in which stages appear in the pipeline UI. Lower numbers appear first. |
| `color` | `VARCHAR(7)` | `NOT NULL` | `'#6B7280'` | Hex color code for the stage (e.g., "#3B82F6"). Must be a valid 7-character hex color. |
| `default_probability` | `INTEGER` | `NOT NULL, CHECK (default_probability >= 0 AND default_probability <= 100)` | `0` | Default win probability (0-100%) assigned to deals entering this stage. |
| `is_system` | `BOOLEAN` | `NOT NULL` | `false` | Whether this is a system stage. System stages cannot be deleted by users. |
| `is_won` | `BOOLEAN` | `NOT NULL` | `false` | Whether this stage represents a won deal. When a deal enters a won stage, `actual_close_date` is set. |
| `is_lost` | `BOOLEAN` | `NOT NULL` | `false` | Whether this stage represents a lost deal. When a deal enters a lost stage, `actual_close_date` is set and `lost_reason` is required. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the stage was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the stage was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `deal_stages_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_deal_stages_name` | `name` | UNIQUE | Automatic from UNIQUE constraint |
| `idx_deal_stages_display_order` | `display_order` | BTREE | For ordering stages in pipeline views. |

**Constraints:**

- `chk_deal_stages_probability`: `CHECK (default_probability >= 0 AND default_probability <= 100)`

**Seed Data:**

| name | display_order | default_probability | is_system | is_won | is_lost | color |
|---|---|---|---|---|---|---|
| `Lead` | 1 | 0 | true | false | false | `#6B7280` |
| `Qualified` | 2 | 25 | true | false | false | `#3B82F6` |
| `Proposal` | 3 | 50 | true | false | false | `#F59E0B` |
| `Negotiation` | 4 | 75 | true | false | false | `#8B5CF6` |
| `Closed Won` | 5 | 100 | true | true | false | `#10B981` |
| `Closed Lost` | 6 | 0 | true | false | true | `#EF4444` |

**SQL:**

```sql
CREATE TABLE deal_stages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL UNIQUE,
    display_order INTEGER NOT NULL,
    color VARCHAR(7) NOT NULL DEFAULT '#6B7280',
    default_probability INTEGER NOT NULL DEFAULT 0
        CHECK (default_probability >= 0 AND default_probability <= 100),
    is_system BOOLEAN NOT NULL DEFAULT false,
    is_won BOOLEAN NOT NULL DEFAULT false,
    is_lost BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_deal_stages_display_order ON deal_stages (display_order);
```

---

### 5.2.7 `activities`

Stores all activity records (calls, emails, meetings, notes, tasks, lunches, deadlines). Activities can be linked to contacts, companies, and/or deals. At least one of `contact_id`, `company_id`, or `deal_id` SHOULD be set (enforced at application level, not DB level).

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique activity identifier |
| `type` | `VARCHAR(30)` | `NOT NULL` | — | Type of activity. Validated at application level against the `activity_types` table. No DB-level CHECK constraint because types are configurable. |
| `subject` | `VARCHAR(255)` | `NOT NULL` | — | Activity subject/title. Must be 1-255 characters. |
| `description` | `TEXT` | `NULLABLE` | `NULL` | Detailed description of the activity. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE CASCADE` | `NULL` | Associated contact. Activity is deleted if the contact is deleted. |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE CASCADE` | `NULL` | Associated company. Activity is deleted if the company is deleted. |
| `deal_id` | `UUID` | `NULLABLE, FK → deals(id) ON DELETE CASCADE` | `NULL` | Associated deal. Activity is deleted if the deal is deleted. |
| `owner_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user responsible for this activity. |
| `due_date` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | When this activity is due. Relevant for tasks and deadlines. |
| `completed_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | When this activity was marked as completed. Set automatically when `is_completed` changes to true. |
| `is_completed` | `BOOLEAN` | `NOT NULL` | `false` | Whether this activity has been completed. When set to true, `completed_at` is set to NOW() if not already set. When set back to false, `completed_at` is cleared to NULL. |
| `duration_minutes` | `INTEGER` | `NULLABLE` | `NULL` | Duration of the activity in minutes. Must be > 0 if provided (application-enforced). Relevant for calls, meetings, lunches. |
| `outcome` | `TEXT` | `NULLABLE` | `NULL` | Outcome or result of the activity (e.g., "Left voicemail", "Scheduled follow-up"). |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this activity. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the activity was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the activity was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `activities_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_activities_type` | `type` | BTREE | For filtering by activity type. |
| `idx_activities_contact_id` | `contact_id` | BTREE | For finding all activities for a contact. |
| `idx_activities_company_id` | `company_id` | BTREE | For finding all activities for a company. |
| `idx_activities_deal_id` | `deal_id` | BTREE | For finding all activities for a deal. |
| `idx_activities_owner_id` | `owner_id` | BTREE | For finding all activities assigned to a user. |
| `idx_activities_due_date` | `due_date` | BTREE | For querying upcoming/overdue activities. |
| `idx_activities_is_completed` | `is_completed` | BTREE | For filtering completed vs. incomplete activities. |
| `idx_activities_created_at` | `created_at` | BTREE | For sorting by creation date. |

**Constraints:**

- `fk_activities_contact_id`: `FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE`
- `fk_activities_company_id`: `FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE`
- `fk_activities_deal_id`: `FOREIGN KEY (deal_id) REFERENCES deals(id) ON DELETE CASCADE`
- `fk_activities_owner_id`: `FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE SET NULL`
- `fk_activities_created_by`: `FOREIGN KEY (created_by) REFERENCES users(id)`

**SQL:**

```sql
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(30) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    description TEXT,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    deal_id UUID REFERENCES deals(id) ON DELETE CASCADE,
    owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
    due_date TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    is_completed BOOLEAN NOT NULL DEFAULT false,
    duration_minutes INTEGER,
    outcome TEXT,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activities_type ON activities (type);
CREATE INDEX idx_activities_contact_id ON activities (contact_id);
CREATE INDEX idx_activities_company_id ON activities (company_id);
CREATE INDEX idx_activities_deal_id ON activities (deal_id);
CREATE INDEX idx_activities_owner_id ON activities (owner_id);
CREATE INDEX idx_activities_due_date ON activities (due_date);
CREATE INDEX idx_activities_is_completed ON activities (is_completed);
CREATE INDEX idx_activities_created_at ON activities (created_at);
```

---

### 5.2.8 `activity_types`

Configurable activity type definitions. System types (where `is_system = true`) cannot be deleted by users. The `type` column on the `activities` table is validated at application level against the `name` values in this table.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique activity type identifier |
| `name` | `VARCHAR(50)` | `NOT NULL, UNIQUE` | — | Activity type name. Must be unique. Used as the value stored in `activities.type`. |
| `label` | `VARCHAR(100)` | `NOT NULL` | — | Human-readable display name for UI (e.g., "Phone Call", "Email"). |
| `icon` | `VARCHAR(50)` | `NOT NULL` | `'activity'` | Lucide icon name for UI display (e.g., "phone", "mail", "calendar"). |
| `color` | `VARCHAR(7)` | `NOT NULL` | `'#6B7280'` | Hex color code for the activity type. Must be a valid 7-character hex color. |
| `is_system` | `BOOLEAN` | `NOT NULL` | `false` | Whether this is a system activity type. System types cannot be deleted by users. |
| `display_order` | `INTEGER` | `NOT NULL` | — | Order in which types appear in the UI. Lower numbers appear first. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the activity type was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the activity type was last updated. Auto-set via trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `activity_types_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_activity_types_name` | `name` | UNIQUE | Automatic from UNIQUE constraint |
| `idx_activity_types_display_order` | `display_order` | BTREE | For ordering types in UI. |

**Seed Data:**

| name | label | icon | color | is_system | display_order |
|---|---|---|---|---|---|
| `call` | `Phone Call` | `phone` | `#3B82F6` | true | 1 |
| `email` | `Email` | `mail` | `#10B981` | true | 2 |
| `meeting` | `Meeting` | `calendar` | `#8B5CF6` | true | 3 |
| `note` | `Note` | `sticky-note` | `#F59E0B` | true | 4 |
| `task` | `Task` | `check-square` | `#EF4444` | true | 5 |
| `lunch` | `Lunch` | `utensils` | `#EC4899` | true | 6 |
| `deadline` | `Deadline` | `clock` | `#6B7280` | true | 7 |

**SQL:**

```sql
CREATE TABLE activity_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE,
    label VARCHAR(100) NOT NULL,
    icon VARCHAR(50) NOT NULL DEFAULT 'activity',
    color VARCHAR(7) NOT NULL DEFAULT '#6B7280',
    is_system BOOLEAN NOT NULL DEFAULT false,
    display_order INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_activity_types_display_order ON activity_types (display_order);
```

---

### 5.2.9 `meetings`

Stores scheduled meeting records with time slots, location, and status tracking. Distinct from the `activities` table — meetings have structured start/end times and attendee tracking.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique meeting identifier |
| `title` | `VARCHAR(255)` | `NOT NULL` | — | Meeting title. Must be 1-255 characters. |
| `description` | `TEXT` | `NULLABLE` | `NULL` | Meeting agenda or description. |
| `location` | `VARCHAR(255)` | `NULLABLE` | `NULL` | Physical meeting location. |
| `meeting_url` | `VARCHAR(500)` | `NULLABLE` | `NULL` | Video/virtual meeting URL (e.g., Zoom, Google Meet link). Must be a valid URL if provided. |
| `start_time` | `TIMESTAMPTZ` | `NOT NULL` | — | Meeting start time. Must be before `end_time`. |
| `end_time` | `TIMESTAMPTZ` | `NOT NULL` | — | Meeting end time. Must be after `start_time`. Application MUST validate `end_time > start_time`. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE SET NULL` | `NULL` | Primary contact for this meeting. |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE SET NULL` | `NULL` | Associated company. |
| `deal_id` | `UUID` | `NULLABLE, FK → deals(id) ON DELETE SET NULL` | `NULL` | Associated deal. |
| `owner_id` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who organized this meeting. Required — every meeting must have an organizer. |
| `outcome` | `TEXT` | `NULLABLE` | `NULL` | Notes on meeting outcome. Typically filled after the meeting. |
| `status` | `VARCHAR(20)` | `NOT NULL, CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'))` | `'scheduled'` | Meeting status. |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this meeting record. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the meeting was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the meeting was last modified. Auto-updated by trigger. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `meetings_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_meetings_start_time` | `start_time` | BTREE | For date-range queries. |
| `idx_meetings_contact_id` | `contact_id` | BTREE | For finding meetings with a contact. |
| `idx_meetings_company_id` | `company_id` | BTREE | For finding meetings with a company. |
| `idx_meetings_deal_id` | `deal_id` | BTREE | For finding meetings related to a deal. |
| `idx_meetings_owner_id` | `owner_id` | BTREE | For finding meetings organized by a user. |
| `idx_meetings_status` | `status` | BTREE | For filtering by meeting status. |

**Constraints:**

- `chk_meetings_status`: `CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'))`
- `fk_meetings_contact_id`: `FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE SET NULL`
- `fk_meetings_company_id`: `FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE SET NULL`
- `fk_meetings_deal_id`: `FOREIGN KEY (deal_id) REFERENCES deals(id) ON DELETE SET NULL`
- `fk_meetings_owner_id`: `FOREIGN KEY (owner_id) REFERENCES users(id)`
- `fk_meetings_created_by`: `FOREIGN KEY (created_by) REFERENCES users(id)`

**SQL:**

```sql
CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    location VARCHAR(255),
    meeting_url VARCHAR(500),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,
    company_id UUID REFERENCES companies(id) ON DELETE SET NULL,
    deal_id UUID REFERENCES deals(id) ON DELETE SET NULL,
    owner_id UUID NOT NULL REFERENCES users(id),
    outcome TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled'
        CHECK (status IN ('scheduled', 'confirmed', 'completed', 'cancelled', 'no_show')),
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_meetings_start_time ON meetings (start_time);
CREATE INDEX idx_meetings_contact_id ON meetings (contact_id);
CREATE INDEX idx_meetings_company_id ON meetings (company_id);
CREATE INDEX idx_meetings_deal_id ON meetings (deal_id);
CREATE INDEX idx_meetings_owner_id ON meetings (owner_id);
CREATE INDEX idx_meetings_status ON meetings (status);
```

---

### 5.2.10 `meeting_attendees`

Join table linking meetings to their attendees. Attendees can be either contacts (external) or users (internal).

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique attendee record identifier |
| `meeting_id` | `UUID` | `NOT NULL, FK → meetings(id) ON DELETE CASCADE` | — | The meeting this attendee belongs to. Cascade delete — removing a meeting removes all attendees. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE CASCADE` | `NULL` | The contact attending. Exactly one of `contact_id` or `user_id` must be set (application-enforced). |
| `user_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE CASCADE` | `NULL` | The internal user attending. Exactly one of `contact_id` or `user_id` must be set (application-enforced). |
| `status` | `VARCHAR(20)` | `NOT NULL, CHECK (status IN ('pending', 'accepted', 'declined', 'tentative'))` | `'pending'` | RSVP status for this attendee. |

**Unique Constraints:**

- `uq_meeting_attendees_contact`: `UNIQUE (meeting_id, contact_id)` — A contact can only appear once per meeting. This constraint only applies when `contact_id` is NOT NULL (PostgreSQL UNIQUE constraints ignore NULLs).
- `uq_meeting_attendees_user`: `UNIQUE (meeting_id, user_id)` — A user can only appear once per meeting. Same NULL behavior.

**Application-Level Validation:**

- Exactly one of `contact_id` or `user_id` MUST be provided. Both cannot be NULL. Both cannot be set simultaneously.
- When inserting, if a duplicate would occur (same meeting + same contact, or same meeting + same user), return a 409 Conflict error.

**Constraints:**

- `chk_meeting_attendees_status`: `CHECK (status IN ('pending', 'accepted', 'declined', 'tentative'))`
- `fk_meeting_attendees_meeting_id`: `FOREIGN KEY (meeting_id) REFERENCES meetings(id) ON DELETE CASCADE`
- `fk_meeting_attendees_contact_id`: `FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE`
- `fk_meeting_attendees_user_id`: `FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE`

**SQL:**

```sql
CREATE TABLE meeting_attendees (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'accepted', 'declined', 'tentative')),
    UNIQUE (meeting_id, contact_id),
    UNIQUE (meeting_id, user_id)
);
```

---

### 5.2.11 `notes`

Stores rich-text notes associated with contacts, companies, or deals. Notes can be pinned for visibility.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique note identifier |
| `content` | `TEXT` | `NOT NULL` | — | Note content. Must be at least 1 character. May contain plain text or HTML (sanitized at application level). |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE CASCADE` | `NULL` | Associated contact. Note is deleted if the contact is deleted. |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE CASCADE` | `NULL` | Associated company. Note is deleted if the company is deleted. |
| `deal_id` | `UUID` | `NULLABLE, FK → deals(id) ON DELETE CASCADE` | `NULL` | Associated deal. Note is deleted if the deal is deleted. |
| `is_pinned` | `BOOLEAN` | `NOT NULL` | `false` | Whether this note is pinned to the top of the notes list for its parent entity. |
| `created_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who created this note. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the note was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the note was last modified. Auto-updated by trigger. |

**Application-Level Validation:**

- At least one of `contact_id`, `company_id`, or `deal_id` MUST be provided. A note must be associated with at least one entity.
- A note CAN be associated with multiple entities simultaneously (e.g., a note about a contact AND a deal).

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `notes_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_notes_contact_id` | `contact_id` | BTREE | For finding all notes for a contact. |
| `idx_notes_company_id` | `company_id` | BTREE | For finding all notes for a company. |
| `idx_notes_deal_id` | `deal_id` | BTREE | For finding all notes for a deal. |
| `idx_notes_created_by` | `created_by` | BTREE | For finding all notes by a user. |
| `idx_notes_created_at` | `created_at` | BTREE | For sorting notes chronologically. |

**SQL:**

```sql
CREATE TABLE notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    deal_id UUID REFERENCES deals(id) ON DELETE CASCADE,
    is_pinned BOOLEAN NOT NULL DEFAULT false,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notes_contact_id ON notes (contact_id);
CREATE INDEX idx_notes_company_id ON notes (company_id);
CREATE INDEX idx_notes_deal_id ON notes (deal_id);
CREATE INDEX idx_notes_created_by ON notes (created_by);
CREATE INDEX idx_notes_created_at ON notes (created_at);
```

---

### 5.2.12 `attachments`

Stores metadata for files uploaded to Cloudflare R2 object storage. The actual file content is in R2; this table stores the reference and metadata.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique attachment identifier |
| `filename` | `VARCHAR(255)` | `NOT NULL` | — | Server-generated unique filename (e.g., UUID-based). Used as part of the R2 key. |
| `original_filename` | `VARCHAR(255)` | `NOT NULL` | — | The original filename as uploaded by the user. Preserved for display and download purposes. Must be sanitized of path traversal characters. |
| `mime_type` | `VARCHAR(100)` | `NOT NULL` | — | MIME type of the file (e.g., "application/pdf", "image/png"). Validated against an allowlist at upload time. |
| `size_bytes` | `BIGINT` | `NOT NULL` | — | File size in bytes. Must be > 0. Maximum file size enforced at application level (default: 25MB). |
| `r2_key` | `VARCHAR(500)` | `NOT NULL, UNIQUE` | — | The full object key in R2 storage. Format: `{entity_type}/{entity_id}/{filename}`. |
| `r2_bucket` | `VARCHAR(100)` | `NOT NULL` | — | The R2 bucket name where the file is stored. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE CASCADE` | `NULL` | Associated contact. Attachment is deleted (both DB record and R2 object) if the contact is deleted. |
| `company_id` | `UUID` | `NULLABLE, FK → companies(id) ON DELETE CASCADE` | `NULL` | Associated company. Same cascade behavior. |
| `deal_id` | `UUID` | `NULLABLE, FK → deals(id) ON DELETE CASCADE` | `NULL` | Associated deal. Same cascade behavior. |
| `uploaded_by` | `UUID` | `NOT NULL, FK → users(id)` | — | The user who uploaded this file. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the attachment was uploaded. |

**Application-Level Validation:**

- At least one of `contact_id`, `company_id`, or `deal_id` MUST be provided.
- On CASCADE delete of the parent entity, the application MUST also delete the corresponding R2 object. This requires a database trigger or application-level hook that fires on row deletion and issues an R2 delete request. If the R2 delete fails, log the error but do not block the database deletion — implement a cleanup job to handle orphaned R2 objects.
- **Allowed MIME types** (configurable via settings, default list): `application/pdf`, `image/png`, `image/jpeg`, `image/gif`, `image/webp`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `text/plain`, `text/csv`.
- **SVG files (`image/svg+xml`) are NOT allowed.** SVGs can contain embedded scripts and are a vector for XSS attacks. They MUST be rejected at upload time.

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `attachments_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_attachments_r2_key` | `r2_key` | UNIQUE | Automatic from UNIQUE constraint |
| `idx_attachments_contact_id` | `contact_id` | BTREE | For finding all attachments for a contact. |
| `idx_attachments_company_id` | `company_id` | BTREE | For finding all attachments for a company. |
| `idx_attachments_deal_id` | `deal_id` | BTREE | For finding all attachments for a deal. |
| `idx_attachments_uploaded_by` | `uploaded_by` | BTREE | For finding all attachments by a user. |
| `idx_attachments_created_at` | `created_at` | BTREE | For sorting by upload date. |

**SQL:**

```sql
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    r2_key VARCHAR(500) NOT NULL UNIQUE,
    r2_bucket VARCHAR(100) NOT NULL,
    contact_id UUID REFERENCES contacts(id) ON DELETE CASCADE,
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    deal_id UUID REFERENCES deals(id) ON DELETE CASCADE,
    uploaded_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_attachments_contact_id ON attachments (contact_id);
CREATE INDEX idx_attachments_company_id ON attachments (company_id);
CREATE INDEX idx_attachments_deal_id ON attachments (deal_id);
CREATE INDEX idx_attachments_uploaded_by ON attachments (uploaded_by);
CREATE INDEX idx_attachments_created_at ON attachments (created_at);
```

---

### 5.2.13 `api_keys`

Stores hashed API keys for programmatic access. The actual key is shown once at creation and never stored — only the HMAC hash is persisted.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique API key record identifier |
| `user_id` | `UUID` | `NOT NULL, FK → users(id) ON DELETE CASCADE` | — | The user this API key belongs to. Key is deleted if user is deleted. |
| `name` | `VARCHAR(100)` | `NOT NULL` | — | Human-readable name for the key (e.g., "CI/CD Integration"). Must be 1-100 characters. |
| `key_hash` | `VARCHAR(255)` | `NOT NULL` | — | HMAC-SHA256 hash of the API key, using a server-side secret. Used for lookup during authentication. |
| `key_prefix` | `VARCHAR(10)` | `NOT NULL` | — | First 8 characters of the API key. Displayed in the UI for identification (e.g., "crm_a1b2..."). |
| `permissions` | `JSONB` | `NOT NULL` | `'["read"]'::jsonb` | Array of permission strings. Valid values: `"read"`, `"write"`, `"delete"`, `"admin"`. Application validates on creation and update. |
| `last_used_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | When this key was last used for an API request. Updated on each use. |
| `expires_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | When this key expires. NULL means no expiration. Application checks this on each request. |
| `is_active` | `BOOLEAN` | `NOT NULL` | `true` | Whether this key is active. Inactive keys are rejected during authentication. |
| `revoked_at` | `TIMESTAMPTZ` | `NULLABLE` | `NULL` | When the key was revoked. Set to `NOW()` when key is revoked (`is_active` set to false). NULL if not revoked. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the key was created. |

**API Key Generation Process:**

1. Generate 32 random bytes using `crypto.randomBytes(32)`.
2. Encode as hex string, prefix with `crm_` — this is the raw API key (e.g., `crm_a1b2c3d4...`).
3. Hash with HMAC-SHA256 using the server-side secret (`API_KEY_SECRET` env var) — store as `key_hash`.
4. Take first 8 characters of the raw key (after `crm_` prefix) — store as `key_prefix`.
5. Return the raw key to the user ONCE. It is never stored or retrievable again.

> **Security Note:** Keys are hashed with HMAC-SHA256 using a server-side secret (`API_KEY_SECRET` env var). This prevents brute-force attacks if the database is compromised, because the attacker would also need the server secret to compute valid hashes.

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `api_keys_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_api_keys_key_hash` | `key_hash` | BTREE | For lookup during API authentication. |
| `idx_api_keys_user_id` | `user_id` | BTREE | For finding all keys belonging to a user. |
| `idx_api_keys_is_active` | `is_active` | BTREE | For filtering active/inactive keys. |

**SQL:**

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL,
    key_prefix VARCHAR(10) NOT NULL,
    permissions JSONB NOT NULL DEFAULT '["read"]'::jsonb,
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT true,
    revoked_at TIMESTAMPTZ DEFAULT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_keys_key_hash ON api_keys (key_hash);
CREATE INDEX idx_api_keys_user_id ON api_keys (user_id);
CREATE INDEX idx_api_keys_is_active ON api_keys (is_active);
```

---

### 5.2.14 `audit_log`

Immutable append-only log of all significant actions in the system. Rows are NEVER updated or deleted (except by a data retention policy if implemented).

> **IMPORTANT:** This table is INSERT-only. Create a database trigger or use application-level enforcement to prevent UPDATE and DELETE operations. See Section 5.3.2.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique audit entry identifier |
| `user_id` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who performed the action. NULL for system-generated events. Set to NULL if user is hard-deleted (but users are soft-deleted, so this is a safety measure). |
| `action` | `VARCHAR(50)` | `NOT NULL` | — | The action performed. Values: `'create'`, `'update'`, `'delete'`, `'login'`, `'logout'`, `'login_failed'`, `'export'`, `'import'`, `'password_change'`, `'password_reset'`, `'api_key_create'`, `'api_key_revoke'`, `'settings_update'`. |
| `entity_type` | `VARCHAR(50)` | `NOT NULL` | — | The type of entity affected. Values: `'contact'`, `'company'`, `'deal'`, `'activity'`, `'meeting'`, `'note'`, `'attachment'`, `'user'`, `'setting'`, `'api_key'`, `'email_template'`, `'session'`. |
| `entity_id` | `UUID` | `NULLABLE` | `NULL` | The ID of the affected entity. NULL for actions that don't target a specific entity (e.g., login). |
| `changes` | `JSONB` | `NULLABLE` | `NULL` | JSON object describing what changed. Format: `{ "before": { ... }, "after": { ... } }`. For create actions, only `after` is populated. For delete actions, only `before` is populated. For login/logout, this is NULL. Sensitive fields (`password_hash`) MUST NEVER appear in changes. |
| `ip_address` | `VARCHAR(45)` | `NULLABLE` | `NULL` | IP address of the request. Supports both IPv4 and IPv6. Obtained from `req.ip` or `x-forwarded-for` header behind a proxy. |
| `user_agent` | `TEXT` | `NULLABLE` | `NULL` | User-Agent header from the request. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When this audit entry was recorded. Immutable. |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `audit_log_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_audit_log_user_id` | `user_id` | BTREE | For finding all actions by a user. |
| `idx_audit_log_entity_type` | `entity_type` | BTREE | For filtering by entity type. |
| `idx_audit_log_entity_id` | `entity_id` | BTREE | For finding all audit entries for a specific entity. |
| `idx_audit_log_action` | `action` | BTREE | For filtering by action type. |
| `idx_audit_log_created_at` | `created_at` | BTREE | For time-range queries. This is the primary sorting column. |

**SQL:**

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID,
    changes JSONB,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_user_id ON audit_log (user_id);
CREATE INDEX idx_audit_log_entity_type ON audit_log (entity_type);
CREATE INDEX idx_audit_log_entity_id ON audit_log (entity_id);
CREATE INDEX idx_audit_log_action ON audit_log (action);
CREATE INDEX idx_audit_log_created_at ON audit_log (created_at);
```

---

### 5.2.15 `settings`

Key-value store for system-wide configuration. Each key is unique and stores its value as JSONB for flexibility.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique setting identifier |
| `key` | `VARCHAR(100)` | `UNIQUE, NOT NULL` | — | Setting key identifier (e.g., `'crm_name'`, `'timezone'`). |
| `value` | `JSONB` | `NOT NULL` | — | Setting value. Structure varies by key. |
| `updated_by` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who last updated this setting. NULL for system-set defaults. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When this setting was last changed. Auto-updated by trigger. |

**Seed Data:**

| Key | Default Value | Description |
|---|---|---|
| `crm_name` | `"CRM"` | Display name for the CRM application |
| `company_name` | `""` | Organization name using the CRM |
| `logo_url` | `null` | URL to the organization's logo |
| `primary_color` | `"#3B82F6"` | Primary brand color for the UI |
| `timezone` | `"UTC"` | Default timezone for the application |
| `date_format` | `"YYYY-MM-DD"` | Default date format |
| `currency` | `"USD"` | Default currency code |

**SQL:**

```sql
CREATE TABLE settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(100) UNIQUE NOT NULL,
    value JSONB NOT NULL,
    updated_by UUID REFERENCES users(id) ON DELETE SET NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### 5.2.16 `email_templates`

Stores reusable email templates with variable substitution support. System templates (where `is_system = true`) cannot be deleted by users.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique template identifier |
| `key` | `VARCHAR(100)` | `UNIQUE, NOT NULL` | — | Programmatic lookup key (e.g., `'welcome'`, `'password_reset'`). Used by application code to find specific templates. |
| `name` | `VARCHAR(100)` | `NOT NULL` | — | Human-readable template name. Must be 1-100 characters. Used for identification in the UI. |
| `subject` | `VARCHAR(255)` | `NOT NULL` | — | Email subject line. May contain variables in `{{variable_name}}` format. |
| `body_html` | `TEXT` | `NOT NULL` | — | HTML email body. May contain variables in `{{variable_name}}` format. Sanitized for XSS at application level. |
| `body_text` | `TEXT` | `NULLABLE` | `NULL` | Plain text fallback email body. May contain variables. If NULL, plain text is auto-generated from HTML by stripping tags. |
| `variables` | `TEXT[]` | `NOT NULL` | `'{}'` | List of variable names used in this template (e.g., `'{first_name, company_name, reset_url}'`). Used for documentation and UI display; actual substitution is done by scanning the body. |
| `is_active` | `BOOLEAN` | `NOT NULL` | `true` | Whether this template is available for use. Inactive templates are hidden from selection but not deleted. |
| `is_system` | `BOOLEAN` | `NOT NULL` | `false` | Whether this is a system template. System templates cannot be deleted by users. |
| `created_by` | `UUID` | `NULLABLE, FK → users(id)` | `NULL` | The user who created this template. NULL for system-seeded templates. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the template was created. |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the template was last modified. Auto-updated by trigger. |

**Seed Data:**

| key | name | is_system |
|---|---|---|
| `welcome` | Welcome Email | true |
| `password_reset` | Password Reset | true |
| `meeting_invitation` | Meeting Invitation | true |
| `meeting_update` | Meeting Update | true |
| `meeting_cancellation` | Meeting Cancellation | true |

**SQL:**

```sql
CREATE TABLE email_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    body_html TEXT NOT NULL,
    body_text TEXT,
    variables TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    is_system BOOLEAN NOT NULL DEFAULT false,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

### 5.2.17 `email_log`

Stores a record of every email sent by the system. Used for auditing, debugging delivery issues, and tracking communication with contacts.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique email log entry identifier |
| `to_email` | `VARCHAR(255)` | `NOT NULL` | — | The recipient email address. |
| `template_key` | `VARCHAR(100)` | `NULLABLE` | `NULL` | The `key` of the email template used, if any. Not a FK — templates may be deleted while log entries persist. |
| `subject` | `VARCHAR(255)` | `NOT NULL` | — | The rendered email subject line (after variable substitution). |
| `status` | `VARCHAR(20)` | `NOT NULL, CHECK (status IN ('sent', 'failed', 'bounced'))` | — | Delivery status of the email. |
| `error_message` | `TEXT` | `NULLABLE` | `NULL` | Error details if status is `'failed'` or `'bounced'`. NULL for successful sends. |
| `contact_id` | `UUID` | `NULLABLE, FK → contacts(id) ON DELETE SET NULL` | `NULL` | The contact this email was sent to, if applicable. Set to NULL if the contact is deleted. |
| `sent_by` | `UUID` | `NULLABLE, FK → users(id) ON DELETE SET NULL` | `NULL` | The user who triggered the email, if applicable. NULL for system-generated emails. Set to NULL if the user is deleted. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the email was sent (or attempted). |

**Indexes:**

| Index Name | Column(s) | Type | Notes |
|---|---|---|---|
| `email_log_pkey` | `id` | PRIMARY KEY | Automatic |
| `idx_email_log_contact_id` | `contact_id` | BTREE | For finding all emails sent to a contact. |
| `idx_email_log_created_at` | `created_at` | BTREE | For time-range queries on email history. |
| `idx_email_log_status` | `status` | BTREE | For filtering by delivery status. |

**Constraints:**

- `chk_email_log_status`: `CHECK (status IN ('sent', 'failed', 'bounced'))`
- `fk_email_log_contact_id`: `FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE SET NULL`
- `fk_email_log_sent_by`: `FOREIGN KEY (sent_by) REFERENCES users(id) ON DELETE SET NULL`

**SQL:**

```sql
CREATE TABLE email_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    to_email VARCHAR(255) NOT NULL,
    template_key VARCHAR(100),
    subject VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('sent', 'failed', 'bounced')),
    error_message TEXT,
    contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,
    sent_by UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_log_contact_id ON email_log (contact_id);
CREATE INDEX idx_email_log_created_at ON email_log (created_at);
CREATE INDEX idx_email_log_status ON email_log (status);
```

---

### 5.2.18 `tags`

Canonical tag definitions with colors and entity type scoping. The `tags` TEXT[] columns on contacts, companies, and deals store tag names directly (denormalized for query performance). This table serves as the master list for autocomplete and color assignment.

| Column | Type | Constraints | Default | Description |
|---|---|---|---|---|
| `id` | `UUID` | `PRIMARY KEY` | `gen_random_uuid()` | Unique tag identifier |
| `name` | `VARCHAR(50)` | `UNIQUE, NOT NULL` | — | Tag name. Must be 1-50 characters. Stored lowercase, trimmed. |
| `color` | `VARCHAR(7)` | `NOT NULL` | `'#6B7280'` | Hex color code for the tag (e.g., "#FF5733"). Must be a valid 7-character hex color string starting with `#`. |
| `entity_type` | `VARCHAR(20)` | `NOT NULL, CHECK (entity_type IN ('contact', 'company', 'deal'))` | — | Which entity type this tag applies to. A tag name is unique globally, not per entity type. |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | `NOW()` | When the tag was created. |

**Application-Level Behavior:**

- When a tag is added to an entity's `tags` array, if a matching `tags` row does not exist, one is auto-created with the default color.
- When a tag is renamed in the `tags` table, the application MUST update all corresponding entries in the `tags` TEXT[] arrays on contacts, companies, and deals.
- When a tag is deleted from the `tags` table, the application MUST remove it from all `tags` TEXT[] arrays on the corresponding entity type.

**Constraints:**

- `chk_tags_entity_type`: `CHECK (entity_type IN ('contact', 'company', 'deal'))`

**SQL:**

```sql
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    color VARCHAR(7) NOT NULL DEFAULT '#6B7280',
    entity_type VARCHAR(20) NOT NULL CHECK (entity_type IN ('contact', 'company', 'deal')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 5.3 Database Triggers

### 5.3.1 `updated_at` Auto-Update Trigger

A reusable trigger function that sets `updated_at = NOW()` on every UPDATE. Applied to all tables with an `updated_at` column.

```sql
CREATE OR REPLACE FUNCTION trigger_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Apply to each table:**

```sql
CREATE TRIGGER set_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON contacts FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON companies FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON deals FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON deal_stages FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON activities FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON meetings FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON notes FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON email_templates FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON settings FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
```

### 5.3.2 `audit_log` Immutability Trigger

Prevents UPDATE and DELETE operations on the `audit_log` table to enforce its append-only nature.

```sql
CREATE OR REPLACE FUNCTION trigger_audit_log_immutable()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_log table is immutable. UPDATE and DELETE operations are not permitted.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_log_no_update BEFORE UPDATE ON audit_log FOR EACH ROW EXECUTE FUNCTION trigger_audit_log_immutable();
CREATE TRIGGER audit_log_no_delete BEFORE DELETE ON audit_log FOR EACH ROW EXECUTE FUNCTION trigger_audit_log_immutable();
```

---

## 5.4 Migration Strategy

### 5.4.1 Migration Tool

**Knex.js** is the migration tool. All migrations use TypeScript and follow Knex conventions:

- **Timestamped migration files**: Format `YYYYMMDDHHMMSS_description.ts` (e.g., `20260101000001_create_sessions_table.ts`).
- **Up and down functions**: Every migration MUST export both `up` and `down` functions for full reversibility.
- **Atomic execution**: Each migration runs within a single database transaction. If any statement fails, the entire migration is rolled back.
- **Migration tracking**: A `knex_migrations` table (auto-created by Knex) records which migrations have been applied.

**Example Knex migration structure:**

```typescript
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
    await knex.schema.createTable('example', (table) => {
        table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
        table.string('name', 100).notNullable();
        table.timestamp('created_at', { useTz: true }).notNullable().defaultTo(knex.fn.now());
        table.timestamp('updated_at', { useTz: true }).notNullable().defaultTo(knex.fn.now());
    });
}

export async function down(knex: Knex): Promise<void> {
    await knex.schema.dropTableIfExists('example');
}
```

### 5.4.2 Migration Order

Migrations MUST be created and run in the following order (respecting foreign key dependencies):

| # | Migration File | Description |
|---|---|---|
| 1 | `20260101000001_create_sessions_table.ts` | Create `sessions` table (connect-pg-simple) |
| 2 | `20260101000002_create_users_table.ts` | Create `users` table |
| 3 | `20260101000003_create_updated_at_trigger.ts` | Create `trigger_set_updated_at` function |
| 4 | `20260101000004_create_audit_log_table.ts` | Create `audit_log` table + immutability trigger |
| 5 | `20260101000005_create_settings_table.ts` | Create `settings` table |
| 6 | `20260101000006_create_deal_stages_table.ts` | Create `deal_stages` table |
| 7 | `20260101000007_create_activity_types_table.ts` | Create `activity_types` table |
| 8 | `20260101000008_create_companies_table.ts` | Create `companies` table (references `users`) |
| 9 | `20260101000009_create_contacts_table.ts` | Create `contacts` table (references `users`, `companies`) |
| 10 | `20260101000010_create_deals_table.ts` | Create `deals` table (references `users`, `companies`, `contacts`) |
| 11 | `20260101000011_create_activities_table.ts` | Create `activities` table (references `users`, `contacts`, `companies`, `deals`) |
| 12 | `20260101000012_create_meetings_table.ts` | Create `meetings` table (references `users`, `contacts`, `companies`, `deals`) |
| 13 | `20260101000013_create_meeting_attendees_table.ts` | Create `meeting_attendees` table (references `meetings`, `contacts`, `users`) |
| 14 | `20260101000014_create_notes_table.ts` | Create `notes` table (references `users`, `contacts`, `companies`, `deals`) |
| 15 | `20260101000015_create_attachments_table.ts` | Create `attachments` table (references `users`, `contacts`, `companies`, `deals`) |
| 16 | `20260101000016_create_api_keys_table.ts` | Create `api_keys` table (references `users`) |
| 17 | `20260101000017_create_email_templates_table.ts` | Create `email_templates` table (references `users`) |
| 18 | `20260101000018_create_email_log_table.ts` | Create `email_log` table (references `contacts`, `users`) |
| 19 | `20260101000019_create_tags_table.ts` | Create `tags` table |
| 20 | `20260101000020_seed_defaults.ts` | Seed: settings, deal stages, activity types, email templates, admin user |

### 5.4.3 Seed Data

The seed migration (`20260101000020_seed_defaults.ts`) MUST insert:

1. **Default settings** (all entries from the settings seed data described in Section 5.2.15).

2. **Default deal stages** (all entries from the deal_stages seed data described in Section 5.2.6).

3. **Default activity types** (all entries from the activity_types seed data described in Section 5.2.8).

4. **Default email templates** (all entries from the email_templates seed data described in Section 5.2.16). Templates should include placeholder subject/body content with appropriate `{{variable}}` markers.

5. **Default admin user:**
   - email: `admin@example.com`
   - password: `Admin12345!` (bcrypt hashed, meets 10-character minimum)
   - first_name: `Admin`
   - last_name: `User`
   - role: `admin`
   - is_active: `true`

6. **Default tags:**
   - Contact tags: `hot_lead` (#EF4444), `vip` (#F59E0B), `do_not_contact` (#6B7280)
   - Company tags: `partner` (#3B82F6), `enterprise` (#8B5CF6), `startup` (#10B981)
   - Deal tags: `urgent` (#EF4444), `high_value` (#F59E0B), `renewal` (#3B82F6)

### 5.4.4 Down Migrations

Every migration's `down` function MUST:

- Drop the table it created (with `CASCADE` to handle dependent objects).
- Drop any indexes, triggers, or functions it created.
- Delete any seed data it inserted (for the seed migration).

### 5.4.5 Running Migrations

- **Development**: Migrations run automatically on server startup (configurable via `RUN_MIGRATIONS=true` environment variable).
- **Production**: Migrations are run as a separate step before deployment (e.g., as part of the CI/CD pipeline). The application server does NOT auto-run migrations in production.
- **Commands**:
  - `npx knex migrate:latest` — Apply all pending migrations
  - `npx knex migrate:rollback` — Revert the last batch of migrations
  - `npx knex migrate:rollback --all` — Revert all migrations

---

## 5.5 Entity Relationship Summary

```
users (1) ──────── (M) contacts             [owner_id]
users (1) ──────── (M) companies            [owner_id]
users (1) ──────── (M) deals                [owner_id]
users (1) ──────── (M) activities           [owner_id]
users (1) ──────── (M) meetings             [owner_id]
users (1) ──────── (M) api_keys             [user_id]
users (1) ──────── (M) audit_log            [user_id]
users (1) ──────── (M) notes                [created_by]
users (1) ──────── (M) attachments          [uploaded_by]
users (1) ──────── (M) email_templates      [created_by]
users (1) ──────── (M) email_log            [sent_by]
users (1) ──────── (M) meeting_attendees    [user_id]
users (1) ──────── (M) settings             [updated_by]

companies (1) ──── (M) contacts             [company_id]
companies (1) ──── (M) deals                [company_id]
companies (1) ──── (M) activities           [company_id]
companies (1) ──── (M) meetings             [company_id]
companies (1) ──── (M) notes                [company_id]
companies (1) ──── (M) attachments          [company_id]

contacts (1) ───── (M) deals                [contact_id]
contacts (1) ───── (M) activities           [contact_id]
contacts (1) ───── (M) meetings             [contact_id]
contacts (1) ───── (M) notes                [contact_id]
contacts (1) ───── (M) attachments          [contact_id]
contacts (1) ───── (M) meeting_attendees    [contact_id]
contacts (1) ───── (M) email_log            [contact_id]

deals (1) ──────── (M) activities           [deal_id]
deals (1) ──────── (M) meetings             [deal_id]
deals (1) ──────── (M) notes                [deal_id]
deals (1) ──────── (M) attachments          [deal_id]

meetings (1) ───── (M) meeting_attendees    [meeting_id]
```

**FK Behavior Summary:**

| Relationship | ON DELETE Behavior |
|---|---|
| `*.owner_id → users(id)` | SET NULL |
| `*.created_by → users(id)` | NO ACTION (users are soft-deactivated, never hard-deleted) |
| `api_keys.user_id → users(id)` | CASCADE |
| `meeting_attendees.user_id → users(id)` | CASCADE |
| `audit_log.user_id → users(id)` | SET NULL |
| `settings.updated_by → users(id)` | SET NULL |
| `email_log.sent_by → users(id)` | SET NULL |
| `email_log.contact_id → contacts(id)` | SET NULL |
| `contacts.company_id → companies(id)` | SET NULL |
| `deals.company_id → companies(id)` | SET NULL |
| `deals.contact_id → contacts(id)` | SET NULL |
| `activities.contact_id → contacts(id)` | CASCADE |
| `activities.company_id → companies(id)` | CASCADE |
| `activities.deal_id → deals(id)` | CASCADE |
| `meetings.contact_id → contacts(id)` | SET NULL |
| `meetings.company_id → companies(id)` | SET NULL |
| `meetings.deal_id → deals(id)` | SET NULL |
| `meeting_attendees.meeting_id → meetings(id)` | CASCADE |
| `meeting_attendees.contact_id → contacts(id)` | CASCADE |
| `notes.contact_id → contacts(id)` | CASCADE |
| `notes.company_id → companies(id)` | CASCADE |
| `notes.deal_id → deals(id)` | CASCADE |
| `attachments.contact_id → contacts(id)` | CASCADE |
| `attachments.company_id → companies(id)` | CASCADE |
| `attachments.deal_id → deals(id)` | CASCADE |
# SECTION 6a: Core Features — Authentication & User Management

This section defines the complete authentication, session management, and user management features. Every endpoint, validation rule, error case, and behavioral detail is specified for zero-ambiguity implementation. All endpoints use the `/api/v1/` prefix. All responses follow the canonical response format defined in Section 4.

---

## 6.1 Authentication System

### 6.1.1 Login

**Endpoint:** `POST /api/v1/auth/login`

**Request Body:**

```json
{
    "email": "string (required)",
    "password": "string (required)"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `email` | Required | `"Email is required"` |
| `email` | Must be a valid email format (RFC 5322 basic) | `"Invalid email format"` |
| `email` | Max 255 characters | `"Email must not exceed 255 characters"` |
| `password` | Required | `"Password is required"` |
| `password` | Min 10 characters | `"Password must be at least 10 characters"` |

**Process (Step-by-Step):**

1. **Validate input** — Run all validation rules. If any fail, return `400` with validation errors array.
2. **Normalize email** — Convert to lowercase, trim whitespace.
3. **Check rate limit** — Look up failed login attempts for this email in the last 15 minutes. If >= 5 attempts in the current 15-minute window, return `429`. If cumulative failed attempts >= 20 (lifetime), account is locked until admin unlock — return `403` with message `"Account is locked. Contact your administrator."`. If cumulative failed attempts >= 10 (lifetime), account is locked for 1 hour from last failed attempt — return `403` with message `"Account is temporarily locked. Try again later."`. Log all lockout events to `audit_log`.
4. **Find user by email** — Query `users` table for matching email. If not found, perform a dummy `bcrypt.compare` to normalize timing, then return `401` (generic error).
5. **Check account active** — If `is_active = false`, return `403` with message `"Account is deactivated"`. Do NOT increment rate limit counter (this is a known account, not a brute-force attempt).
6. **Compare password** — Use `bcrypt.compare(password, user.password_hash)`. If mismatch, increment rate limit counter (both sliding window and cumulative), return `401`.
7. **Regenerate session** — Call `req.session.regenerate()` to prevent session fixation. THEN store `userId`, `role`, and `createdAt` in the new session: `{ userId: user.id, role: user.role, createdAt: new Date().toISOString() }`.
8. **Update last_login_at** — Set `users.last_login_at = NOW()` for this user.
9. **Reset rate limit** — On successful login, reset the sliding-window counter for this email (cumulative counter is NOT reset).
10. **Log to audit_log** — Insert: `{ user_id, action: 'login', entity_type: 'session', entity_id: null, ip_address, user_agent }`.
11. **Return user data** — Return the user object in canonical format.

**Rate Limiting Implementation:**

- Use `rate-limiter-flexible` (or equivalent) with PostgreSQL backend, keyed by email address.
- **Sliding window (5 per 15 min):** On each failed login (step 6 failure), increment the counter for that email. Counter expires after 15 minutes from the first failed attempt. Maximum 5 failed attempts per email per 15-minute window. On successful login, reset this counter. Rate limit response includes `Retry-After` header with seconds remaining.
- **Cumulative lockout (10 total):** Track total failed login attempts per email (not reset on success). After 10 cumulative failed attempts, lock the account for 1 hour from the time of the 10th failure. Log lockout to `audit_log` with `action: 'account_locked_temporary'`.
- **Permanent lockout (20 total):** After 20 cumulative failed attempts, lock the account until an admin explicitly unlocks it (by setting a flag or resetting the counter). Log lockout to `audit_log` with `action: 'account_locked_permanent'`.

**Session Configuration:**

```javascript
{
    store: new PGSimpleStore({
        pool: pgPool,
        tableName: 'sessions',
        createTableIfMissing: false,
        pruneSessionInterval: 60 * 15,
    }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    rolling: true,
    cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 24 * 60 * 60 * 1000, // 24 hours
        path: '/',
    },
    name: 'crm.sid',
}
```

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "role": "member",
            "avatarUrl": null
        }
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [{ "field": "email", "message": "Email is required" }] }` |
| `401` | Email not found OR password mismatch | `{ "success": false, "error": "Invalid email or password" }` |
| `403` | Account deactivated | `{ "success": false, "error": "Account is deactivated" }` |
| `403` | Account temporarily locked (10+ cumulative failures) | `{ "success": false, "error": "Account is temporarily locked. Try again later." }` |
| `403` | Account permanently locked (20+ cumulative failures) | `{ "success": false, "error": "Account is locked. Contact your administrator." }` |
| `429` | Rate limit exceeded (5 per 15 min) | `{ "success": false, "error": "Too many login attempts. Try again in X minutes." }` |

**Security Notes:**

- The `401` response MUST be identical whether the email doesn't exist or the password is wrong. This prevents email enumeration.
- Response time should be consistent regardless of whether the email exists (use a dummy bcrypt compare when email not found to normalize timing).
- Password is NEVER logged or included in audit_log changes.

---

### 6.1.2 Logout

**Endpoint:** `POST /api/v1/auth/logout`

**Authentication:** Required (session must exist).

**Request Body:** None.

**Process:**

1. **Get session data** — Read `req.session.userId` before destroying.
2. **Destroy session** — Call `req.session.destroy()` to remove the session from PostgreSQL.
3. **Clear cookie** — Call `res.clearCookie('crm.sid')` with the same options used to set it (path, httpOnly, secure, sameSite).
4. **Log to audit_log** — Insert: `{ user_id, action: 'logout', entity_type: 'session', entity_id: null, ip_address, user_agent }`.
5. **Return success**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "message": "Logged out successfully"
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `401` | No active session | `{ "success": false, "error": "Not authenticated" }` |

**Edge Cases:**

- If the session has already expired (e.g., server restarted, session pruned), the middleware will catch this and return `401` before reaching the logout handler.
- If `req.session.destroy()` fails (DB error), log the error, still clear the cookie, and return `200` — the session will be pruned on expiry anyway.

---

### 6.1.3 Get Current User

**Endpoint:** `GET /api/v1/auth/me`

**Authentication:** Required.

**Request Body:** None.

**Process:**

1. **Read session** — Get `userId` from `req.session`.
2. **Fetch user** — Query `users` table by `id`. Select all fields except `password_hash`, `password_reset_token`, `password_reset_expires`.
3. **Check user exists and is active** — If user no longer exists or `is_active = false`, destroy session, return `401`.
4. **Return user data**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "role": "member",
            "avatarUrl": null,
            "isActive": true,
            "lastLoginAt": "2026-03-25T10:00:00Z",
            "createdAt": "2026-01-01T00:00:00Z",
            "updatedAt": "2026-03-20T15:30:00Z"
        }
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `401` | No session, session expired, user not found, or user deactivated | `{ "success": false, "error": "Not authenticated" }` |

**Frontend Usage:**

- Called on application load (page refresh, initial navigation) to restore auth state.
- If `401`, redirect to `/login` and clear any cached user data in the frontend store.
- The response is used to populate the frontend user context/store.

---

### 6.1.4 Change Password

**Endpoint:** `POST /api/v1/auth/change-password`

**Authentication:** Required.

**Request Body:**

```json
{
    "currentPassword": "string (required)",
    "newPassword": "string (required)",
    "confirmPassword": "string (required)"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `currentPassword` | Required | `"Current password is required"` |
| `newPassword` | Required | `"New password is required"` |
| `newPassword` | Min 10 characters | `"Password must be at least 10 characters"` |
| `newPassword` | Max 128 characters | `"Password must not exceed 128 characters"` |
| `newPassword` | Must contain at least one uppercase letter | `"Password must contain at least one uppercase letter"` |
| `newPassword` | Must contain at least one lowercase letter | `"Password must contain at least one lowercase letter"` |
| `newPassword` | Must contain at least one number | `"Password must contain at least one number"` |
| `newPassword` | Must contain at least one special character (`!@#$%^&*()_+-=[]{};\|:'",./<>?`) | `"Password must contain at least one special character"` |
| `newPassword` | Must not be the same as `currentPassword` | `"New password must be different from current password"` |
| `confirmPassword` | Required | `"Password confirmation is required"` |
| `confirmPassword` | Must match `newPassword` | `"Passwords do not match"` |

**Shared Zod Schema (used by both frontend and backend):**

```typescript
// File: packages/shared/src/schemas/password.ts
import { z } from 'zod';

export const passwordSchema = z
    .string()
    .min(10, 'Password must be at least 10 characters')
    .max(128, 'Password must not exceed 128 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .regex(
        /[!@#$%^&*()_+\-=\[\]{};'\\|:",.\/<>?]/,
        'Password must contain at least one special character'
    );

export const changePasswordSchema = z.object({
    currentPassword: z.string().min(1, 'Current password is required'),
    newPassword: passwordSchema,
    confirmPassword: z.string().min(1, 'Password confirmation is required'),
}).refine((data) => data.newPassword === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'],
}).refine((data) => data.newPassword !== data.currentPassword, {
    message: 'New password must be different from current password',
    path: ['newPassword'],
});
```

**Process:**

1. **Validate input** — Run all validation rules using the shared Zod schema.
2. **Fetch user** — Get full user record including `password_hash`.
3. **Verify current password** — `bcrypt.compare(currentPassword, user.password_hash)`. If mismatch, return `401`.
4. **Hash new password** — `bcrypt.hash(newPassword, 12)` (cost factor 12).
5. **Update password** — Set `password_hash` to new hash in `users` table.
6. **Invalidate ALL other sessions** — Delete all rows from `sessions` table where `sess->>'userId' = user.id` AND `sid != req.sessionID`. This logs out all other devices.
7. **Log to audit_log** — Insert: `{ user_id, action: 'password_change', entity_type: 'user', entity_id: user.id, changes: null, ip_address, user_agent }`. NEVER log password values.
8. **Return success**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "message": "Password changed successfully"
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [...] }` |
| `401` | Current password incorrect | `{ "success": false, "error": "Current password is incorrect" }` |

---

### 6.1.5 Forgot Password

**Endpoint:** `POST /api/v1/auth/forgot-password`

**Authentication:** None (public endpoint).

**Request Body:**

```json
{
    "email": "string (required)"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `email` | Required | `"Email is required"` |
| `email` | Valid email format | `"Invalid email format"` |

**Rate Limiting:** Max 3 forgot-password requests per email per hour. Max 10 forgot-password requests per IP per hour. If exceeded, still return the same success response (do not reveal rate limiting to potential attackers). Silently drop the request without generating a token or sending an email.

**Process:**

1. **Validate input**.
2. **Record start time** — `const startTime = Date.now()`.
3. **Normalize email** — Lowercase, trim.
4. **Check rate limits** — If either limit exceeded, skip to step 6 (still return same response).
5. **Find user by email** — Query `users` table.
6. **If user found AND is_active:**
   a. Generate token: `crypto.randomBytes(32).toString('hex')` — this is the raw (unhashed) token.
   b. Hash token: `crypto.createHash('sha256').update(rawToken).digest('hex')`.
   c. Set expiry: `new Date(Date.now() + 60 * 60 * 1000)` (1 hour from now).
   d. Update user: set `password_reset_token = hashedToken`, `password_reset_expires = expiry`.
   e. Send email via Brevo transactional API:
      - To: user's email
      - Subject: "Password Reset Request"
      - Body includes: reset link `{FRONTEND_URL}/reset-password?token={rawToken}&email={encodeURIComponent(email)}`
      - Body includes: "This link expires in 1 hour."
      - Body includes: "If you did not request this, please ignore this email."
   f. Log to audit_log: `{ user_id, action: 'password_reset_request', entity_type: 'user', entity_id: user.id }`.
7. **If user NOT found or is_active = false:** Do nothing. Do NOT send any email. Do NOT log.
8. **Timing normalization** — Calculate elapsed time since step 2. Always wait at least 500ms total, plus random jitter of 200-800ms additional. Implementation: `const elapsed = Date.now() - startTime; const minDelay = 500 + Math.floor(Math.random() * 601) + 200; const remaining = minDelay - elapsed; if (remaining > 0) await sleep(remaining);`
9. **Return identical response regardless of step 6 or 7 outcome.**

**Success Response (200) — ALWAYS returned:**

```json
{
    "success": true,
    "data": {
        "message": "If an account exists with that email, a password reset link has been sent."
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [...] }` |

**Security Notes:**

- The response is ALWAYS the same 200 whether the email exists or not. This prevents email enumeration.
- Timing normalization (minimum 500ms + 200-800ms random jitter) prevents timing-based enumeration. The jitter makes it impossible to distinguish "user found + email sent" from "user not found + no-op" by measuring response time.
- Previous reset tokens are overwritten — only the latest token is valid.
- The raw token is sent via email; only the SHA-256 hash is stored in the database.

---

### 6.1.6 Reset Password

**Endpoint:** `POST /api/v1/auth/reset-password`

**Authentication:** None (public endpoint).

**Request Body:**

```json
{
    "token": "string (required)",
    "email": "string (required)",
    "newPassword": "string (required)",
    "confirmPassword": "string (required)"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `token` | Required | `"Reset token is required"` |
| `email` | Required | `"Email is required"` |
| `email` | Valid email format | `"Invalid email format"` |
| `newPassword` | Required | `"New password is required"` |
| `newPassword` | Min 10 characters, max 128 characters, at least 1 uppercase, 1 lowercase, 1 number, 1 special char (uses shared `passwordSchema`) | (same messages as change-password) |
| `confirmPassword` | Required | `"Password confirmation is required"` |
| `confirmPassword` | Must match `newPassword` | `"Passwords do not match"` |

**Process:**

1. **Validate input** — Using the shared Zod password schema.
2. **Normalize email** — Lowercase, trim.
3. **Hash provided token** — `crypto.createHash('sha256').update(token).digest('hex')`.
4. **Find user** — Query `users` table: `WHERE email = $1 AND password_reset_token = $2 AND password_reset_expires > NOW()`.
5. **If no matching user:** Return `400` with generic error (do not reveal which check failed).
6. **Hash new password** — `bcrypt.hash(newPassword, 12)`.
7. **Update user** — Set `password_hash = newHash`, `password_reset_token = NULL`, `password_reset_expires = NULL`. The token is single-use: cleared immediately after use.
8. **Invalidate ALL sessions** — Delete all rows from `sessions` where `sess->>'userId' = user.id`.
9. **Log to audit_log** — `{ user_id, action: 'password_reset', entity_type: 'user', entity_id: user.id }`.
10. **Send confirmation email** via Brevo:
    - Subject: "Your password has been reset"
    - Body: "Your password was successfully changed. If you did not make this change, please contact support immediately."
11. **Return success**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "message": "Password has been reset successfully. Please log in with your new password."
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [...] }` |
| `400` | Token invalid, expired, or email mismatch | `{ "success": false, "error": "Invalid or expired reset token" }` |

**Edge Cases:**

- If the token has been used (cleared from DB), the query in step 4 finds no match — `400`.
- If the token exists but is expired (`password_reset_expires < NOW()`), the query in step 4 finds no match — `400`.
- If someone uses the correct token but wrong email, the query finds no match — `400`.
- After successful reset, the token is cleared — it cannot be used again (single-use).

---

### 6.1.7 Session Management

**Session Store Configuration:**

```javascript
{
    store: new PGSimpleStore({
        pool: pgPool,                    // PostgreSQL connection pool
        tableName: 'sessions',           // Table name (matches our schema)
        createTableIfMissing: false,     // We create it via migrations
        pruneSessionInterval: 60 * 15,  // Prune expired sessions every 15 minutes (in seconds)
    }),
    secret: process.env.SESSION_SECRET,  // MUST be a strong random string (min 32 chars)
    resave: false,                       // Don't resave unchanged sessions
    saveUninitialized: false,            // Don't save empty sessions
    rolling: true,                       // Reset maxAge on every request (extends on each request)
    cookie: {
        httpOnly: true,                  // Not accessible via JavaScript
        secure: process.env.NODE_ENV === 'production', // HTTPS only in production
        sameSite: 'lax',                 // CSRF protection
        maxAge: 24 * 60 * 60 * 1000,    // 24 hours in milliseconds
        path: '/',                       // Cookie valid for all paths
    },
    name: 'crm.sid',                    // Custom cookie name (not default 'connect.sid')
}
```

**Session Data Structure:**

The `sess` JSON column contains:

```json
{
    "cookie": {
        "originalMaxAge": 86400000,
        "expires": "2026-03-26T10:00:00.000Z",
        "httpOnly": true,
        "secure": true,
        "sameSite": "lax",
        "path": "/"
    },
    "userId": "uuid-string",
    "role": "member",
    "createdAt": "2026-03-25T10:00:00.000Z"
}
```

**Rolling Sessions Behavior:**

- `rolling: true` means that on EVERY authenticated request, the session's `expire` timestamp in the database is updated to `NOW() + maxAge` (24 hours).
- This means a session stays alive as long as the user is actively making requests within the 24-hour window.
- If a user is inactive for 24 hours, their session expires and they must log in again.
- This is a rolling expiry — NOT absolute. There is no maximum session lifetime beyond continuous inactivity.

**Session Pruning:**

- `connect-pg-simple` automatically deletes expired sessions from the `sessions` table every 15 minutes.
- The `idx_sessions_expire` index ensures this cleanup is efficient.
- No additional cron job or manual cleanup is needed.

**Per-Request User Validation:**

- On every authenticated request, the `isAuthenticated` middleware checks that the user still exists and `is_active = true` in the database. This ensures that deactivated users are immediately logged out, even if their session has not yet expired.

---

### 6.1.8 Authentication Middleware

#### `isAuthenticated`

Applied to all routes that require a logged-in user.

**Behavior:**

1. Check if `req.session` exists and `req.session.userId` is set.
2. If no session or no userId: return `401 { "success": false, "error": "Not authenticated" }`.
3. Query `users` table for `req.session.userId`. Select `id`, `is_active`, `role`.
4. If user not found: destroy session, return `401`.
5. If `is_active = false`: destroy session, return `401`.
6. Attach user data to `req.user = { id, role, isActive }` for downstream handlers.
7. Call `next()`.

**Performance Consideration:**

- The user lookup in step 3 happens on EVERY authenticated request. This is a simple primary key lookup (~1ms with connection pooling).
- The chosen approach: **query on every request** for maximum security (immediate deactivation takes effect).

#### `isAdmin`

Applied after `isAuthenticated` to routes that require admin privileges.

**Behavior:**

1. Check `req.user.role === 'admin'`.
2. If not admin: return `403 { "success": false, "error": "Admin access required" }`.
3. Call `next()`.

---

### 6.1.9 CSRF Protection

**Implementation Strategy: Synchronizer Token Pattern**

The server generates a CSRF token, stores it in the session, and the frontend sends it back on state-changing requests. This is the synchronizer token pattern (NOT double-submit cookie).

1. **On session creation (login):** Generate a CSRF token using `crypto.randomBytes(32).toString('hex')`. Store in the session: `req.session.csrfToken = token`.
2. **Expose token endpoint:** `GET /api/v1/auth/csrf-token` — Returns the CSRF token from the server-side session.

   **Success Response (200):**
   ```json
   { "success": true, "data": { "csrfToken": "abc123..." } }
   ```

3. **Frontend behavior:** On app initialization (after successful `/api/v1/auth/me`), call `GET /api/v1/auth/csrf-token`. Store the token in memory (NOT localStorage — XSS risk). Include the token in all state-changing requests via the `X-CSRF-Token` header.
4. **Middleware (`csrfProtection`):** Applied to all `POST`, `PUT`, `PATCH`, `DELETE` routes:
   a. Read `X-CSRF-Token` header from request. For multipart form data requests, also check for a `_csrf` form field as a fallback.
   b. Compare with `req.session.csrfToken` (server-side session lookup).
   c. If missing or mismatch: return `403 { "success": false, "error": "Invalid CSRF token" }`.
   d. If match: call `next()`.
5. **Exempt routes:** `POST /api/v1/auth/login`, `POST /api/v1/auth/forgot-password`, `POST /api/v1/auth/reset-password` — these are pre-authentication and don't have a session yet.
6. **API key authenticated requests** are exempt from CSRF (they use a different auth mechanism and are not vulnerable to CSRF).
7. **Multipart form data:** For file upload or multipart requests, the CSRF token can be included either as the `X-CSRF-Token` header (preferred) or as a `_csrf` form field within the multipart body.

---

## 6.2 User Management

All user management endpoints require admin authorization unless otherwise specified.

### 6.2.1 List Users

**Endpoint:** `GET /api/v1/users`

**Authentication:** Required. **Authorization:** Admin only.

**Query Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `1` | Page number (1-indexed). Must be >= 1. |
| `limit` | integer | `25` | Items per page. Min 1, max 100. |
| `sort` | string | `created_at` | Sort field. Allowed values: `created_at`, `email`, `first_name`, `last_name`, `role`, `last_login_at`. |
| `order` | string | `desc` | Sort direction. Allowed values: `asc`, `desc`. |
| `role` | string | (none) | Filter by role. Allowed values: `admin`, `member`. |
| `is_active` | boolean | (none) | Filter by active status. Values: `true`, `false`. |
| `search` | string | (none) | Search string. Searches across `first_name`, `last_name`, and `email` using case-insensitive `ILIKE '%search%'` on each field (OR'd together). Min 1 character if provided. |

**Query Construction:**

```sql
SELECT id, email, first_name, last_name, role, is_active, avatar_url, last_login_at, created_at, updated_at
FROM users
WHERE 1=1
  [AND role = $role]
  [AND is_active = $is_active]
  [AND (first_name ILIKE $search OR last_name ILIKE $search OR email ILIKE $search)]
ORDER BY $sort $order
LIMIT $limit OFFSET ($page - 1) * $limit
```

A separate `COUNT(*)` query with the same WHERE clause provides the total count for pagination metadata.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "users": [
            {
                "id": "uuid",
                "email": "user@example.com",
                "firstName": "John",
                "lastName": "Doe",
                "role": "member",
                "isActive": true,
                "avatarUrl": null,
                "lastLoginAt": "2026-03-25T10:00:00Z",
                "createdAt": "2026-01-01T00:00:00Z",
                "updatedAt": "2026-03-20T15:30:00Z"
            }
        ],
    },
    "meta": {
        "page": 1,
        "limit": 25,
        "total": 42,
        "totalPages": 2
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Invalid query parameters | `{ "success": false, "error": "Validation failed", "details": [...] }` |
| `401` | Not authenticated | `{ "success": false, "error": "Not authenticated" }` |
| `403` | Not admin | `{ "success": false, "error": "Admin access required" }` |

**Edge Cases:**

- If `page` exceeds total pages, return empty `users` array with correct pagination metadata (not a 404).
- If `search` is empty string, ignore the search filter.
- `password_hash`, `password_reset_token`, and `password_reset_expires` are NEVER included in the response.

---

### 6.2.2 Get User

**Endpoint:** `GET /api/v1/users/:id`

**Authentication:** Required. **Authorization:** Admin only.

**URL Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | UUID | User ID. Must be a valid UUID format. |

**Process:**

1. **Validate UUID format** — If `id` is not a valid UUID, return `400 { "success": false, "error": "Invalid user ID format" }`.
2. **Query user** — Select all fields except `password_hash`, `password_reset_token`, `password_reset_expires`.
3. **If not found** — Return `404 { "success": false, "error": "User not found" }`.
4. **Return user data**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "role": "member",
            "isActive": true,
            "avatarUrl": null,
            "lastLoginAt": "2026-03-25T10:00:00Z",
            "createdAt": "2026-01-01T00:00:00Z",
            "updatedAt": "2026-03-20T15:30:00Z"
        }
    }
}
```

---

### 6.2.3 Create User

**Endpoint:** `POST /api/v1/users`

**Authentication:** Required. **Authorization:** Admin only.

**Request Body:**

```json
{
    "email": "string (required)",
    "firstName": "string (required)",
    "lastName": "string (required)",
    "role": "string (required: 'admin' or 'member')",
    "password": "string (required)"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `email` | Required | `"Email is required"` |
| `email` | Valid email format | `"Invalid email format"` |
| `email` | Max 255 characters | `"Email must not exceed 255 characters"` |
| `email` | Must be unique (case-insensitive) | `"A user with this email already exists"` |
| `firstName` | Required | `"First name is required"` |
| `firstName` | 1-100 characters after trimming | `"First name must be between 1 and 100 characters"` |
| `lastName` | Required | `"Last name is required"` |
| `lastName` | 1-100 characters after trimming | `"Last name must be between 1 and 100 characters"` |
| `role` | Required | `"Role is required"` |
| `role` | Must be `'admin'` or `'member'` | `"Role must be 'admin' or 'member'"` |
| `password` | Required | `"Password is required"` |
| `password` | Min 10 chars, max 128 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char (uses shared `passwordSchema`) | (same messages as change-password) |

**Process:**

1. **Validate input** — Using the shared Zod password schema for the password field.
2. **Normalize email** — Lowercase, trim.
3. **Trim names** — Trim `firstName` and `lastName`.
4. **Check email uniqueness** — Query `users` where `email = normalizedEmail`. If exists, return `409`. The `users` table also has a DB-level unique constraint on `email` as a safety net.
5. **Hash password** — `bcrypt.hash(password, 12)` (cost factor 12).
6. **Insert user** — Insert into `users` table with all fields. `is_active` defaults to `true`.
7. **Send welcome email** via Brevo:
   - To: new user's email
   - Subject: "Welcome to CRM"
   - Body includes: "Your account has been created. You can log in at {FRONTEND_URL}/login with your email and the password provided by your administrator."
   - Do NOT include the password in the email.
   - **Graceful failure:** If Brevo is unavailable or the email send fails, log a warning (including the user ID and email address) but do NOT block user creation. The user is still created successfully. The admin should be informed that the welcome email was not sent (include a note in the response or log).
8. **Log to audit_log** — `{ user_id: req.user.id (admin), action: 'create', entity_type: 'user', entity_id: newUser.id, changes: { email: { new: email }, firstName: { new: firstName }, lastName: { new: lastName }, role: { new: role } } }`.
9. **Return created user** (without password_hash).

**Success Response (201):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "newuser@example.com",
            "firstName": "Jane",
            "lastName": "Smith",
            "role": "member",
            "isActive": true,
            "avatarUrl": null,
            "lastLoginAt": null,
            "createdAt": "2026-03-25T10:00:00Z",
            "updatedAt": "2026-03-25T10:00:00Z"
        }
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [...] }` |
| `409` | Email already exists | `{ "success": false, "error": "A user with this email already exists" }` |

---

### 6.2.4 Update User

**Endpoint:** `PUT /api/v1/users/:id`

**Authentication:** Required. **Authorization:** Admin only.

**URL Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | UUID | User ID to update. |

**Request Body (all fields optional):**

```json
{
    "firstName": "string",
    "lastName": "string",
    "role": "string ('admin' or 'member')",
    "isActive": "boolean"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `firstName` | If provided, 1-100 characters after trimming | `"First name must be between 1 and 100 characters"` |
| `lastName` | If provided, 1-100 characters after trimming | `"Last name must be between 1 and 100 characters"` |
| `role` | If provided, must be `'admin'` or `'member'` | `"Role must be 'admin' or 'member'"` |
| `isActive` | If provided, must be a boolean | `"isActive must be a boolean"` |

**Business Rules:**

| Rule | Error |
|---|---|
| Admin cannot change their own role | `403 "Cannot change your own role"` |
| Admin cannot deactivate themselves | `403 "Cannot deactivate your own account"` |
| At least one field must be provided | `400 "At least one field must be provided for update"` |

**Process:**

1. **Validate UUID format**.
2. **Validate input**.
3. **Fetch existing user** — If not found, return `404`.
4. **Check business rules** — Compare `req.user.id` with `:id` for self-modification checks.
5. **Build update** — Only update provided fields. Track changes (before/after values for each changed field).
6. **Update user** — Execute UPDATE query.
7. **If `isActive` changed to `false`:** Invalidate all sessions for this user — `DELETE FROM sessions WHERE sess->>'userId' = :id`.
8. **Log to audit_log** — Include before/after values for each changed field: `{ user_id: req.user.id, action: 'update', entity_type: 'user', entity_id: :id, changes: { role: { old: 'member', new: 'admin' } } }`.
9. **Return updated user**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "user@example.com",
            "firstName": "John",
            "lastName": "Doe",
            "role": "admin",
            "isActive": true,
            "avatarUrl": null,
            "lastLoginAt": "2026-03-25T10:00:00Z",
            "createdAt": "2026-01-01T00:00:00Z",
            "updatedAt": "2026-03-25T12:00:00Z"
        }
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Invalid input or no fields provided | `{ "success": false, "error": "..." }` |
| `403` | Self-modification violation | `{ "success": false, "error": "Cannot change your own role" }` or `"Cannot deactivate your own account"` |
| `404` | User not found | `{ "success": false, "error": "User not found" }` |

---

### 6.2.5 Delete User (Soft Delete)

**Endpoint:** `DELETE /api/v1/users/:id`

**Authentication:** Required. **Authorization:** Admin only.

**URL Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `id` | UUID | User ID to delete. |

**Business Rules:**

| Rule | Error |
|---|---|
| Admin cannot delete themselves | `403 "Cannot delete your own account"` |

**Process:**

1. **Validate UUID format**.
2. **Fetch existing user** — If not found, return `404`.
3. **Check business rules** — Cannot delete self.
4. **Soft delete** — Set `is_active = false` in `users` table. Do NOT hard-delete the row.
5. **Invalidate all sessions** — `DELETE FROM sessions WHERE sess->>'userId' = :id`.
6. **Handle owned entities** — Set `owner_id = NULL` on all contacts, companies, and deals owned by this user. This leaves them unassigned rather than transferring to the admin. The admin can reassign them manually via the UI.
7. **Log to audit_log** — `{ user_id: req.user.id, action: 'delete', entity_type: 'user', entity_id: :id, changes: { isActive: { old: true, new: false } } }`.
8. **Return success**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "message": "User has been deactivated"
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Invalid UUID format | `{ "success": false, "error": "Invalid user ID format" }` |
| `403` | Attempting to delete self | `{ "success": false, "error": "Cannot delete your own account" }` |
| `404` | User not found | `{ "success": false, "error": "User not found" }` |

**Edge Cases:**

- Deleting an already-deactivated user: This is idempotent — set `is_active = false` again, return `200`. Log the action.
- Deleting a user who owns entities: Those entities become unassigned (`owner_id = NULL`). The entities themselves are NOT deleted.

---

### 6.2.6 Update Own Profile

**Endpoint:** `PUT /api/v1/auth/profile`

**Authentication:** Required. **Authorization:** Any authenticated user.

**Request Body (all fields optional):**

```json
{
    "firstName": "string",
    "lastName": "string",
    "avatarUrl": "string or null"
}
```

**Validation Rules:**

| Field | Rule | Error Message |
|---|---|---|
| `firstName` | If provided, 1-100 characters after trimming | `"First name must be between 1 and 100 characters"` |
| `lastName` | If provided, 1-100 characters after trimming | `"Last name must be between 1 and 100 characters"` |
| `avatarUrl` | If provided and not null, must be a valid URL | `"Avatar URL must be a valid URL"` |
| `avatarUrl` | If provided and not null, max 2000 characters | `"Avatar URL must not exceed 2000 characters"` |

**Restrictions:**

- Users CANNOT change their own `email`, `role`, or `isActive` status via this endpoint.
- If any disallowed fields are included in the request body, they are silently ignored (not an error).

**Process:**

1. **Validate input**.
2. **Build update** — Only update provided fields. Track changes.
3. **Update user** — Execute UPDATE query for `req.user.id`.
4. **Log to audit_log** — `{ user_id: req.user.id, action: 'update', entity_type: 'user', entity_id: req.user.id, changes: { firstName: { old: 'John', new: 'Jonathan' } } }`.
5. **Return updated user**.

**Success Response (200):**

```json
{
    "success": true,
    "data": {
        "user": {
            "id": "uuid",
            "email": "user@example.com",
            "firstName": "Jonathan",
            "lastName": "Doe",
            "role": "member",
            "isActive": true,
            "avatarUrl": "https://example.com/avatar.jpg",
            "lastLoginAt": "2026-03-25T10:00:00Z",
            "createdAt": "2026-01-01T00:00:00Z",
            "updatedAt": "2026-03-25T12:00:00Z"
        }
    }
}
```

**Error Responses:**

| Status | Condition | Response Body |
|---|---|---|
| `400` | Validation failure | `{ "success": false, "error": "Validation failed", "details": [...] }` |

---

## 6.3 API Key Authentication

API keys provide an alternative authentication mechanism for programmatic access (integrations, scripts, CI/CD).

### 6.3.1 Authentication Flow

1. Client includes the API key in the `Authorization` header: `Authorization: Bearer crm_a1b2c3d4...`
2. **Middleware (`authenticateApiKey`):**
   a. Extract the key from the `Authorization` header.
   b. Verify it starts with `crm_` prefix.
   c. Hash the key: `crypto.createHmac('sha256', process.env.API_KEY_SECRET).update(key).digest('hex')`.
   d. Query `api_keys` table: `WHERE key_hash = $hash AND is_active = true`.
   e. If not found: return `401 { "success": false, "error": "Invalid API key" }`.
   f. If found, check `expires_at`: if set and in the past, return `401 { "success": false, "error": "API key has expired" }`.
   g. Fetch the owning user: `SELECT id, role, is_active FROM users WHERE id = api_key.user_id`.
   h. If user not found or `is_active = false`: return `401 { "success": false, "error": "Invalid API key" }`.
   i. Update `last_used_at = NOW()` on the API key (fire-and-forget, don't block the request).
   j. Set `req.user = { id: user.id, role: user.role, isActive: true }`.
   k. Set `req.apiKey = { id: apiKey.id, permissions: apiKey.permissions }`.
   l. Call `next()`.

3. **Permission checking middleware (`requirePermission(permission)`):**
   - If request is session-authenticated (no `req.apiKey`): all permissions are granted (session users have full access).
   - If request is API key-authenticated: check `req.apiKey.permissions.includes(permission)`.
   - If permission missing: return `403 { "success": false, "error": "API key does not have 'write' permission" }`.

4. **Combined authentication middleware (`authenticate`):**
   - Check for `Authorization: Bearer` header first (API key auth).
   - If no Bearer header, fall back to session auth (`req.session.userId`).
   - If neither: return `401`.

### 6.3.2 API Key Management Endpoints

#### List API Keys (GET /api/v1/api-keys)

**Authentication:** Required (session only, not API key). **Authorization:** Any authenticated user sees only their own keys. Admins can see all keys with `?all=true`.

**Response includes:** `id`, `name`, `keyPrefix`, `permissions`, `lastUsedAt`, `expiresAt`, `isActive`, `createdAt`. Never includes `key_hash`.

#### Create API Key (POST /api/v1/api-keys)

**Authentication:** Required (session only). **Authorization:** Any authenticated user.

**Request Body:**

```json
{
    "name": "string (required, 1-100 chars)",
    "permissions": ["read", "write"],
    "expiresAt": "2027-03-25T00:00:00Z (optional, must be in the future)"
}
```

**Response (201) — includes the raw key ONCE:**

```json
{
    "success": true,
    "data": {
        "apiKey": {
            "id": "uuid",
            "name": "CI/CD Integration",
            "keyPrefix": "crm_a1b2",
            "key": "crm_a1b2c3d4e5f6...",
            "permissions": ["read", "write"],
            "expiresAt": "2027-03-25T00:00:00Z",
            "isActive": true,
            "createdAt": "2026-03-25T10:00:00Z"
        }
    }
}
```

**The `key` field is ONLY returned on creation. It cannot be retrieved again.**

#### Revoke API Key (DELETE /api/v1/api-keys/:id)

**Authentication:** Required (session only). **Authorization:** Owner of the key or admin.

**Process:** Set `is_active = false`. Do not hard-delete (for audit trail). Log to audit_log.

---

## 6.4 Authorization Model

All authenticated users (admin and member) have full read and write access to all CRM entities. The following operations are restricted:

| Operation | Who Can Perform It |
|---|---|
| User management (CRUD on `/api/v1/users`) | Admin only |
| Settings management | Admin only |
| API key management | Admin only |
| Email template management | Admin only |
| Audit log viewing | Admin only |
| Delete operations on CRM entities (contacts, companies, deals) | Owner of the entity OR admin |
| All other CRUD (create, read, update, list) on CRM entities | Any authenticated user |

This is a simple two-role model. There is no per-entity permission system, no team-based access control, and no field-level permissions. Every authenticated user can see and edit every contact, company, and deal. The only distinction is between admin (who can manage users and system settings) and member (who cannot).

---

## 6.5 Frontend Routes & Authentication Behavior

### 6.5.1 Route Definitions

| Path | Component | Auth Required | Role Required | Notes |
|---|---|---|---|---|
| `/login` | LoginPage | No | — | Redirects to `/` if already authenticated |
| `/forgot-password` | ForgotPasswordPage | No | — | |
| `/reset-password` | ResetPasswordPage | No | — | Token and email passed as URL query params |
| `/` | Dashboard (or redirect to `/contacts` until Milestone 9) | Yes | — | Default landing page |
| All other routes | Various | Yes | — | Wrapped in `ProtectedRoute` component |
| Admin routes (e.g., `/settings`, `/users`) | Various | Yes | Admin | Wrapped in `AdminRoute` component |

### 6.5.2 ProtectedRoute Component

Wraps all routes that require authentication. Checks if user is authenticated (via user context). If not, redirects to `/login` with the intended destination stored (e.g., in URL search params or state) for post-login redirect.

### 6.5.3 AdminRoute Component

Extends `ProtectedRoute`. Additionally checks `user.role === 'admin'`. If user is authenticated but not admin, shows a 403 page or redirects to `/`.

### 6.5.4 Login Flow

1. User navigates to `/login`.
2. Frontend renders login form with email and password fields.
3. On submit, POST to `/api/v1/auth/login`.
4. On success: store user data in application state (React context or similar), redirect to `/` (or the originally intended route if stored).
5. On `401`: show "Invalid email or password" error on the form.
6. On `403`: show "Account is deactivated. Contact your administrator." error (or the specific lockout message if account is locked).
7. On `429`: show "Too many login attempts. Try again in X minutes." error. Disable the submit button for the remaining duration.

### 6.5.5 Session Restoration

1. On app initialization (before rendering protected routes), call `GET /api/v1/auth/me`.
2. On success: populate user context, allow access to protected routes.
3. On `401`: redirect to `/login`, clear any stale user data.

### 6.5.6 Global 401 Handling

1. Set up an Axios/fetch interceptor that watches for `401` responses on ANY API call.
2. On `401`: clear user context, redirect to `/login`, show a toast notification "Your session has expired. Please log in again."
3. Exception: do NOT redirect on `401` from `/api/v1/auth/login` itself (that's a login failure, not a session expiry).

### 6.5.7 CSRF Token Handling

1. On app initialization (after successful `/api/v1/auth/me`), call `GET /api/v1/auth/csrf-token`.
2. Store the token in memory (NOT localStorage — XSS risk).
3. Add an Axios/fetch interceptor that attaches `X-CSRF-Token` header to all `POST`, `PUT`, `PATCH`, `DELETE` requests.
4. If a `403` with "Invalid CSRF token" is received, re-fetch the CSRF token and retry the request once.
# SECTION 6b: Core Features — Contacts Module

> All API endpoints use the `/api/v1/` prefix. All successful responses use the canonical response format (Section 4): `{ success: true, data: T | T[], meta?: { page, limit, total, totalPages } }`. All error responses use the canonical error format (Section 4): `{ success: false, error: { code, message, details } }`. All response field names are camelCase; the service layer transforms from snake_case DB columns. Validation uses Zod schemas defined in the `shared/` package. Authorization: any authenticated user can create/read/update; delete requires owner or admin.

## Frontend Routes

| Route             | View                     |
|-------------------|--------------------------|
| `/contacts`       | Contact list view        |
| `/contacts/new`   | Contact creation form    |
| `/contacts/:id`   | Contact detail view      |

---

## Contact List View — GET /api/v1/contacts

### Pagination

| Parameter | Type    | Default | Constraints    | Description                        |
|-----------|---------|---------|----------------|------------------------------------|
| `page`    | integer | `1`     | Min 1          | The 1-based page number to return. |
| `limit`   | integer | `25`    | Min 1, Max 100 | Number of records per page.        |

**Edge Cases:**
- If `page` exceeds `totalPages`, return an empty `data` array with correct `meta` (do NOT return 404).
- If `page` is 0, negative, or non-integer, return 400 with canonical error format: `code: "VALIDATION_ERROR"`, `message: "page must be a positive integer"`.
- If `limit` is 0, negative, non-integer, or exceeds 100, return 400: `code: "VALIDATION_ERROR"`, `message: "limit must be an integer between 1 and 100"`.
- If no filters are applied and table is empty, return `{ success: true, data: [], meta: { total: 0, page: 1, limit: 25, totalPages: 0 } }`.

### Sorting

| Parameter | Type   | Default      | Description                                 |
|-----------|--------|--------------|---------------------------------------------|
| `sort`    | string | `created_at` | Field name to sort by.                      |
| `order`   | string | `desc`       | Sort direction: `asc` or `desc` (lowercase).|

**Sortable Fields:**

| Sort Value           | Database Column / Expression                | Notes                                            |
|----------------------|---------------------------------------------|--------------------------------------------------|
| `first_name`         | `contacts.first_name`                       | Case-insensitive (`LOWER(first_name)`)           |
| `last_name`          | `contacts.last_name`                        | Case-insensitive (`LOWER(last_name)`)            |
| `email`              | `contacts.email`                            | Case-insensitive; NULLs sorted last              |
| `company.name`       | `companies.name` (via LEFT JOIN)            | Case-insensitive; contacts with no company last  |
| `status`             | `contacts.status`                           | Alphabetical on status string                    |
| `created_at`         | `contacts.created_at`                       | Timestamp ordering                               |
| `last_contacted_at`  | `contacts.last_contacted_at`                | Timestamp; NULLs sorted last (never contacted)   |
| `owner.last_name`    | `users.last_name` (via LEFT JOIN on owner)  | Case-insensitive; unassigned contacts last       |

**Validation:** If `sort` is not in the allowed list, return 400 with canonical error format. If `order` is not `asc` or `desc`, return 400.

**NULL Handling:** For all nullable sort fields (`email`, `company.name`, `last_contacted_at`, `owner.last_name`): NULLs are always sorted last regardless of sort direction. Use `NULLS LAST` in the SQL query.

**Secondary Sort:** Always apply `contacts.id ASC` as a deterministic secondary sort for stable pagination.

### Filtering

All filters are applied as AND conditions. Multiple filters narrow the result set.

| Query Parameter          | Type                     | Matching Logic                                                               | Validation                                                                 |
|--------------------------|--------------------------|------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| `search`                 | string                   | `ILIKE '%value%'` across `first_name`, `last_name`, `email`, `phone`, `mobile` — combined with OR. Escape `%` and `_` in user input before wrapping with `%`. | Min 1 char. Max 255 chars. |
| `status`                 | string (CSV)             | Exact match. Comma-separated for OR: `?status=active,lead` → `status IN ('active','lead')` | Each value must be a valid status enum value. Invalid values → 400. |
| `company_id`             | UUID                     | Exact match: `company_id = ?`                                                | Must be valid UUID format. Does NOT verify existence (returns empty if no match). |
| `owner_id`               | UUID                     | Exact match: `owner_id = ?`                                                  | Must be valid UUID format. |
| `source`                 | string                   | Exact match: `source = ?`                                                    | Must be a valid source enum value. |
| `tags`                   | string (CSV)             | Array contains ALL specified tags: `tags @> ARRAY[?]`                        | Comma-separated. Each tag trimmed. Empty tags ignored. |
| `created_after`          | ISO 8601 date/datetime   | `created_at >= ?`                                                            | If date only (no time), treat as `YYYY-MM-DDT00:00:00.000Z`. |
| `created_before`         | ISO 8601 date/datetime   | `created_at <= ?`                                                            | If date only, treat as `YYYY-MM-DDT23:59:59.999Z`. |
| `last_contacted_after`   | ISO 8601 date/datetime   | `last_contacted_at >= ?`                                                     | Same as above. |
| `last_contacted_before`  | ISO 8601 date/datetime   | `last_contacted_at <= ?`                                                     | Same as above. |
| `has_email`              | string (`true`/`false`)  | `true` → `email IS NOT NULL AND email != ''`; `false` → `email IS NULL OR email = ''` | Case-insensitive. Other values → 400. |
| `has_phone`              | string (`true`/`false`)  | `true` → `(phone IS NOT NULL AND phone != '') OR (mobile IS NOT NULL AND mobile != '')`; `false` → inverse | Same as `has_email`. |
| `custom_fields`          | key=value pairs          | JSONB exact match filtering only: `custom_fields->>'key' = 'value'`. GIN index on `custom_fields` column. | Keys must be strings. Only exact match supported (no ILIKE, no range). |

**Edge Cases:**
- Empty `search` string (e.g., `?search=` or `?search=   `): ignore the filter entirely.
- Multiple values for single-value filters (e.g., `?company_id=uuid1,uuid2`): return 400.
- Date filters with invalid date strings: return 400 with message identifying the parameter.
- `created_after` > `created_before`: return empty results (valid but impossible range — do NOT error).
- Unrecognized query parameters: ignore silently.

### List Response Shape

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "firstName": "Jane",
      "lastName": "Doe",
      "email": "jane@example.com",
      "phone": "+1-555-0100",
      "jobTitle": "VP of Engineering",
      "company": {
        "id": "uuid",
        "name": "Acme Corp"
      },
      "owner": {
        "id": "uuid",
        "name": "John Smith"
      },
      "status": "active",
      "tags": ["vip", "enterprise"],
      "lastContactedAt": "2026-03-20T14:30:00Z",
      "createdAt": "2026-01-15T09:00:00Z"
    }
  ],
  "meta": {
    "total": 482,
    "page": 1,
    "limit": 25,
    "totalPages": 20
  }
}
```

**Field Details:**
- `company`: `null` if contact has no associated company. Never omit the key — always include it as `null`.
- `owner`: `null` if unassigned. Same rule — always present.
- `owner.name`: Concatenation of `users.first_name + ' ' + users.last_name`.
- `tags`: Always an array, even if empty (`[]`). Never `null`.
- `lastContactedAt`: `null` if never contacted.
- All datetime fields returned in ISO 8601 UTC format with `Z` suffix.
- All string fields returned as-is (no HTML encoding in API response).

### Frontend List View

**Table Layout:**
- Default visible columns: First Name, Last Name, Email, Company, Status, Owner, Last Contacted, Created At.
- All columns listed above plus Phone, Job Title, Tags, Source are available via a "Columns" dropdown button. User can toggle visibility. Column preferences are stored in `localStorage` under key `crm_contact_list_columns` as a JSON array of column IDs.
- Minimum 3 columns must remain visible. If user tries to hide below 3, show toast: "At least 3 columns must be visible."
- Column widths: resizable by dragging column border. Widths stored in `localStorage` under `crm_contact_list_col_widths`.
- Row hover: light background highlight (`bg-muted/50`).
- Row click: navigate to `/contacts/:id`. Entire row is clickable except checkboxes and action buttons.
- Empty state: "No contacts found." with CTA button "Create your first contact" if no contacts exist at all, or "No contacts match your filters." with "Clear filters" link if filters are active.

**Column Header Sort Behavior:**
- Click column header to sort ascending by that field.
- Click again to toggle to descending.
- Click a third time to remove sort (revert to default `created_at desc`).
- Visual indicator: up arrow (ascending), down arrow (descending), no arrow (unsorted).
- Only one sort field active at a time.
- Non-sortable columns (e.g., Phone, Tags): header is not clickable, no hover cursor change.

**Search Bar:**
- Positioned top-left of the table area.
- Placeholder text: "Search contacts..."
- Debounced: 300ms after last keystroke before firing API request.
- Shows a small "x" clear button when text is present.
- On clear, immediately re-fetch without search filter.
- Minimum 1 character to trigger search.
- Search icon (magnifying glass) on the left inside the input.
- While fetching, show a subtle loading spinner inside the search bar (replacing the search icon).

**Filter Panel:**
- Toggle button labeled "Filters" with a filter icon and badge showing active filter count (e.g., "Filters (3)").
- Panel slides in from the right or expands below the toolbar — use a collapsible sidebar pattern.
- Filter controls:
  - **Status**: Multi-select dropdown with checkboxes. Options populated from status enum.
  - **Company**: Searchable dropdown (autocomplete, fetches from `/api/v1/companies?search=`). Shows company name. Selecting sets `company_id`.
  - **Owner**: Searchable dropdown (autocomplete from `/api/v1/users?search=`). Shows user full name.
  - **Source**: Single-select dropdown. Options from source enum.
  - **Tags**: Tag input (type to add, click x to remove). Comma-separated sent to API.
  - **Created Date Range**: Two date pickers (from/to).
  - **Last Contacted Date Range**: Two date pickers (from/to).
  - **Has Email**: Toggle switch (Yes / No / Any). Default: Any (no filter).
  - **Has Phone**: Toggle switch (Yes / No / Any). Default: Any.
- "Apply Filters" button at bottom of panel.
- "Clear All" link to reset all filters.
- Active filters shown as removable chips/badges above the table: e.g., "Status: Active, Lead x", "Owner: John Smith x". Clicking x removes that filter and re-fetches.

**Pagination Controls:**
- Bottom of table, full width.
- Left side: "Showing {start}-{end} of {total} contacts".
- Center: Page number buttons. Show first page, last page, current page +/- 2, with ellipsis.
- Prev/Next arrow buttons (disabled on first/last page respectively).
- Right side: "Items per page" dropdown: options 10, 25, 50, 100. Changing resets to page 1.
- Keyboard: Left/Right arrows for prev/next page when pagination is focused.

**"New Contact" Button:**
- Top-right corner of the list view. Primary button style. Label: "+ New Contact".
- Click: opens a slide-over panel from the right (keeps user on the list).
- On successful creation: close panel, show success toast "Contact created", re-fetch list.

**"Export to CSV" Button:**
- Located in toolbar area, secondary button style. Label: "Export" with download icon.
- Click: triggers `GET /api/v1/contacts/export` with current active filters as query params.
- Shows loading state on button while downloading.
- If more than 10,000 records match: API returns 400, frontend shows error toast.

**"Import from CSV" Button:**
- Located in toolbar area, secondary button style. Visible only to admin users.
- Label: "Import" with upload icon.
- Click: opens import modal/wizard (see Import Contacts section below).

**Responsive:** Below 768px, the contacts table renders as stacked cards showing name, email, company, and status. Filter panel becomes a slide-out drawer triggered by a filter icon button. Bulk action bar sticks to bottom of screen on mobile.

---

## Contact Detail View — GET /api/v1/contacts/:id

### API Response Shape

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane@example.com",
    "phone": "+1-555-0100",
    "mobile": "+1-555-0101",
    "jobTitle": "VP of Engineering",
    "department": "Engineering",
    "company": {
      "id": "uuid",
      "name": "Acme Corp",
      "domain": "acme.com",
      "industry": "Technology"
    },
    "owner": {
      "id": "uuid",
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@crm.com",
      "avatarUrl": "https://..."
    },
    "status": "active",
    "source": "website",
    "tags": ["vip", "enterprise"],
    "addressLine1": "123 Main St",
    "addressLine2": "Suite 400",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94105",
    "country": "US",
    "customFields": {},
    "lastContactedAt": "2026-03-20T14:30:00Z",
    "createdAt": "2026-01-15T09:00:00Z",
    "updatedAt": "2026-03-24T10:00:00Z"
  }
}
```

**Error Handling:**
- If `:id` is not a valid UUID format: return 400 with canonical error format: `code: "VALIDATION_ERROR"`, `message: "Invalid contact ID format"`.
- If contact not found: return 404: `code: "NOT_FOUND"`, `message: "Contact not found"`.

### Overview Section (Frontend)

**Layout:** Two-column layout on desktop (left: main info, right: sidebar with metadata). Single column on mobile.

**Left Column — Contact Information Form:**
- All fields displayed in an inline-editable form. Default state: read mode. Click a field or click "Edit" button to switch to edit mode.
- **Name:** Display as "Jane Doe" in read mode. In edit mode: two fields (First Name, Last Name) side by side.
- **Email:** Displayed as clickable `mailto:` link in read mode. In edit mode: text input with email validation.
- **Phone / Mobile:** Displayed as clickable `tel:` links. In edit mode: text inputs.
- **Job Title / Department:** Plain text / text input.
- **Address:** Displayed as formatted multi-line address. In edit mode: individual fields (line1, line2, city, state, postal, country). Country is a searchable dropdown.
- **Custom Fields:** Rendered dynamically based on field type definitions.
- **Source:** Displayed as badge. In edit mode: dropdown of allowed source values.
- **Save / Cancel buttons** appear when in edit mode. Save calls `PUT /api/v1/contacts/:id` with only the changed fields + `updatedAt` for optimistic locking. Cancel reverts all changes.
- On save conflict (409): show modal "This contact was updated by {user} at {time}. Reload the latest version?" with "Reload" and "Overwrite" buttons.
- On save success: show success toast "Contact updated". Switch back to read mode.
- On validation error: highlight invalid fields with red border and inline error message below field.

**Right Column — Sidebar:**
- **Company Link:** Clickable link to `/companies/:companyId`. If no company: "No company" with "+" button to associate via searchable dropdown. "x" to disassociate (sets `companyId: null`).
- **Owner Assignment:** Dropdown showing current owner (avatar + name). Click to open searchable dropdown of all active users. Selecting saves immediately. Confirm if changing from another user.
- **Status Badge:** Colored badge. Click to open status dropdown. Saves immediately.
  - `active` → green, `lead` → blue, `prospect` → purple, `customer` → teal, `inactive` → gray, `churned` → red
- **Tags:** Displayed as removable chips. "Add tag" input with autocomplete. Tags save immediately on add/remove.
- **Key Dates:** Created, Last Updated, Last Contacted (or "Never").

**Quick Actions Bar:**
- **Log Call** (phone icon): Modal → creates activity `type: 'call'`. Updates `lastContactedAt`.
- **Send Email** (mail icon): Modal → creates activity `type: 'email'`. Updates `lastContactedAt`. NOTE: Logs the email only, does NOT send an actual email.
- **Schedule Meeting** (calendar icon): Modal → creates activity `type: 'meeting'`.
- **Add Note** (notepad icon): Modal → creates activity `type: 'note'`.

### Activity Timeline

Vertical timeline, most recent at top. API: `GET /api/v1/activities?contact_id=:contactId&sort=created_at&order=desc&limit=20` with cursor-based pagination for infinite scroll.

Each entry: type icon (colored circle) + subject (bold) + description preview (first 150 chars) + metadata line (date, duration, creator) + action menu (edit, delete).

**Filter Timeline:** Horizontal pill buttons: "All", "Calls", "Emails", "Meetings", "Notes", "Tasks". Multiple selectable (OR logic).

**Infinite Scroll:** Load 20 initially. Fetch next page on scroll within 200px of bottom. "No more activities" when all loaded.

### Related Records Tabs

Tabs: Deals | Notes | Attachments | Activity Log.

**Deals Tab:**
- Table of deals where `contact_id = :contactId`. Columns: Title, Stage, Amount, Expected Close Date, Owner.
- Row click → `/deals/:dealId`. "Add Deal" button with `contactId` pre-filled.
- "Unlink" action per row (sets `contact_id = null` on the deal, does not delete).

**Notes Tab:**
- List of notes, most recent first. Pin toggle, add, edit, delete. Markdown rendering.

**Attachments Tab:**
- Upload via `POST /api/v1/contacts/:id/attachments` (multipart). Max 25MB per file. Stored in R2.
- Download via `GET /api/v1/attachments/:id/download` → 302 redirect to signed R2 URL (5-minute expiry).
- Delete: `DELETE /api/v1/attachments/:id` (deletes R2 object + DB record).
- Drag-and-drop upload supported.

**Activity Log Tab:** Same timeline component in full tab layout with additional date range and owner filters.

### Audit Trail

Collapsible section at bottom. API: `GET /api/v1/audit-log?entity_type=contact&entity_id=:contactId&sort=created_at&order=desc&limit=50`.

Each entry: action icon + "{User Name} {action} this contact" + relative timestamp (hover for exact) + expandable field-level changes for updates.

---

## Create Contact — POST /api/v1/contacts

### Request Body

```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane@example.com",
  "phone": "+1-555-0100",
  "mobile": "+1-555-0101",
  "jobTitle": "VP of Engineering",
  "department": "Engineering",
  "companyId": "uuid-of-company",
  "ownerId": "uuid-of-user",
  "addressLine1": "123 Main St",
  "addressLine2": "Suite 400",
  "city": "San Francisco",
  "state": "CA",
  "postalCode": "94105",
  "country": "US",
  "source": "website",
  "status": "active",
  "tags": ["vip", "enterprise"],
  "customFields": { "referral_source": "Google" },
  "notesText": "Met at tech conference"
}
```

### Validation Rules (Zod schema in `shared/`)

| Field          | Required | Type          | Min | Max    | Format / Constraints                                                          | Default        |
|----------------|----------|---------------|-----|--------|-------------------------------------------------------------------------------|----------------|
| `firstName`    | Yes      | string        | 1   | 100    | Trim. After trim, must be >= 1 char.                                          | —              |
| `lastName`     | Yes      | string        | 1   | 100    | Same as firstName.                                                            | —              |
| `email`        | No       | string        | —   | 255    | RFC 5322 email format if provided. Trim. Lowercase before storing. Empty string → null. | `null` |
| `phone`        | No       | string        | —   | 50     | Trim. No format enforcement. Empty string → null.                             | `null`         |
| `mobile`       | No       | string        | —   | 50     | Same as phone.                                                                | `null`         |
| `jobTitle`     | No       | string        | —   | 150    | Trim. Empty string → null.                                                    | `null`         |
| `department`   | No       | string        | —   | 100    | Trim. Empty string → null.                                                    | `null`         |
| `companyId`    | No       | UUID          | —   | —      | Must reference existing record in `companies` table. UUID valid but not found → 400. | `null` |
| `ownerId`      | No       | UUID          | —   | —      | Must reference existing active user. Not found → 400.                         | Current user.  |

**Frontend entity selector requirements for Contact form:**
- **Company** (`companyId`) — `EntitySelector` component searching companies by name/domain. Shows company name + domain in dropdown. On selection, sets `companyId`. NOT a text input.
- **Owner** (`ownerId`) — `EntitySelector` component searching users by first/last name. Shows user name + email in dropdown. Defaults to current user. NOT a text input.
| `addressLine1` | No       | string        | —   | 255    | Trim.                                                                         | `null`         |
| `addressLine2` | No       | string        | —   | 255    | Trim.                                                                         | `null`         |
| `city`         | No       | string        | —   | 100    | Trim.                                                                         | `null`         |
| `state`        | No       | string        | —   | 100    | Trim.                                                                         | `null`         |
| `postalCode`   | No       | string        | —   | 20     | Trim.                                                                         | `null`         |
| `country`      | No       | string        | —   | 100    | Trim.                                                                         | `null`         |
| `source`       | No       | string        | —   | —      | Must be one of: `website`, `referral`, `linkedin`, `cold_call`, `trade_show`, `other`. Case-sensitive. | `null` |
| `status`       | No       | string        | —   | —      | Must be one of: `active`, `inactive`, `lead`, `prospect`, `customer`, `churned`. Case-sensitive. | `active` |
| `tags`         | No       | array         | —   | —      | Array of strings. Each: trimmed, max 50 chars, non-empty. Max 20 tags. Duplicates removed silently. | `[]` |
| `customFields` | No       | object        | —   | —      | Flat JSON (no nested objects). Keys: strings. Values: string, number, boolean, or null. Max 10KB. | `{}` |
| `notesText`    | No       | string (text) | —   | 10000  | If provided and non-empty: creates a note record linked to this contact.      | —              |

### Validation Error Response

Returns canonical error format (Section 4) with HTTP 400:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "field": "firstName", "message": "First name is required" },
      { "field": "email", "message": "Invalid email format" }
    ]
  }
}
```

Return ALL validation errors at once (do not fail on first error). `field` matches camelCase request body key.

### Process Flow

1. **Parse and validate** all input fields per Zod schema. Collect all errors. If any, return 400.
2. **Duplicate check:** `SELECT id, first_name, last_name, email FROM contacts WHERE LOWER(email) = LOWER(:email) AND email IS NOT NULL`. If match found, do NOT block — include `warning` in response.
3. **Insert contact** into `contacts` table. Generate UUID. Set `created_at` and `updated_at` to `NOW()`. Set `last_contacted_at` to `null`.
4. **Create initial note** if `notesText` provided.
5. **Audit log:** action `create`, entity_type `contact`, full snapshot in `changes.after`.
6. **Return** HTTP 201 with expanded `company` and `owner`.

### Success Response

HTTP 201:

```json
{
  "success": true,
  "data": {
    "id": "new-uuid",
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane@example.com",
    "phone": "+1-555-0100",
    "mobile": "+1-555-0101",
    "jobTitle": "VP of Engineering",
    "department": "Engineering",
    "company": {
      "id": "uuid",
      "name": "Acme Corp",
      "domain": "acme.com",
      "industry": "Technology"
    },
    "owner": {
      "id": "uuid",
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@crm.com",
      "avatarUrl": null
    },
    "status": "active",
    "source": "website",
    "tags": ["enterprise", "vip"],
    "addressLine1": "123 Main St",
    "addressLine2": "Suite 400",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94105",
    "country": "US",
    "customFields": { "referral_source": "Google" },
    "lastContactedAt": null,
    "createdAt": "2026-03-25T12:00:00Z",
    "updatedAt": "2026-03-25T12:00:00Z"
  },
  "warning": "A contact with email jane@example.com already exists (Jane Doe, ID: uuid). You may want to review."
}
```

`warning` is `null` if no duplicate detected.

---

## Update Contact — PUT /api/v1/contacts/:id

### Request Body

Same shape as create, but ALL fields are optional. Only include fields that are changing. Additionally:

| Field       | Required | Description                                                                 |
|-------------|----------|-----------------------------------------------------------------------------|
| `updatedAt` | Yes      | The `updatedAt` timestamp the client last received. ISO 8601 string. Used for optimistic locking. |

Example — updating just status and tags:

```json
{
  "status": "customer",
  "tags": ["vip", "enterprise", "renewed"],
  "updatedAt": "2026-03-24T10:00:00Z"
}
```

### Validation

- Same Zod schema rules as Create for each present field.
- `updatedAt` is required. If missing, return 400: `code: "VALIDATION_ERROR"`, `message: "updatedAt is required for conflict detection"`.
- Fields not included in the body are NOT changed (partial update semantics).
- To set a nullable field to null, include it as `null` (e.g., `{ "companyId": null }`).

### Optimistic Locking (Conflict Detection)

1. Before applying updates, fetch current record. Compare DB `updated_at` with request `updatedAt`.
2. If mismatch → HTTP 409 Conflict with canonical error format:

```json
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "This contact has been modified since you last loaded it.",
    "details": {
      "currentUpdatedAt": "2026-03-25T08:00:00Z",
      "updatedBy": {
        "id": "uuid",
        "name": "Other User"
      }
    }
  }
}
```

3. If match: proceed. Set `updated_at = NOW()`.

### Audit Log

Record before/after for changed fields only. If no fields actually changed, return 200 with current record but do NOT create audit log entry.

### Success Response

HTTP 200 with canonical response format, same data shape as GET /api/v1/contacts/:id.

---

## Delete Contact — DELETE /api/v1/contacts/:id

### Authorization

- Admin users: can delete any contact.
- Regular users: can only delete contacts they own (`owner_id = current_user.id`). Otherwise return 403: `code: "FORBIDDEN"`, `message: "You do not have permission to delete this contact"`.

### Cascade Behavior (Hard Delete)

All deletes within a single database transaction (except R2 operations):

1. Delete all activities where `contact_id = :id` (CASCADE).
2. Delete all notes where `contact_id = :id` (CASCADE).
3. For each attachment where `contact_id = :id`:
   - Delete the file from R2 storage using `storage_path`.
   - Delete the attachment DB record.
   - If R2 deletion fails: log error but continue (orphaned files cleaned by periodic job).
4. For deals where `contact_id = :id`: SET `contact_id = NULL` (deals are NOT deleted).
5. Delete the contact record.
6. Audit log with snapshot of deleted contact in `changes.before`.

If any DB operation fails, rollback the entire transaction and return 500.

### Frontend Confirmation

1. Call `GET /api/v1/contacts/:id/related-counts` to get counts:
   ```json
   { "success": true, "data": { "activities": 15, "notes": 3, "attachments": 2, "deals": 2 } }
   ```
2. Show confirmation modal with breakdown. Lines with 0 count omitted.
3. User must type the contact's full name to confirm if more than 10 total related records.

### Response

HTTP 200:

```json
{
  "success": true,
  "data": {
    "deleted": {
      "activities": 15,
      "notes": 3,
      "attachments": 2
    },
    "updated": {
      "dealsUnlinked": 2
    }
  }
}
```

---

## Import Contacts — POST /api/v1/contacts/import

### Authorization

Admin users only. Non-admin → 403.

### Request

- `Content-Type: multipart/form-data`
- Field name: `file`
- Accept `.csv` extension with `text/csv` or `application/octet-stream` MIME type.
- Max file size: 5MB.
- Query param: `?duplicates=skip|warn` — controls duplicate email handling. `skip` = silently skip rows with duplicate emails. `warn` = create the contact but include warnings. Default: `warn`.

### CSV Parsing

**Required Columns:** `first_name`, `last_name`.

**Optional Columns:**

| CSV Column      | Maps to DB Field    | Notes                                                             |
|-----------------|---------------------|-------------------------------------------------------------------|
| `email`         | `email`             | Validated as email format. Invalid → row error.                   |
| `phone`         | `phone`             | No format validation.                                             |
| `mobile`        | `mobile`            | No format validation.                                             |
| `job_title`     | `job_title`         |                                                                   |
| `department`    | `department`        |                                                                   |
| `company_name`  | → lookup/create     | Case-insensitive match against `companies.name`. If not found, create new company. |
| `source`        | `source`            | Must be valid source enum. Invalid → row error.                   |
| `status`        | `status`            | Must be valid status enum. Invalid → row error. Default: `active`.|
| `tags`          | `tags`              | Semicolon-separated: `vip;enterprise`.                            |
| `address_line1` | `address_line1`     |                                                                   |
| `address_line2` | `address_line2`     |                                                                   |
| `city`          | `city`              |                                                                   |
| `state`         | `state`             |                                                                   |
| `postal_code`   | `postal_code`       |                                                                   |
| `country`       | `country`           |                                                                   |
| `custom_fields` | `custom_fields`     | JSON string. Must be valid JSON object. Invalid → row error.      |

**Column Name Matching:**
- Case-insensitive. Trim whitespace. Underscores, spaces, and hyphens interchangeable.
- Unrecognized columns silently ignored.
- Missing `first_name` or `last_name` headers → 400.

### Processing Rules

1. **Row limit:** Max 1000 data rows (excluding header). More → 400.
2. **Parse all rows first** — validate every row before creating any records.
3. **Encoding:** UTF-8. Strip BOM (`\xEF\xBB\xBF`) if present.
4. **Line endings:** Handle `\n`, `\r\n`, and `\r`.
5. **Quoted fields:** Standard CSV quoting (double-quote enclosure, escaped quotes as `""`).
6. **Empty rows:** Skip silently.
7. **Company matching/creation:** Case-insensitive match. Create at most once per unique name per import.
8. **Duplicate email handling:** Controlled by `?duplicates=` query param. `skip` → do not create. `warn` → create and include in `duplicateWarnings`.
9. **Owner:** All imported contacts get `owner_id` = importing user.
10. **Audit log:** One entry per created contact with metadata `"imported"`.

### Response

HTTP 200:

```json
{
  "success": true,
  "data": {
    "summary": {
      "total": 150,
      "created": 142,
      "skipped": 3,
      "errors": 5,
      "companiesCreated": 8
    },
    "errors": [
      { "row": 23, "field": "email", "message": "Invalid email format: not-an-email" }
    ],
    "duplicateWarnings": [
      { "row": 10, "email": "jane@example.com", "existingContactId": "uuid", "existingContactName": "Jane Doe" }
    ]
  }
}
```

`row` is 1-based relative to data rows (header not counted).

---

## Export Contacts — GET /api/v1/contacts/export

### Authorization

All authenticated users.

### Query Parameters

Same as Contact List filtering parameters. NO pagination params.

### Limits

Max 10,000 records. If more → 400: `code: "EXPORT_LIMIT_EXCEEDED"`, `message: "Export limited to 10,000 records. Please narrow your filters."`. Count query runs first.

### CSV Output

**Headers:**
```
first_name,last_name,email,phone,mobile,job_title,department,company_name,company_domain,owner_name,status,source,tags,address_line1,address_line2,city,state,postal_code,country,last_contacted_at,created_at
```

**Formatting Rules:**
- NULL values → empty string (not the string "null").
- Fields with commas, quotes, or newlines enclosed in double quotes. Quotes escaped as `""`.
- Line ending: `\r\n`. Encoding: UTF-8 with BOM for Excel compatibility.
- Sort: current list sort or default `created_at desc`.

### Response Headers
```
Content-Type: text/csv; charset=utf-8
Content-Disposition: attachment; filename="contacts_export_2026-03-25.csv"
```

---

## Bulk Operations

### POST /api/v1/contacts/bulk-delete

**Authorization:** Owner of all specified contacts or admin.

**Request Body:**
```json
{
  "ids": ["uuid-1", "uuid-2", "uuid-3"]
}
```

- Max 100 IDs per request. More → 400.
- Each contact follows the same cascade behavior as single delete (activities, notes, attachments CASCADE; attachments R2 files deleted; deals SET contact_id=NULL).
- All within a single transaction (except R2 ops).

**Response:**
```json
{
  "success": true,
  "data": {
    "affected": 3
  }
}
```

### POST /api/v1/contacts/bulk-update

**Authorization:** Any authenticated user.

**Request Body:**
```json
{
  "ids": ["uuid-1", "uuid-2"],
  "updates": {
    "ownerId": "uuid",
    "status": "active",
    "addTags": ["vip"]
  }
}
```

- Max 100 IDs per request. More → 400.
- Only `ownerId`, `status`, and `addTags` are allowed in `updates`. Other fields → 400.
- `addTags` merges with existing tags (no duplicates).
- Creates audit log entry per affected record.

**Response:**
```json
{
  "success": true,
  "data": {
    "affected": 2
  }
}
```

**Frontend Bulk Actions:**
- Checkbox column on far left. "Select All" selects current page only.
- When rows selected, floating action bar: "{N} contacts selected" with Delete, Change Owner, Change Status, Add Tags buttons.
- Delete: confirmation modal, red destructive button.
- After completion: success toast, deselect all, re-fetch list.

---

---

# SECTION 6c: Core Features — Companies Module

> Same canonical format conventions as Section 6b. All endpoints under `/api/v1/`. Responses use canonical response format (Section 4). Errors use canonical error format (Section 4). Validation via Zod in `shared/`. Authorization: any authenticated user can create/read/update; delete requires owner or admin.

## Frontend Routes

| Route              | View                      |
|--------------------|---------------------------|
| `/companies`       | Company list view         |
| `/companies/new`   | Company creation form     |
| `/companies/:id`   | Company detail view       |

---

## Company List View — GET /api/v1/companies

### Pagination

Same as Contacts: `page` (default 1), `limit` (default 25, max 100). Response meta: `{ page, limit, total, totalPages }`. Same edge case handling.

### Sorting

| Parameter | Type   | Default      | Description           |
|-----------|--------|--------------|-----------------------|
| `sort`    | string | `created_at` | Field name to sort by.|
| `order`   | string | `desc`       | `asc` or `desc`.      |

**Sortable Fields:**

| Sort Value         | Database Column / Expression         | Notes                                                |
|--------------------|--------------------------------------|------------------------------------------------------|
| `name`             | `companies.name`                     | Case-insensitive (`LOWER(name)`)                     |
| `industry`         | `companies.industry`                 | Case-insensitive; NULLs last                         |
| `size`             | `companies.size`                     | Ordered by size ordinal (not alphabetical). NULLs last. |
| `owner.last_name`  | `users.last_name` (via LEFT JOIN)    | Case-insensitive; unassigned last                    |
| `created_at`       | `companies.created_at`               | Timestamp                                            |
| `annual_revenue`   | `companies.annual_revenue`           | Numeric; NULLs last                                  |

**Company Size Values (ordinal order):** `1-10` (1) < `11-50` (2) < `51-200` (3) < `201-500` (4) < `501-1000` (5) < `1001-5000` (6) < `5000+` (7). Sort uses this ordinal, not alphabetical.

**Secondary Sort:** `companies.id ASC`.

### Filtering

| Query Parameter    | Type                     | Matching Logic                                                   | Validation                                |
|--------------------|--------------------------|------------------------------------------------------------------|-------------------------------------------|
| `search`           | string                   | `ILIKE '%value%'` across `name`, `domain`, `email` (OR). Escape `%` and `_`. | Same rules as contacts search.  |
| `industry`         | string                   | Exact match: `industry = ?`                                      | No enum validation (free-text field).     |
| `size`             | string (CSV)             | `size IN (?)` for comma-separated                                | Each value must be valid size enum.       |
| `owner_id`         | UUID                     | Exact match: `owner_id = ?`                                      | Valid UUID format.                        |
| `tags`             | string (CSV)             | `tags @> ARRAY[?]` (contains all specified tags)                 | Same as contacts.                         |
| `created_after`    | ISO date/datetime        | `created_at >= ?`                                                | Valid ISO date.                           |
| `created_before`   | ISO date/datetime        | `created_at <= ?`                                                | Valid ISO date.                           |
| `revenue_min`      | number                   | `annual_revenue >= ?`                                            | Non-negative number.                      |
| `revenue_max`      | number                   | `annual_revenue <= ?`                                            | Non-negative number.                      |

**Edge Cases:**
- `revenue_min` > `revenue_max`: return empty results.
- Revenue filters exclude companies with NULL `annual_revenue`.

### List Response Shape

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "Acme Corp",
      "domain": "acme.com",
      "industry": "Technology",
      "size": "201-500",
      "phone": "+1-555-0200",
      "email": "info@acme.com",
      "website": "https://acme.com",
      "owner": {
        "id": "uuid",
        "name": "John Smith"
      },
      "tags": ["enterprise", "tech"],
      "annualRevenue": 5000000,
      "contactCount": 12,
      "dealCount": 3,
      "createdAt": "2025-12-01T09:00:00Z"
    }
  ],
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 25,
    "totalPages": 6
  }
}
```

- `contactCount` and `dealCount`: computed via subquery/join.
- `owner`: `null` if unassigned (key always present).
- `annualRevenue`: `null` if not set.

### Frontend

**Table View:**
- Default columns: Name, Industry, Size, Owner, Contacts, Deals, Revenue, Created At.
- Additional: Domain, Email, Phone, Website, Tags.
- Preferences in `localStorage` key `crm_company_list_columns`.
- Same column resize, sort, hover, click behaviors as Contacts.
- Row click → `/companies/:id`.
- Empty state: "No companies found." with appropriate CTA.

**Search + Filter Panel:**
- Search: placeholder "Search companies...", debounced 300ms.
- Filters: Industry (autocomplete), Size (multi-select checkboxes), Owner, Tags, Created Date Range, Revenue Range (two number inputs with currency prefix).
- Active filter chips. "Apply Filters", "Clear All".

**Toolbar Buttons:**
- "+ New Company" (top-right, primary).
- "Export" (secondary, same pattern as contacts → `GET /api/v1/companies/export`).
- No import button for companies in v1 (auto-created during contact import).

**Responsive:** Below 768px, companies table renders as stacked cards showing name, industry, size, and contact count.

---

## Company Detail View — GET /api/v1/companies/:id

### API Response Shape

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "Acme Corp",
    "domain": "acme.com",
    "industry": "Technology",
    "size": "201-500",
    "phone": "+1-555-0200",
    "email": "info@acme.com",
    "website": "https://acme.com",
    "description": "Leading provider of...",
    "owner": {
      "id": "uuid",
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@crm.com",
      "avatarUrl": null
    },
    "tags": ["enterprise", "tech"],
    "annualRevenue": 5000000,
    "addressLine1": "456 Corporate Blvd",
    "addressLine2": null,
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "US",
    "customFields": {},
    "createdAt": "2025-12-01T09:00:00Z",
    "updatedAt": "2026-03-20T14:00:00Z"
  }
}
```

**Error Handling:** Invalid UUID → 400. Not found → 404. Both use canonical error format.

### Overview Section (Frontend)

**Layout:** Same two-column layout as Contact Detail.

**Left Column — Company Information Form:**
- Inline-editable (same read/edit mode pattern).
- Fields: Name, Domain (clickable link in read mode, no `http://` prefix — just domain), Industry, Size (dropdown), Phone (`tel:` link), Email (`mailto:` link), Website (clickable external link), Description (textarea), Address fields, Custom Fields.
- Domain validation: basic domain format `^[a-zA-Z0-9][a-zA-Z0-9-]{0,61}[a-zA-Z0-9]?\.[a-zA-Z]{2,}$`.
- Website validation: must start with `http://` or `https://`.
- Save/Cancel, optimistic locking, conflict resolution — same as contacts.

**Right Column — Sidebar:**
- **Owner Assignment:** Same as contacts.
- **Tags:** Same as contacts.
- **Key Metrics** (read-only):
  - Contacts count (link to `/contacts?company_id=:id`)
  - Deals count (link to filtered deals list)
  - Total Deal Value (sum of all open deal amounts)
  - Open Deals Value (sum where stage is not won/lost)
  - Activities count
- **Key Dates:** Created, Last Updated.

**API for Metrics:** `GET /api/v1/companies/:id/metrics`

```json
{
  "success": true,
  "data": {
    "contactCount": 12,
    "dealCount": 5,
    "totalDealValue": 1250000,
    "openDealValue": 750000,
    "activityCount": 89
  }
}
```

### Related Records Tabs

**Contacts Tab:**
- Table: Name, Email, Phone, Job Title, Status, Last Contacted. Sort by last_name asc.
- Row click → `/contacts/:contactId`.
- "New Contact" (pre-fills `companyId`) and "Add Existing" (searchable dropdown).
- "Remove" per row: sets `company_id = NULL` on contact (does not delete).

**Deals Tab:**
- Table: Title, Stage (badge), Amount, Expected Close, Owner, Probability.
- Sort by expected_close_date asc. Row click → `/deals/:dealId`.
- "Add Deal" (pre-fills `companyId`).

**Activities Tab:**
- Timeline filtered to `company_id = :companyId` (includes company activities AND activities for contacts at this company).
- "Log Activity" pre-fills `companyId`. Filter pills. Infinite scroll.

**Notes Tab:** Same as Contact detail notes, for entity_type 'company'.

**Attachments Tab:** Same as Contact detail attachments, for entity_type 'company'.

---

## Create Company — POST /api/v1/companies

### Request Body

```json
{
  "name": "Acme Corp",
  "domain": "acme.com",
  "industry": "Technology",
  "size": "201-500",
  "phone": "+1-555-0200",
  "email": "info@acme.com",
  "website": "https://acme.com",
  "description": "Leading provider of innovative solutions.",
  "ownerId": "uuid",
  "addressLine1": "456 Corporate Blvd",
  "addressLine2": null,
  "city": "New York",
  "state": "NY",
  "postalCode": "10001",
  "country": "US",
  "tags": ["enterprise", "tech"],
  "customFields": {},
  "annualRevenue": 5000000
}
```

### Validation Rules (Zod schema in `shared/`)

| Field           | Required | Type          | Min | Max        | Format / Constraints                                                  | Default        |
|-----------------|----------|---------------|-----|------------|------------------------------------------------------------------------|----------------|
| `name`          | Yes      | string        | 1   | 255        | Trim. After trim, >= 1 char.                                           | —              |
| `domain`        | No       | string        | —   | 255        | Basic domain format (no protocol). Trim. Lowercase. Empty → null.      | `null`         |
| `industry`      | No       | string        | —   | 100        | Trim. Free-text. Empty → null.                                         | `null`         |
| `size`          | No       | string        | —   | —          | Must be one of: `1-10`, `11-50`, `51-200`, `201-500`, `501-1000`, `1001-5000`, `5000+`. | `null` |
| `phone`         | No       | string        | —   | 50         | Trim. No format enforcement.                                           | `null`         |
| `email`         | No       | string        | —   | 255        | Valid email if provided. Trim. Lowercase.                              | `null`         |
| `website`       | No       | string        | —   | 500        | Must start with `http://` or `https://` if provided. Trim.            | `null`         |
| `description`   | No       | string (text) | —   | 10000      | Trim.                                                                  | `null`         |
| `ownerId`       | No       | UUID          | —   | —          | Must reference existing active user.                                   | Current user.  |
| `addressLine1`  | No       | string        | —   | 255        | Trim.                                                                  | `null`         |
| `addressLine2`  | No       | string        | —   | 255        | Trim.                                                                  | `null`         |
| `city`          | No       | string        | —   | 100        | Trim.                                                                  | `null`         |
| `state`         | No       | string        | —   | 100        | Trim.                                                                  | `null`         |
| `postalCode`    | No       | string        | —   | 20         | Trim.                                                                  | `null`         |
| `country`       | No       | string        | —   | 100        | Trim.                                                                  | `null`         |
| `tags`          | No       | array         | —   | —          | Array of strings. Each max 50 chars. Max 20 tags. Deduplicated.        | `[]`           |
| `customFields`  | No       | object        | —   | —          | Flat JSON. Max 10KB. GIN index.                                        | `{}`           |
| `annualRevenue` | No       | number        | —   | —          | Positive number (> 0) or null. Max 2 decimal places. Max 999,999,999,999.99. | `null` |

### Process Flow

1. **Validate** all fields via Zod. Return all errors at once (400 with canonical error format).
2. **Duplicate check:** If `domain` provided: `SELECT id, name FROM companies WHERE LOWER(domain) = LOWER(:domain)`. If found, include warning (don't block).
3. **Insert** company. Generate UUID. Set `created_at`, `updated_at` to NOW().
4. **Audit log:** action `create`, entity_type `company`.
5. **Return** HTTP 201 with canonical response format, expanded owner. Optional `warning`.

---

## Update Company — PUT /api/v1/companies/:id

- Partial update: only included fields are changed.
- `updatedAt` required in request body for optimistic locking.
- Server compares `updatedAt` with DB `updated_at`. If mismatch → 409 with canonical error format.
- Audit log with before/after for changed fields only.
- HTTP 200 with full company detail in canonical response format.

---

## Delete Company — DELETE /api/v1/companies/:id

### Authorization

- Admin users: any company.
- Regular users: only companies they own. Otherwise → 403 with canonical error format.

### Cascade Behavior (Hard Delete)

All within a single transaction (except R2 ops):

1. **Contacts** where `company_id = :id`: SET `company_id = NULL`. Contacts are NOT deleted.
2. **Deals** where `company_id = :id`: SET `company_id = NULL`. Deals are NOT deleted.
3. **Activities** where `company_id = :id` (directly associated): DELETE (CASCADE).
4. **Notes** where `entity_type = 'company' AND entity_id = :id`: DELETE (CASCADE).
5. **Attachments** where `entity_type = 'company' AND entity_id = :id`: Delete R2 files + DB records. Log R2 failures, don't block.
6. **Delete** the company record.
7. **Audit log** with snapshot of deleted company.

### Frontend Confirmation

Same pattern as Contact delete: fetch related counts, confirmation modal with breakdown, name-typing for > 10 related records.

### Response

HTTP 200:

```json
{
  "success": true,
  "data": {
    "deleted": {
      "activities": 25,
      "notes": 5,
      "attachments": 3
    },
    "updated": {
      "contactsUnlinked": 12,
      "dealsUnlinked": 3
    }
  }
}
```

---

## Bulk Operations

### POST /api/v1/companies/bulk-delete

**Authorization:** Owner of all specified companies or admin.

**Request Body:**
```json
{
  "ids": ["uuid-1", "uuid-2"]
}
```

- Max 100 IDs. Each company follows same cascade as single delete.

**Response:**
```json
{
  "success": true,
  "data": {
    "affected": 2
  }
}
```

### POST /api/v1/companies/bulk-update

**Authorization:** Any authenticated user.

**Request Body:**
```json
{
  "ids": ["uuid-1", "uuid-2"],
  "updates": {
    "ownerId": "uuid",
    "addTags": ["enterprise"]
  }
}
```

- Max 100 IDs. Allowed updates: `ownerId`, `addTags`.
- `addTags` merges with existing (no duplicates).
- Audit log per record.

**Response:**
```json
{
  "success": true,
  "data": {
    "affected": 2
  }
}
```

---

---

# SECTION 6d: Core Features — Deals/Pipeline Module

> Same canonical format conventions. All endpoints under `/api/v1/`. Deal stages are loaded from the `deal_stages` table (not hardcoded). Stage validation is at the application level against `deal_stages` rows. Validation via Zod in `shared/`.

## Frontend Routes

| Route          | View                |
|----------------|---------------------|
| `/deals`       | Deal list/pipeline  |
| `/deals/new`   | Deal creation form  |
| `/deals/:id`   | Deal detail view    |

---

## Deal List View — GET /api/v1/deals

### Two View Modes

**View Toggle:** Button group: "Table" (list icon) and "Pipeline" (kanban icon). Preference in `localStorage` key `crm_deals_view_mode`. Default: Pipeline.

#### Table View

Standard list with pagination, sorting, filtering — same patterns as Contacts/Companies.

**List Response Shape:**

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "title": "Acme Corp Enterprise Deal",
      "stage": "proposal",
      "amount": 50000.00,
      "currency": "USD",
      "probability": 60,
      "expectedCloseDate": "2026-06-15",
      "company": {
        "id": "uuid",
        "name": "Acme Corp"
      },
      "contact": {
        "id": "uuid",
        "name": "Jane Doe"
      },
      "owner": {
        "id": "uuid",
        "name": "John Smith"
      },
      "tags": ["enterprise"],
      "createdAt": "2026-02-01T09:00:00Z"
    }
  ],
  "meta": {
    "total": 45,
    "page": 1,
    "limit": 25,
    "totalPages": 2
  }
}
```

- `contact.name` = `first_name + ' ' + last_name`.
- `amount` is a decimal number (formatting is frontend concern).
- `expectedCloseDate`: date only `YYYY-MM-DD` or `null`.
- `company`, `contact`, `owner`: `null` if not associated (keys always present).

**Table Columns (default):** Title, Stage, Amount, Company, Contact, Expected Close, Owner, Probability.
**Additional:** Currency, Tags, Created At.

#### Pipeline/Kanban View

**Columns from `deal_stages` table, ordered by `display_order`.**

Frontend fetches all open deals (stage where `is_won = false` AND `is_lost = false`) with limit=500 and groups client-side by stage. For large datasets, fetch per-stage.

**Kanban Layout:**
- Horizontal scrollable container of columns.
- Each column: header (stage name + deal count + total value), vertically stacked cards ordered by `expected_close_date ASC` then `created_at ASC`, "+ Add Deal" link at bottom.

**Deal Card:**
- Title (bold, 1-line truncated), Company name (muted), Contact name (muted), Amount (formatted), Expected close date (red if past due, yellow if within 7 days), Owner avatar (bottom-right).
- Click card → `/deals/:id`.

**Drag-and-Drop:**
- Drag card between columns to change stage.
- On drop → `PUT /api/v1/deals/:id` with `{ stage: newStageName, updatedAt: ... }`.
- **If target stage has `is_won = true`:** show modal for `actualCloseDate` (default today). User confirms → save. Auto-set probability to stage default.
- **If target stage has `is_lost = true`:** show modal for `actualCloseDate` (default today) + `lostReason` (required). Cancel → revert card.
- On 409: toast "This deal was modified. Refreshing...", re-fetch.
- Animate card movement (200ms).

**Pipeline Summary Bar:**
- Above kanban: Total Pipeline Value | Open Deals: N | Avg Deal Size | Weighted Pipeline (sum of amount * probability / 100).

**Responsive:** Below 768px, kanban view switches to a single-column list grouped by stage with collapsible stage headers. Table view uses card layout like contacts. Deal detail pipeline visual becomes a vertical stepper instead of horizontal.

### Pagination

Same as Contacts: `page` (default 1), `limit` (default 25, max 100).

### Sorting

| Sort Value              | Database Column / Expression            | Notes                          |
|-------------------------|-----------------------------------------|--------------------------------|
| `title`                 | `deals.title`                           | Case-insensitive               |
| `amount`                | `deals.amount`                          | NULLs last                     |
| `stage`                 | `deal_stages.display_order` (via JOIN)  | Ordinal from deal_stages       |
| `expected_close_date`   | `deals.expected_close_date`             | NULLs last                     |
| `company.name`          | `companies.name` (LEFT JOIN)            | Case-insensitive; NULLs last   |
| `contact.last_name`     | `contacts.last_name` (LEFT JOIN)        | Case-insensitive; NULLs last   |
| `owner.last_name`       | `users.last_name` (LEFT JOIN)           | Case-insensitive; NULLs last   |
| `created_at`            | `deals.created_at`                      | Timestamp                      |
| `probability`           | `deals.probability`                     | NULLs last                     |

**Default sort:** `created_at DESC`. **Secondary sort:** `deals.id ASC`.

### Filtering

| Query Parameter          | Type              | Matching Logic                                 | Validation                               |
|--------------------------|-------------------|-------------------------------------------------|------------------------------------------|
| `search`                 | string            | `ILIKE '%val%'` across `title`, `description`. Escape `%`, `_`. | Same rules as contacts.      |
| `stage`                  | string (CSV)      | `stage IN (?)` — comma-separated               | Each must be valid stage name from `deal_stages`. |
| `owner_id`               | UUID              | Exact match                                    | Valid UUID.                              |
| `company_id`             | UUID              | Exact match                                    | Valid UUID.                              |
| `contact_id`             | UUID              | Exact match                                    | Valid UUID.                              |
| `amount_min`             | number            | `amount >= ?`                                  | Non-negative.                            |
| `amount_max`             | number            | `amount <= ?`                                  | Non-negative.                            |
| `expected_close_after`   | ISO date          | `expected_close_date >= ?`                     | Valid date.                              |
| `expected_close_before`  | ISO date          | `expected_close_date <= ?`                     | Valid date.                              |
| `probability_min`        | integer           | `probability >= ?`                             | 0-100.                                   |
| `probability_max`        | integer           | `probability <= ?`                             | 0-100.                                   |
| `tags`                   | string (CSV)      | `tags @> ARRAY[?]`                             | Same as contacts.                        |

All filters AND logic.

---

## Deal Detail View — GET /api/v1/deals/:id

### API Response Shape

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "Acme Corp Enterprise Deal",
    "stage": "proposal",
    "amount": 50000.00,
    "currency": "USD",
    "probability": 60,
    "expectedCloseDate": "2026-06-15",
    "actualCloseDate": null,
    "lostReason": null,
    "description": "Enterprise license deal for 500 seats...",
    "company": {
      "id": "uuid",
      "name": "Acme Corp",
      "domain": "acme.com"
    },
    "contact": {
      "id": "uuid",
      "firstName": "Jane",
      "lastName": "Doe",
      "email": "jane@acme.com"
    },
    "owner": {
      "id": "uuid",
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@crm.com",
      "avatarUrl": null
    },
    "tags": ["enterprise"],
    "customFields": {},
    "createdAt": "2026-02-01T09:00:00Z",
    "updatedAt": "2026-03-20T14:00:00Z"
  }
}
```

### Overview Section (Frontend)

**Header:**
- Deal title (large, editable inline).
- Stage visual pipeline indicator: horizontal bar of connected segments from `deal_stages` ordered by `display_order`. Current stage highlighted, past stages completed, future stages hollow.
- Click any segment to change stage (same confirmation flows for won/lost stages).

**Left Column:**
- **Amount + Currency:** Formatted (e.g., "$50,000.00 USD"). Inline editable.
- **Probability:** Percentage with progress bar. Inline editable (0-100).
- **Expected Close Date:** Formatted. Red if past and deal still open. Inline editable.
- **Description:** Markdown rendered. Inline editable textarea.
- **Tags / Custom Fields:** Same as contacts.

**Right Column — Sidebar:**
- **Company Link:** Clickable → `/companies/:id`. Associate/disassociate.
- **Contact Link:** Clickable → `/contacts/:id`. Associate/disassociate.
- **Owner:** Searchable dropdown assignment.
- **Key Dates:** Created, Last Updated, Expected Close, Actual Close (only if closed).

**Won Deal Display:**
- `is_won = true` stage: Green banner "Deal Won" with actual close date. Stage pipeline all green.

**Lost Deal Display:**
- `is_lost = true` stage: Red banner "Deal Lost" with actual close date. Lost Reason displayed below.

Closed deals remain editable for corrections.

### Stage Change Behavior

**When stage changes to a stage with `is_won = true`:**
1. Modal: "Mark deal as Won?" with Actual Close Date (default today, required).
2. On confirm: `PUT /api/v1/deals/:id` with `{ stage: newStage, actualCloseDate: '2026-03-25', updatedAt: '...' }`.
3. Server auto-sets probability to stage's default probability from `deal_stages`.

**When stage changes to a stage with `is_lost = true`:**
1. Modal: "Mark deal as Lost?" with Actual Close Date (default today, required) + Lost Reason (required, 1-1000 chars).
2. On confirm: `PUT /api/v1/deals/:id` with `{ stage: newStage, actualCloseDate, lostReason, updatedAt }`.

**When stage changes to any other stage:**
- Immediate update, no modal.
- If reopening from closed: clear `actualCloseDate` and `lostReason` (set to null).

**Auto-Update Probability:**
- Each row in `deal_stages` has a `default_probability` column.
- When stage changes: if the deal's current probability matches the old stage's `default_probability`, update to the new stage's `default_probability`. If user manually overrode probability, preserve it.
- This logic is server-side.

**Activity Log Entry:**
- On every stage change, auto-create activity: `type: 'note'`, `subject: 'Stage changed from {oldStage} to {newStage}'`, `dealId: this deal`, `ownerId: user who changed`, `isCompleted: true`.

### Related Records Tabs

Tabs: Activities | Notes | Attachments.

- **Activities**: Filtered to `deal_id = :dealId`. Timeline. "Log Activity" with `dealId` pre-filled.
- **Notes**: Notes for entity_type 'deal'. Pin, add, edit, delete.
- **Attachments**: Attachments for entity_type 'deal'. Upload, download, delete.

---

## Create Deal — POST /api/v1/deals

### Request Body

```json
{
  "title": "Acme Corp Enterprise Deal",
  "companyId": "uuid",
  "contactId": "uuid",
  "ownerId": "uuid",
  "stage": "discovery",
  "amount": 50000.00,
  "currency": "USD",
  "probability": 10,
  "expectedCloseDate": "2026-06-15",
  "description": "Enterprise license deal...",
  "tags": ["enterprise"],
  "customFields": {}
}
```

### Validation Rules (Zod schema in `shared/`)

| Field              | Required | Type          | Constraints                                                                | Default                       |
|--------------------|----------|---------------|----------------------------------------------------------------------------|-------------------------------|
| `title`            | Yes      | string        | 1-255 chars. Trim. Non-empty after trim.                                   | —                             |
| `companyId`        | No       | UUID          | Must exist in companies table.                                             | `null`                        |
| `contactId`        | No       | UUID          | Must exist in contacts table.                                              | `null`                        |
| `ownerId`          | No       | UUID          | Must reference existing active user.                                       | Current user.                 |

**Frontend entity selector requirements for Deal form:**
- **Company** (`companyId`) — `EntitySelector` for companies. On selection, the Contact selector below filters to contacts at this company.
- **Contact** (`contactId`) — `EntitySelector` for contacts. If company is already selected, filters to `?company_id={companyId}`. If contact is selected first and they have a `companyId`, auto-populate the Company field.
- **Owner** (`ownerId`) — `EntitySelector` for users.
| `stage`            | No       | string        | Must be a valid stage name from `deal_stages` table. Validated at app level.| First stage by `display_order`.|
| `amount`           | No       | number        | Positive decimal. Max 2 decimal places. Max 999,999,999,999.99. `0` not allowed (use null). | `null` |
| `currency`         | No       | string        | 3-char ISO 4217 code. Uppercase.                                           | System default (USD).         |
| `probability`      | No       | integer       | 0-100 inclusive.                                                           | `default_probability` for selected stage from `deal_stages`. |
| `expectedCloseDate`| No       | string (date) | `YYYY-MM-DD`. Must be today or future on create. Past dates → 400.         | `null`                        |
| `description`      | No       | string (text) | Max 10,000 chars. Trim.                                                    | `null`                        |
| `tags`             | No       | array         | Same rules as contacts.                                                    | `[]`                          |
| `customFields`     | No       | object        | Same rules as contacts.                                                    | `{}`                          |

**Cross-field:**
- No validation that contact belongs to company.
- If stage is won/lost on creation: `actualCloseDate` auto-set to today. If lost, `lostReason` optional on create.

### Process Flow

1. **Validate** via Zod. Return all errors.
2. **Set defaults:** stage → first in `deal_stages` by `display_order`, probability → stage default, currency → system default, ownerId → current user.
3. **Insert** deal record.
4. **Create initial activity:** `subject: 'Deal created'`, `type: 'note'`, `dealId: new id`, `isCompleted: true`.
5. **Audit log:** action `create`, entity_type `deal`.
6. **Return** HTTP 201 with canonical response format, full deal detail.

---

## Update Deal — PUT /api/v1/deals/:id

### Behavior

Partial update + optimistic locking. `updatedAt` required in body. Server compares with DB `updated_at`. Mismatch → 409.

### Special Stage Change Logic

When `stage` is in the update payload:

1. Fetch current stage from DB.
2. If old === new: skip stage logic.
3. **New stage `is_won = true`:** `actualCloseDate` provided or default today. Clear `lostReason` to null.
4. **New stage `is_lost = true`:** `actualCloseDate` provided or default today. `lostReason` should be provided (warn if not, don't block).
5. **New stage is open (not won/lost):** If reopening from closed, set `actualCloseDate = null`, `lostReason = null`.
6. **Auto-update probability** per stage default rule.
7. **Create stage change activity:** `subject: 'Stage changed from {old} to {new}'`.

### Additional Update Rules

- `expectedCloseDate`: past dates ARE allowed on update (historical corrections).
- `amount`: 0 IS allowed on update.

### Response

HTTP 200 with canonical response format, full deal detail.

---

## Delete Deal — DELETE /api/v1/deals/:id

### Authorization

Admin: any deal. Regular users: only deals they own. Otherwise → 403.

### Cascade Behavior (Hard Delete)

All in transaction (except R2):

1. Delete activities where `deal_id = :id` (CASCADE).
2. Delete notes where `entity_type = 'deal' AND entity_id = :id` (CASCADE).
3. Delete attachments where `entity_type = 'deal' AND entity_id = :id` (R2 + DB).
4. Delete the deal record.
5. Audit log with snapshot.

### Response

HTTP 200:

```json
{
  "success": true,
  "data": {
    "deleted": {
      "activities": 10,
      "notes": 2,
      "attachments": 1
    }
  }
}
```

### Frontend Confirmation

Same pattern: related counts, confirmation modal, name-typing for > 10 related records.

---

## Bulk Operations

### POST /api/v1/deals/bulk-delete

**Request:** `{ "ids": ["uuid-1", ...] }` — max 100. Same cascade per deal.

**Response:** `{ "success": true, "data": { "affected": N } }`

### POST /api/v1/deals/bulk-update

**Request:**
```json
{
  "ids": ["uuid-1", "uuid-2"],
  "updates": {
    "stage": "negotiation",
    "ownerId": "uuid"
  }
}
```

- Max 100 IDs. Allowed updates: `stage`, `ownerId`.
- Stage changes follow the same stage change logic (including activity log entries). Won/lost stages in bulk update are not allowed (must be done individually due to required `actualCloseDate`/`lostReason`).
- Audit log per record.

**Response:** `{ "success": true, "data": { "affected": N } }`

---

## Deal Statistics — GET /api/v1/deals/stats

### Authorization

All authenticated users.

### Query Parameters

| Parameter  | Type   | Required | Constraints                                        | Default    |
|------------|--------|----------|----------------------------------------------------|------------|
| `owner_id` | UUID   | No       | Filter stats to specific user's deals.             | All users. |
| `period`   | string | No       | `this_month`, `this_quarter`, `this_year`, `all_time` | `all_time` |

### Response

```json
{
  "success": true,
  "data": {
    "pipeline": {
      "totalValue": 1250000.00,
      "dealCount": 45,
      "weightedValue": 562500.00
    },
    "byStage": [
      {
        "stage": "discovery",
        "count": 10,
        "totalValue": 200000.00,
        "avgValue": 20000.00
      }
    ],
    "winRate": 70.0,
    "averageDealSize": 60000.00,
    "averageTimeToClose": 45,
    "closingThisMonth": {
      "count": 8,
      "totalValue": 320000.00,
      "deals": [
        {
          "id": "uuid",
          "title": "Quick Deal",
          "amount": 40000.00,
          "expectedCloseDate": "2026-03-28",
          "stage": "negotiation",
          "company": { "id": "uuid", "name": "Acme" }
        }
      ]
    }
  }
}
```

### Metric Definitions

| Metric                   | Calculation                                                                                    |
|--------------------------|-----------------------------------------------------------------------------------------------|
| `pipeline.totalValue`    | `SUM(amount)` for deals where stage `is_won = false` AND `is_lost = false`. NULL amounts → 0. |
| `pipeline.dealCount`     | `COUNT(*)` for open deals.                                                                    |
| `pipeline.weightedValue` | `SUM(amount * probability / 100)` for open deals.                                             |
| `byStage[].count`        | `COUNT(*)` grouped by stage. Includes ALL stages from `deal_stages` (even with 0 deals).      |
| `byStage[].totalValue`   | `SUM(amount)` per stage.                                                                      |
| `byStage[].avgValue`     | `AVG(amount)` per stage (non-null only). `null` if none.                                      |
| `winRate`                | `won / (won + lost) * 100`. If denominator 0: `null`. 1 decimal.                             |
| `averageDealSize`        | `AVG(amount)` for won deals (non-null). `null` if none.                                       |
| `averageTimeToClose`     | `AVG(actual_close_date - created_at)` in days for won deals. Integer. `null` if none.         |
| `closingThisMonth`       | Deals with `expected_close_date` in current month AND open stage. Max 20, sorted by date ASC. |

**Period Scoping:**
- `this_month/quarter/year`: scopes `created_at` for pipeline/byStage, `actual_close_date` for win rate/avg size/avg time.
- `closingThisMonth` is always current month regardless of `period`.

---

---

# SECTION 6e: Core Features — Activities Module

> Same canonical format conventions. All endpoints under `/api/v1/`. Activity types are loaded from the `activity_types` table (not hardcoded). Type validation is at the application level against `activity_types` rows. Validation via Zod in `shared/`.

## Frontend Routes

| Route         | View                     |
|---------------|--------------------------|
| `/activities` | Activity feed            |
| `/tasks`      | Task dashboard (My Tasks)|

---

## Activity Types

Activity types are stored in the `activity_types` table with columns: `id`, `name` (unique key, e.g. `call`), `label` (display name, e.g. "Phone Call"), `icon`, `color`, `display_order`, `created_at`.

Default types seeded:

| Name       | Label          | Icon           | Color   | Description                                          |
|------------|----------------|----------------|---------|------------------------------------------------------|
| `call`     | Phone Call     | `phone`        | `blue`  | Logged phone call with optional duration and outcome. |
| `email`    | Email          | `mail`         | `purple`| Logged email sent or received (not actual sending).   |
| `meeting`  | Meeting        | `calendar`     | `green` | Meeting event.                                        |
| `note`     | Note           | `file-text`    | `yellow`| Quick note or system-generated note.                  |
| `task`     | Task           | `check-square` | `orange`| Action item with optional due date and completion.    |
| `lunch`    | Lunch Meeting  | `utensils`     | `pink`  | Casual lunch meeting.                                 |
| `deadline` | Deadline       | `clock`        | `red`   | Important deadline marker.                            |

---

## Activity List/Feed — GET /api/v1/activities

### Two Usage Modes

1. **Global Activity Feed:** No entity filter params. Returns all activities. Used on dashboard and `/activities`.
2. **Entity Activity Feed:** Filtered by `contact_id`, `company_id`, or `deal_id`. Used on detail views.

### Pagination

**Two modes:**

#### Cursor-Based (for infinite scroll)

| Parameter | Type    | Default | Description                                             |
|-----------|---------|---------|---------------------------------------------------------|
| `cursor`  | string  | —       | Opaque cursor (base64 `{id}:{created_at}`). Omit for first page. |
| `limit`   | integer | `20`    | Min 1, max 100.                                        |

Response:

```json
{
  "success": true,
  "data": [ ... ],
  "meta": {
    "hasMore": true,
    "nextCursor": "eyJpZCI6...",
    "limit": 20
  }
}
```

Query: `WHERE (created_at, id) < (:cursorCreatedAt, :cursorId) ORDER BY created_at DESC, id DESC LIMIT :limit`.

#### Page-Based (for traditional pagination)

| Parameter | Type    | Default | Description       |
|-----------|---------|---------|-------------------|
| `page`    | integer | `1`     | Page number.      |
| `limit`   | integer | `20`    | Min 1, max 100.   |

Response meta: `{ total, page, limit, totalPages }`.

**Mode Selection:** If `cursor` is present → cursor mode. Otherwise → page mode. Do not mix.

### Sorting

| Parameter | Type   | Default      | Description       |
|-----------|--------|--------------|-------------------|
| `sort`    | string | `created_at` | Sort field.       |
| `order`   | string | `desc`       | `asc` or `desc`.  |

**Sortable Fields:**

| Sort Value    | Column                    | Notes                |
|---------------|---------------------------|----------------------|
| `created_at`  | `activities.created_at`   | Default              |
| `due_date`    | `activities.due_date`     | NULLs last           |
| `type`        | `activities.type`         | Alphabetical         |
| `subject`     | `activities.subject`      | Case-insensitive     |

### Filtering

| Query Parameter    | Type              | Matching Logic                            | Validation                                  |
|--------------------|-------------------|-------------------------------------------|---------------------------------------------|
| `type`             | string (CSV)      | `type IN (?)` — comma-separated           | Each must be valid type from `activity_types` table. |
| `owner_id`         | UUID              | Exact match                               | Valid UUID.                                 |
| `contact_id`       | UUID              | Exact match: `contact_id = ?`             | Valid UUID.                                 |
| `company_id`       | UUID              | See special behavior below                | Valid UUID.                                 |
| `deal_id`          | UUID              | Exact match: `deal_id = ?`                | Valid UUID.                                 |
| `is_completed`     | string            | `true` → `is_completed = true`; `false` → `is_completed = false` | `true` or `false`. |
| `due_after`        | ISO date/datetime | `due_date >= ?`                           | Valid ISO date.                             |
| `due_before`       | ISO date/datetime | `due_date <= ?`                           | Valid ISO date.                             |
| `created_after`    | ISO date/datetime | `created_at >= ?`                         | Valid ISO date.                             |
| `created_before`   | ISO date/datetime | `created_at <= ?`                         | Valid ISO date.                             |

All filters AND logic.

**Special `company_id` behavior:** Returns activities where `company_id = :id` OR `contact_id IN (SELECT id FROM contacts WHERE company_id = :id)`. This ensures company detail views show all activities.

**Overdue definition:** `due_date < NOW() AND is_completed = false`.

### List Response Shape

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "type": "call",
      "subject": "Discovery Call with Jane",
      "description": "Discussed requirements for Q3 rollout...",
      "contact": {
        "id": "uuid",
        "name": "Jane Doe"
      },
      "company": {
        "id": "uuid",
        "name": "Acme Corp"
      },
      "deal": {
        "id": "uuid",
        "title": "Acme Enterprise Deal"
      },
      "owner": {
        "id": "uuid",
        "name": "John Smith"
      },
      "dueDate": null,
      "durationMinutes": 45,
      "outcome": "connected",
      "isCompleted": true,
      "completedAt": "2026-03-20T15:15:00Z",
      "createdAt": "2026-03-20T14:30:00Z",
      "updatedAt": "2026-03-20T15:15:00Z"
    }
  ],
  "meta": { "..." : "..." }
}
```

- `contact`, `company`, `deal`: `null` if not associated. Keys always present.
- `dueDate`: ISO datetime or `null`.
- `durationMinutes`: integer or `null`.
- `outcome`: string or `null`.
- `completedAt`: ISO datetime or `null`.

### Frontend

**Timeline View:**
- Vertical timeline with connecting line. Grouped by date: "Today", "Yesterday", etc. Sticky date headers.

**Filter Bar:**
- Type: horizontal pill buttons per activity type + "All". Multiple selectable.
- Date range pickers. Owner dropdown. Active filter chips.

**"Log Activity" Button:**
- Primary button at top. Opens modal: type selector (icon grid) → type-specific form → submit.

**Visual Styling:**
- Overdue: red left border, "OVERDUE" badge, red due date.
- Completed: green check, muted text.
- Today: highlighted background.
- Hover: action menu (edit, delete, mark complete/incomplete).

**Infinite Scroll:** 20 initially. Trigger at 200px. "You've reached the end" when done. "Back to top" after 3+ pages.

**Responsive:** Activity timeline is already vertical and works well on mobile. Task dashboard sections stack vertically. Log Activity modal becomes full-screen on mobile.

---

## Create Activity — POST /api/v1/activities

### Request Body

```json
{
  "type": "call",
  "subject": "Discovery Call with Jane",
  "description": "Discussed requirements for Q3 rollout.",
  "contactId": "uuid",
  "companyId": "uuid",
  "dealId": "uuid",
  "ownerId": "uuid",
  "dueDate": "2026-03-28T10:00:00Z",
  "durationMinutes": 45,
  "outcome": "connected",
  "isCompleted": false
}
```

### Validation Rules (Zod schema in `shared/`)

| Field             | Required | Type          | Constraints                                                                | Default        |
|-------------------|----------|---------------|----------------------------------------------------------------------------|----------------|
| `type`            | Yes      | string        | Must be a valid type from `activity_types` table. Validated at app level.  | —              |
| `subject`         | Yes      | string        | 1-255 chars. Trim. Non-empty after trim.                                   | —              |
| `description`     | No       | string (text) | Max 10,000 chars. Trim. Empty string → null.                               | `null`         |
| `contactId`       | No       | UUID          | Must exist in contacts table if provided.                                  | `null`         |
| `companyId`       | No       | UUID          | Must exist in companies table if provided.                                 | `null`         |
| `dealId`          | No       | UUID          | Must exist in deals table if provided.                                     | `null`         |
| `ownerId`         | No       | UUID          | Must be existing active user.                                              | Current user.  |
| `dueDate`         | No       | string        | ISO 8601 datetime. Past/future both allowed.                               | `null`         |
| `durationMinutes` | No       | integer       | Positive (> 0). Max 1440 (24 hours).                                       | `null`         |
| `outcome`         | No       | string        | Max 500 chars. Trim. Free text.                                            | `null`         |
| `isCompleted`     | No       | boolean       |                                                                            | `false`        |

**Entity Association Warning:** If none of `contactId`, `companyId`, `dealId` provided: include `warning` in response. Do NOT block.

**Auto-set companyId:** If `contactId` provided and `companyId` not: look up contact's `company_id` and auto-set.

**Frontend entity selector requirements for Activity (Log Activity) form:**
- **Contact** — `EntitySelector` for contacts. On selection, if contact has a company, auto-populate Company field.
- **Company** — `EntitySelector` for companies. If contact already selected and has a company, this is auto-populated (but editable). On selection, filters Deal selector to `?company_id=`.
- **Deal** — `EntitySelector` for deals. If company is set, filters to `?company_id=`. On selection, auto-populate Company and Contact from the deal if not already set.
- At least one entity should be linked (show warning toast if none selected, but do not block submission). Users must NEVER see or type raw UUIDs.

### Side Effects

1. **Update `last_contacted_at` on contact:**
   - If `contactId` provided AND `type` is `call`, `email`, `meeting`, or `lunch`:
     - Set `contacts.last_contacted_at = NOW()`.
     - Do NOT update for `note`, `task`, `deadline` (internal, not contact interactions).
     - Only update if NOW() is more recent than current `last_contacted_at`.

2. **Audit log:** action `create`, entity_type `activity`.

### Response

HTTP 201:

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "type": "call",
    "subject": "Discovery Call with Jane",
    "description": "Discussed requirements...",
    "contact": { "id": "uuid", "name": "Jane Doe" },
    "company": { "id": "uuid", "name": "Acme Corp" },
    "deal": null,
    "owner": { "id": "uuid", "name": "John Smith" },
    "dueDate": null,
    "durationMinutes": 45,
    "outcome": "connected",
    "isCompleted": false,
    "completedAt": null,
    "createdAt": "2026-03-25T14:30:00Z",
    "updatedAt": "2026-03-25T14:30:00Z"
  },
  "warning": null
}
```

---

## Update Activity — PUT /api/v1/activities/:id

### Behavior

- Partial update. Only included fields changed.
- `updatedAt` required for optimistic locking. Mismatch → 409.

### Completion Logic

**Marking complete (`isCompleted: true` when previously `false`):**
- Set `is_completed = true`, `completed_at = NOW()`.
- `completed_at` is auto-set — ignore if provided in body.

**Unmarking (`isCompleted: false` when previously `true`):**
- Set `is_completed = false`, `completed_at = NULL`.

### Side Effects

- If `contactId` changes: update `last_contacted_at` on the NEW contact (if applicable type). Do NOT revert old contact.
- Audit log with before/after for changed fields.

### Validation

Same as create, all fields optional.

### Response

HTTP 200 with canonical response format, full activity detail.

---

## Delete Activity — DELETE /api/v1/activities/:id

### Authorization

- Admin: any activity.
- Regular users: only activities they own (`owner_id = current_user.id`). Otherwise → 403.

### Behavior

1. Delete the activity record (hard delete).
2. Do NOT revert `last_contacted_at` on associated contact.
3. Audit log with snapshot.

**No bulk operations for activities.**

### Response

HTTP 200:

```json
{
  "success": true,
  "data": null
}
```

---

## Task Dashboard

### Description

Dedicated view for activities where `type = 'task'`. Accessible from main navigation as "Tasks". Frontend route: `/tasks`.

### API Calls

**My Tasks:** `GET /api/v1/activities?type=task&owner_id=:currentUserId&sort=due_date&order=asc`

Additional filtering per section:
- **Overdue:** `&is_completed=false&due_before={now}` (where `due_date < NOW() AND is_completed = false`)
- **Due Today:** `&is_completed=false&due_after={todayStart}&due_before={todayEnd}`
- **Upcoming:** `&is_completed=false&due_after={todayEnd}`
- **Completed:** `&is_completed=true` (sort by `completed_at desc`)

Alternatively: fetch all tasks at once and group client-side.

### Frontend Layout

**Header:** "My Tasks" title. Tab pills: All / Overdue / Today / Upcoming / Completed.

#### Overdue Section
- Red header: "Overdue ({count})" with warning icon.
- Sorted by due_date ASC (most overdue first).
- Each row: checkbox (mark complete) + subject (bold) + associated entity (muted) + due date (red, e.g., "3 days overdue") + owner avatar.
- Checkbox immediately calls `PUT /api/v1/activities/:id` with `{ isCompleted: true, updatedAt: ... }`. Undo toast for 5 seconds.
- Row click (not checkbox): expand inline or navigate to detail.

#### Due Today Section
- Amber header: "Due Today ({count})". Due date shown as "Due today".

#### Upcoming Section
- Default header: "Upcoming ({count})". Sorted by due_date ASC.
- Sub-grouped: "Tomorrow", "This Week", "Next Week", "Later".
- Tasks with no due date under "No Due Date" at bottom.

#### Completed Section
- Green header: "Completed ({count})". Collapsed by default. Last 30 days. Sorted by completed_at DESC.
- Strikethrough subject, green check. "Uncheck" to re-open.

**"New Task" Button:**
- Top-right, primary. Opens modal: `type: 'task'` pre-selected. Fields: Subject (required), Description, Due Date (default today), Contact/Company/Deal (searchable dropdowns), Assign To (default self).

**Empty States:**
- Overdue section empty: hidden entirely.
- Due Today empty: "No tasks due today".
- Upcoming empty: "No upcoming tasks. Create one?" with CTA.
- Completed empty: "No completed tasks yet."
- Entire dashboard empty: illustration + "No tasks yet. Create your first task." with CTA.

**Notifications / Badge:**
- "Tasks" nav item shows red badge with overdue count.
- Count from: `GET /api/v1/activities?type=task&owner_id=:me&is_completed=false&due_before={now}&limit=1` — use `meta.total`.
- Refreshes every 60 seconds via polling.
- Count 0: no badge.
# SECTION 6f: Core Features — Meetings Module

## Frontend Routes

- `/meetings` — Meeting list (table + calendar views)
- `/meetings/new` — Create meeting form
- `/meetings/:id` — Meeting detail page

---

## Meeting List (GET /api/v1/meetings)

### Views

1. **List View**: Table with sorting/filtering/pagination.
2. **Calendar View**: Simple month grid only (no week or day views). No external calendar library. See "Frontend Calendar" below.

### Sorting & Filtering

- **Sortable columns**: title, startTime, endTime, contact.lastName, company.name, status, owner.lastName
- **Filters**:
  - `?search=` — partial match on title, description, location (case-insensitive, ILIKE)
  - `?status=` — exact match; comma-separated for multi-value (e.g., `?status=scheduled,confirmed`)
  - `?ownerId=` — UUID of owning user
  - `?contactId=` — UUID of linked contact
  - `?companyId=` — UUID of linked company
  - `?dealId=` — UUID of linked deal
  - `?startAfter=` — ISO 8601 datetime; return meetings with start_time >= value
  - `?startBefore=` — ISO 8601 datetime; return meetings with start_time <= value
- **Default behavior**: When no filters are provided, return upcoming meetings only (start_time >= NOW()), sorted by startTime ASC.

### Pagination

- `?page=1&limit=25`
- Max limit: 100
- Default limit: 25
- Response includes `meta.pagination` per Section 4 canonical format.

### Response Format

All responses follow the canonical format defined in Section 4:

```json
{
  "success": true,
  "data": [ ... ],
  "meta": {
    "pagination": { "total": 50, "page": 1, "limit": 25, "totalPages": 2 }
  }
}
```

### Frontend Calendar — Month View

- Render a CSS grid: 7 columns (Sun–Sat) x 5–6 rows. No external calendar library.
- Each cell represents one day.
- Days with meetings display a colored dot indicator (one dot per meeting, max 3 visible; if more, show "+N more" text).
- Dot color is determined by meeting status:
  - `scheduled` → blue
  - `confirmed` → green
  - `completed` → gray
  - `cancelled` → red
  - `no_show` → orange
- Clicking a day cell opens a popover or inline panel listing all meetings for that day (title, time, status badge). Clicking a meeting in this list navigates to the meeting detail page.
- Navigation: left/right arrows to change month; "Today" button to return to current month.
- Current day cell has a highlighted border or background.
- Persist selected view (list vs. calendar) in localStorage key `crm_calendar_view`. Default: Month view.

**Responsive:** Calendar month grid becomes a scrollable list of upcoming meetings on mobile. Meeting detail form stacks to single column.

---

## Meeting Detail (GET /api/v1/meetings/:id)

### Layout

- **Header area**:
  - Title (h1)
  - Status badge (colored pill: scheduled=blue, confirmed=green, completed=gray, cancelled=red, no_show=orange)
  - Edit button (pencil icon) — opens edit modal or inline edit
  - Delete button (trash icon) — opens confirmation dialog

- **Info section** (two-column grid on desktop, single column on mobile):
  - **Start Time**: Displayed in user's timezone, format per settings (e.g., "Mar 25, 2026, 2:00 PM")
  - **End Time**: Same format
  - **Duration**: Calculated display, e.g., "1 hour 30 minutes". Computed as `endTime - startTime`. If duration < 60 min, show only minutes. If >= 60 min, show hours and remaining minutes.
  - **Location**: Text, or if meetingUrl is set, show as clickable link with external link icon
  - **Meeting URL**: Displayed as clickable link; show "Join Meeting" button if URL is present
  - **Owner**: User name (clickable link to user profile if admin, otherwise plain text)

- **Associated Entities** (displayed as clickable pill/chip links):
  - **Contact**: Shows contact full name; links to /contacts/:id
  - **Company**: Shows company name; links to /companies/:id
  - **Deal**: Shows deal title; links to /deals/:id
  - If any association is null, don't display that field at all.

- **Attendees section**:
  - Table or list showing each attendee:
    - Name (user name for internal, contact name for external)
    - Email
    - Type badge: "Internal" (user) or "External" (contact)
    - RSVP Status: accepted (green check), declined (red x), tentative (yellow question mark), pending (gray clock)
  - "Add Attendee" button to add more attendees (autocomplete search for users and contacts)

- **Outcome section**:
  - Displayed below attendees
  - If status is `completed`: show outcome text area (editable inline, auto-save on blur after 1 second debounce)
  - If status is not `completed`: show grayed-out placeholder "Outcome will be available after meeting is completed"
  - Outcome supports plain text only (no markdown)
  - Max length: 5000 characters; show character count

- **Attachments section**:
  - List of attached files (agenda docs, etc.) using the Attachments module
  - Upload button to add new attachments
  - Each attachment shows: filename, size, upload date, download button, delete button (if authorized)

- **Activity Timeline**:
  - Show activities linked to this meeting (created, updated, status changes)
  - Reverse chronological order

---

## Create Meeting (POST /api/v1/meetings)

### Request Body Schema

```json
{
  "title": "string",
  "startTime": "string (ISO 8601)",
  "endTime": "string (ISO 8601)",
  "description": "string | null",
  "location": "string | null",
  "meetingUrl": "string | null",
  "contactId": "string (UUID) | null",
  "companyId": "string (UUID) | null",
  "dealId": "string (UUID) | null",
  "ownerId": "string (UUID)",
  "attendees": [
    {
      "contactId": "string (UUID) | null",
      "userId": "string (UUID) | null",
      "status": "string"
    }
  ],
  "status": "string"
}
```

### Validation (Zod)

All request validation uses Zod schemas. No express-validator.

| Field | Required | Constraints | Error Message |
|-------|----------|-------------|---------------|
| title | Yes | 1–255 characters, trimmed, no leading/trailing whitespace after trim | "Title is required" / "Title must be between 1 and 255 characters" |
| startTime | Yes | Valid ISO 8601 datetime string, must parse to valid Date | "Start time is required" / "Start time must be a valid ISO 8601 datetime" |
| endTime | Yes | Valid ISO 8601 datetime string, must parse to valid Date, MUST be strictly after startTime | "End time is required" / "End time must be a valid ISO 8601 datetime" / "End time must be after start time" |
| description | No | Max 10,000 characters; null or omitted is valid | "Description must not exceed 10,000 characters" |
| location | No | 1–255 characters if provided; null or omitted is valid | "Location must be between 1 and 255 characters" |
| meetingUrl | No | Valid URL format (starts with http:// or https://), max 500 characters; null or omitted is valid | "Meeting URL must be a valid URL" / "Meeting URL must not exceed 500 characters" |
| contactId | No | Valid UUID v4 format; if provided, must reference an existing contact | "Invalid contact ID format" / "Contact not found" |
| companyId | No | Valid UUID v4 format; if provided, must reference an existing company | "Invalid company ID format" / "Company not found" |
| dealId | No | Valid UUID v4 format; if provided, must reference an existing deal | "Invalid deal ID format" / "Deal not found" |
| ownerId | Yes (defaults to current user if omitted) | Valid UUID v4 format; must reference an existing, active user | "Invalid owner ID format" / "Owner not found or inactive" |
| attendees | No | Array; max 50 attendees; each entry must have exactly one of contactId or userId (not both, not neither) | "Each attendee must have either contactId or userId" / "Maximum 50 attendees allowed" |

**Frontend entity selector requirements for Meeting form:**
- **Contact** — `EntitySelector` for contacts.
- **Company** — `EntitySelector` for companies (auto-populated from contact if set).
- **Deal** — `EntitySelector` for deals (filtered by company if set).
- **Attendees** — Multi-select `EntitySelector` supporting both contacts and internal users. Two tabs or a type toggle: 'Contacts' and 'Team Members'. Each attendee shown as a removable chip.
| attendees[].contactId | Conditional | Valid UUID v4; must reference existing contact | "Attendee contact not found" |
| attendees[].userId | Conditional | Valid UUID v4; must reference existing, active user | "Attendee user not found or inactive" |
| attendees[].status | No | One of: 'pending', 'accepted', 'declined', 'tentative'; default 'pending' | "Invalid attendee status" |
| status | No | One of: 'scheduled', 'confirmed', 'completed', 'cancelled', 'no_show'; default 'scheduled' | "Invalid meeting status" |

### Business Rule Validations

- **Duration check**: endTime - startTime must be between 1 minute and 24 hours (1440 minutes). Meetings longer than 24 hours are rejected. Error: "Meeting duration must be between 1 minute and 24 hours"
- **Past meeting creation**: Allowed (for logging past meetings). No validation against current time.
- **Duplicate attendee check**: Same contactId or userId cannot appear twice in attendees array. Error: "Duplicate attendee detected"
- **Owner as attendee**: If ownerId is not already in the attendees array, automatically add them as an attendee with status 'accepted'.

### Side Effects (executed in a database transaction)

1. **Create activity record**:
   - type: 'meeting'
   - subject: "Meeting: {title}"
   - contactId: meeting's contactId
   - companyId: meeting's companyId
   - dealId: meeting's dealId
   - userId: meeting's ownerId
   - meetingId: newly created meeting's ID
   - dueDate: meeting's startTime

2. **Update contact.last_contacted_at**:
   - If contactId is provided AND the meeting's startTime is in the past or now, set contact.last_contacted_at = MAX(current last_contacted_at, meeting.startTime)
   - If the meeting is in the future, do NOT update last_contacted_at (it will be updated when the meeting status changes to 'completed')

3. **Send email notification to attendees** (async, after transaction commits):
   - For each attendee who has an email address:
     - Internal users (userId): look up user.email
     - External contacts (contactId): look up contact.email
   - Use Brevo API to send meeting invitation email
   - Use 'meeting_invitation' email template
   - Template variables: meeting_title, meeting_time (formatted start–end), meeting_location, meeting_url, meeting_description
   - **Rate limit**: max 20 emails per hour per user (meeting notifications count toward this cap)
   - **Graceful degradation**: if Brevo is unavailable or not configured, log warning, create meeting anyway. Email is best-effort.
   - If email sending fails: log error, do NOT roll back the meeting creation.
   - Do not send email to the meeting creator/owner.

4. **Audit log**:
   - action: 'create'
   - entity_type: 'meeting'
   - entity_id: new meeting ID
   - user_id: current user
   - changes: `{ "before": null, "after": { ...full meeting object } }`

### Response

- HTTP 201 Created
- Body follows Section 4 canonical format:

```json
{
  "success": true,
  "data": { /* full meeting object including all relations (contact, company, deal, owner, attendees) populated */ }
}
```

---

## Update Meeting (PUT /api/v1/meetings/:id)

### Request Body

- All fields from Create are optional.
- Only provided fields are updated; omitted fields remain unchanged.
- `updatedAt` field required for optimistic locking.

### Optimistic Locking

- Request must include current `updatedAt` timestamp.
- Server checks: `WHERE id = :id AND updated_at = :updatedAt`
- If mismatch: return 409 CONFLICT:

```json
{
  "success": false,
  "error": {
    "code": "CONFLICT",
    "message": "This meeting has been modified by another user. Please refresh and try again."
  }
}
```

- On successful update: `updated_at` is set to NOW() automatically.

### Validation

- Same Zod validation rules as Create, but all fields optional.
- If startTime is provided without endTime (or vice versa), validate against the existing value in DB.
- Re-validate: endTime must still be after startTime after the update.

### Special Behaviors

1. **Status changed to 'completed'**:
   - Frontend: open a prompt/modal asking for outcome text before saving
   - Backend: if status is changed to 'completed' and outcome is empty, still allow (outcome is optional)
   - Update contact.last_contacted_at to meeting.startTime (if contact linked)
   - Create activity record: "Meeting completed: {title}"

2. **Status changed to 'cancelled'**:
   - Send cancellation email to all attendees (same as delete flow, but meeting is preserved)
   - Create activity record: "Meeting cancelled: {title}"

3. **Time changed** (startTime or endTime different from current values):
   - Send meeting update email to all attendees via Brevo
   - Template: 'meeting_update'
   - Include old time and new time in the email
   - Do not send to the user making the change

4. **Attendees changed**:
   - New attendees added: send meeting invitation email to new attendees only
   - Attendees removed: send cancellation email to removed attendees only
   - Existing attendees: no email

### Side Effects

- Audit log with `changes: { "before": {...}, "after": {...} }` (only changed fields)
- Update linked activity if subject/time changed
- Email notifications as described above (async, best-effort, rate limited to 20/hour/user)

### Response

- HTTP 200 OK
- Body: `{ "success": true, "data": { /* full updated meeting object with all relations populated */ } }`

---

## Delete Meeting (DELETE /api/v1/meetings/:id)

### Authorization

- Meeting owner or admin can delete.
- Other users: return 403 FORBIDDEN.

### Process (hard delete, in a database transaction)

1. Load meeting with all attendees.
2. Delete all meeting_attendees records (cascade).
3. Delete all linked activities where meetingId = this meeting.
4. Delete the meeting record (hard delete).
5. Audit log: action 'delete', entity_type 'meeting', changes: `{ "before": { ...full meeting object }, "after": null }`.

### Post-Delete Side Effects (async, after transaction commits)

- Send cancellation email to all attendees who have email addresses.
- Template: 'meeting_cancellation'
- Variables: meeting_title, meeting_time, cancelled_by (name of user who deleted)
- Do not send to the user who performed the deletion.
- Rate limited to 20 emails/hour/user.
- If Brevo unavailable, log warning, deletion still succeeds.

### Response

- HTTP 200 OK
- Body: `{ "success": true, "data": { "id": "uuid", "deleted": true } }`

### Edge Cases

- Meeting not found: 404 NOT_FOUND
- If email sending fails after deletion: log error, do not affect response (deletion is already committed)

---

---

# SECTION 6g: Core Features — Notes Module

## Notes (always in context of an entity)

Notes are created within a contact, company, or deal detail view. There is **no standalone notes page** in the frontend — notes always appear in entity detail context. The API endpoint `GET /api/v1/notes` with entity filters works for programmatic access and API key consumers.

---

### Note List (GET /api/v1/notes)

#### Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| contactId | UUID | Filter notes for a specific contact |
| companyId | UUID | Filter notes for a specific company |
| dealId | UUID | Filter notes for a specific deal |
| search | string | Full-text search on note content (case-insensitive) |
| createdBy | UUID | Filter by author |
| isPinned | boolean | Filter pinned/unpinned notes |
| page | integer | Page number (default: 1) |
| limit | integer | Items per page (default: 25, max: 100) |
| sort | string | Sort field (default: created_at) |
| order | string | asc or desc (default: desc) |

#### Requirement

- At least one of `contactId`, `companyId`, or `dealId` MUST be provided. If none are provided, return 400:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "At least one of contactId, companyId, or dealId is required"
  }
}
```

- Exception: admin users and API key requests with 'admin' permission may omit entity filters to retrieve all notes.

#### Response

- Section 4 canonical format with `meta.pagination`.
- Each note includes: id, content, isPinned, createdBy (with user name and avatar), contactId, companyId, dealId, createdAt, updatedAt.

---

### Create Note (POST /api/v1/notes)

#### Request Body Schema

```json
{
  "content": "string",
  "contactId": "string (UUID) | null",
  "companyId": "string (UUID) | null",
  "dealId": "string (UUID) | null",
  "isPinned": "boolean"
}
```

#### Validation (Zod)

| Field | Required | Constraints | Error Message |
|-------|----------|-------------|---------------|
| content | Yes | 1–50,000 characters; trimmed; must not be empty after trim | "Content is required" / "Content must not exceed 50,000 characters" |
| contactId | Conditional | Valid UUID v4; must reference existing contact | "Invalid contact ID" / "Contact not found" |
| companyId | Conditional | Valid UUID v4; must reference existing company | "Invalid company ID" / "Company not found" |
| dealId | Conditional | Valid UUID v4; must reference existing deal | "Invalid deal ID" / "Deal not found" |
| isPinned | No | Boolean; default false | "isPinned must be a boolean" |

#### Business Rules

- **At least one entity required**: At least one of contactId, companyId, or dealId must be provided. Error: "At least one of contactId, companyId, or dealId is required"
- **Multiple entities allowed**: A note CAN be linked to multiple entities simultaneously (e.g., a note about a contact within a deal context can have both contactId and dealId).
- **Max pinned notes per entity**: Maximum 10 pinned notes per entity. If adding a pinned note would exceed this limit: return 400 VALIDATION_ERROR with message "Maximum 10 pinned notes per entity"

#### Side Effects

1. **Create activity record**:
   - type: 'note'
   - subject: "Note added" (with first 50 chars of content as description)
   - Linked to same entities as the note
   - userId: current user

2. **Audit log**:
   - action: 'create'
   - entity_type: 'note'
   - entity_id: new note ID
   - changes: `{ "before": null, "after": { content (first 200 chars only for privacy), isPinned, contactId, companyId, dealId } }`

#### Response

- HTTP 201 Created
- Body: `{ "success": true, "data": { /* full note object including createdBy user (name, avatar) */ } }`

---

### Update Note (PUT /api/v1/notes/:id)

#### Authorization

- Only the note creator (createdBy) or a user with role 'admin' can update a note.
- Other users: return 403 FORBIDDEN with message "Only the note creator or an admin can edit this note"

#### Request Body

```json
{
  "content": "string | undefined",
  "isPinned": "boolean | undefined",
  "updatedAt": "string (ISO 8601)"
}
```

#### Optimistic Locking

- Request must include current `updatedAt` timestamp.
- Server checks: `WHERE id = :id AND updated_at = :updatedAt`
- If mismatch: return 409 CONFLICT with error code "CONFLICT" and message "This note has been modified by another user. Please refresh and try again."

#### Validation (Zod)

- content: if provided, same rules as create (1–50,000 chars, non-empty after trim)
- isPinned: if provided, must be boolean
- If changing isPinned from false to true, check max 10 pinned notes per entity limit

#### Side Effects

- Audit log: action 'update', changes: `{ "before": {...}, "after": {...} }` (only changed fields)
- Note's updated_at timestamp is set to NOW()

#### Response

- HTTP 200 OK
- Body: `{ "success": true, "data": { /* full updated note object */ } }`

---

### Delete Note (DELETE /api/v1/notes/:id)

#### Authorization

- Only the note creator (createdBy) or a user with role 'admin' can delete a note.
- Other users: return 403 FORBIDDEN with message "Only the note creator or an admin can delete this note"

#### Process

1. Delete the note record (hard delete).
2. Audit log: action 'delete', entity_type 'note', changes: `{ "before": { ...full note content }, "after": null }`.

#### Response

- HTTP 200 OK
- Body: `{ "success": true, "data": { "id": "uuid", "deleted": true } }`

#### Edge Cases

- Note not found: 404 NOT_FOUND
- Attempting to delete someone else's note as non-admin: 403 FORBIDDEN

---

### Display Rules (Frontend)

#### Within Entity Detail Pages

Notes are displayed in a "Notes" tab or section within Contact, Company, and Deal detail pages. There is no standalone /notes route.

1. **Ordering**:
   - Pinned notes ALWAYS appear first, ordered by updatedAt DESC within the pinned group
   - Non-pinned notes appear after, ordered by createdAt DESC

2. **Note Card Layout**:
   - **Header row**: Author avatar (32x32px circle) + author full name + relative timestamp ("2 hours ago", "Mar 20, 2026")
   - **Pin indicator**: If pinned, show a pin icon in the top-right corner of the note card
   - **Content area**: Rendered markdown content
   - **Action buttons** (shown on hover or via "..." menu, only for authorized users — creator or admin):
     - Edit (pencil icon) — opens inline editor replacing the content area
     - Delete (trash icon) — opens confirmation dialog: "Are you sure you want to delete this note? This action cannot be undone."
     - Pin/Unpin (pin icon) — toggles isPinned; no confirmation needed

3. **Inline Editing**:
   - Clicking Edit replaces the note content with a textarea pre-filled with the raw markdown content
   - Show "Save" and "Cancel" buttons below the textarea
   - Save sends PUT request; on success, re-render the note
   - Cancel reverts to display mode with no changes

4. **New Note Form**:
   - Positioned at the top of the notes section (above the notes list)
   - Textarea with placeholder "Add a note..."
   - "Pin" checkbox
   - "Save" button (disabled when content is empty)
   - After saving, the new note appears at the top of the list (or in pinned section if pinned)

5. **Markdown Rendering**:
   - Use `marked` for markdown parsing + `sanitize-html` for XSS prevention. Do NOT build a custom parser.
   - Supported elements:
     - **Bold**: `**text**` → `<strong>text</strong>`
     - **Italic**: `*text*` → `<em>text</em>`
     - **Links**: `[text](url)` → `<a href="url" target="_blank" rel="noopener noreferrer">text</a>`
     - **Unordered lists**: `- item` → `<ul><li>item</li></ul>`
     - **Ordered lists**: `1. item` → `<ol><li>item</li></ol>`
     - **Code blocks**: triple backticks → `<pre><code>...</code></pre>`
     - **Inline code**: single backticks → `<code>...</code>`
     - **Line breaks**: double newline → paragraph break
   - Sanitize HTML output with `sanitize-html` to prevent XSS before rendering with `dangerouslySetInnerHTML`.

6. **Empty State**:
   - When no notes exist: "No notes yet. Add the first note above."

7. **Pagination / Infinite Scroll**:
   - Load first 20 notes initially
   - "Load more" button at the bottom, or infinite scroll (scroll-based loading)
   - Each load fetches the next 20 notes

---

---

# SECTION 6h: Core Features — Attachments Module (Cloudflare R2)

## R2 Integration

### Configuration

| Environment Variable | Required | Description |
|---------------------|----------|-------------|
| R2_ACCOUNT_ID | Yes | Cloudflare account ID |
| R2_ACCESS_KEY_ID | Yes | R2 API access key ID |
| R2_SECRET_ACCESS_KEY | Yes | R2 API secret access key |
| R2_BUCKET_NAME | Yes | R2 bucket name (e.g., "crm-attachments") |

### R2 Client Setup (AWS SDK v3)

```typescript
// server/src/config/r2.ts
import { S3Client } from '@aws-sdk/client-s3';

const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: R2_ACCESS_KEY_ID,
    secretAccessKey: R2_SECRET_ACCESS_KEY,
  },
});
```

### R2 Key Format

```
attachments/{entity_type}/{entity_id}/{uuid}-{sanitized_filename}
```

- `entity_type`: one of 'contact', 'company', 'deal', 'meeting'
- `entity_id`: UUID of the parent entity
- `uuid`: freshly generated UUID v4 for uniqueness
- `sanitized_filename`: original filename with the following sanitization:
  - Replace spaces with hyphens
  - Remove characters not in: `a-zA-Z0-9._-`
  - Truncate to 100 characters
  - Lowercase
  - Example: `"My Report (Final).PDF"` → `"my-report-final.pdf"`

---

## Upload (POST /api/v1/attachments/upload)

### Primary Approach: Stream Through Backend

The primary upload approach streams files through the backend for strongest access control. The file goes from the client to the Express server, which validates and then uploads to R2. This ensures all access control checks happen before any object reaches R2.

### Request

- Content-Type: `multipart/form-data`
- Fields:
  - `file`: the file (binary)
  - `entityType`: string — one of 'contact', 'company', 'deal', 'meeting'
  - `entityId`: string — UUID of the entity

### Process (step by step)

1. **Parse multipart form data** using multer with memory storage (buffer in memory, NOT written to disk):
   ```typescript
   const upload = multer({
     storage: multer.memoryStorage(),
     limits: { fileSize: 25 * 1024 * 1024 }, // 25 MB
   });
   ```

2. **Validate file presence**: If no file in request, return 400: "File is required"

3. **Validate file size**: multer handles this via limits, but double-check. If file.size > 25 * 1024 * 1024, return 400: "File size must not exceed 25MB"

4. **Validate MIME type via magic bytes**: Use the `file-type` library to detect MIME from the file buffer's magic bytes. Do NOT trust the Content-Type header alone. If the detected type differs from the declared type, use the detected type. If detection fails (e.g., for text files), fall back to declared type.

   **Allowed MIME types** (NO SVG):
   - Images: `image/jpeg`, `image/png`, `image/gif`, `image/webp`
   - Documents: `application/pdf`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
   - Spreadsheets: `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `text/csv`
   - Text: `text/plain`
   - Archives: `application/zip`, `application/x-zip-compressed`

   **SVG is explicitly blocked** — `image/svg+xml` is NOT in the allowed list. SVG files can contain embedded JavaScript and pose XSS risks.

   If MIME type not in allowed list: return 400: "File type '{mimetype}' is not allowed. Allowed types: images (JPEG, PNG, GIF, WebP), PDF, DOC, DOCX, XLS, XLSX, CSV, TXT, ZIP"

5. **Validate entityType**: Must be one of 'contact', 'company', 'deal', 'meeting'. If invalid: return 400: "entityType must be one of: contact, company, deal, meeting"

6. **Validate entityId**: Must be valid UUID v4. If invalid format: return 400: "Invalid entityId format". Query DB to verify entity exists. If not found: return 404: "{entityType} not found"

7. **Check attachment count**: Count existing attachments for this entity. If count >= 50: return 400: "Maximum 50 attachments per entity"

8. **Generate R2 key**:
   ```typescript
   const sanitizedName = sanitizeFilename(file.originalname);
   const key = `attachments/${entityType}/${entityId}/${uuidv4()}-${sanitizedName}`;
   ```

9. **Upload to R2**:
   ```typescript
   await r2Client.send(new PutObjectCommand({
     Bucket: R2_BUCKET_NAME,
     Key: key,
     Body: file.buffer,
     ContentType: detectedMimeType,
     ContentDisposition: `attachment; filename="${file.originalname}"`,
   }));
   ```

10. **Create attachment record in DB**:
    ```sql
    INSERT INTO attachments (id, entity_type, entity_id, filename, original_filename, mime_type, size_bytes, r2_key, uploaded_by, created_at)
    VALUES (uuid, entityType, entityId, sanitizedName, file.originalname, detectedMimeType, file.size, key, currentUserId, NOW())
    ```

11. **Audit log**: action 'create', entity_type 'attachment', entity_id = new attachment ID

12. **Return response** (HTTP 201):
    ```json
    {
      "success": true,
      "data": {
        "id": "uuid",
        "filename": "my-report-final.pdf",
        "originalFilename": "My Report (Final).PDF",
        "mimeType": "application/pdf",
        "sizeBytes": 1048576,
        "downloadUrl": "/api/v1/attachments/{id}/download",
        "entityType": "contact",
        "entityId": "uuid",
        "uploadedBy": { "id": "uuid", "firstName": "John", "lastName": "Doe" },
        "createdAt": "2026-03-25T14:30:00Z"
      }
    }
    ```

### Error Handling

- If R2 upload fails: return 500 "Failed to upload file. Please try again." — do NOT create DB record
- If DB insert fails after R2 upload: attempt to delete the R2 object (best-effort cleanup), return 500
- Rate limit: 10 uploads per minute per user

---

## Download (GET /api/v1/attachments/:id/download)

### Primary Approach: Stream Through Backend

The primary download approach streams the R2 object through the backend, which provides the strongest access control. The backend fetches the object from R2 and pipes it to the client response with appropriate Content-Type and Content-Disposition headers.

### Secondary Approach: Presigned URL

As a secondary option (for large files or to reduce backend load), generate a presigned URL with a **5-minute expiry**:

```typescript
const command = new GetObjectCommand({
  Bucket: R2_BUCKET_NAME,
  Key: attachment.r2Key,
});
const presignedUrl = await getSignedUrl(r2Client, command, { expiresIn: 300 }); // 5 minutes
```

### Process

1. Look up attachment record by ID. If not found: return 404.
2. Verify user has access (user is authenticated — all authenticated users can download any attachment).
3. Default behavior: stream through backend (set Content-Type, Content-Disposition headers).
4. If `?presigned=true`: return JSON `{ "success": true, "data": { "downloadUrl": "https://...", "expiresIn": 300 } }` with a 5-minute presigned URL.

### Edge Cases

- R2 object deleted externally (not through app): return 404 with error "File not found in storage".
- Expired attachment record referencing non-existent R2 object: same handling.

---

## Delete (DELETE /api/v1/attachments/:id)

### Authorization

- Only the uploader (uploaded_by) or a user with role 'admin' can delete an attachment.
- Other users: return 403 FORBIDDEN: "Only the uploader or an admin can delete this attachment"

### Process

1. Look up attachment record. If not found: return 404.
2. Delete the R2 object:
   ```typescript
   await r2Client.send(new DeleteObjectCommand({
     Bucket: R2_BUCKET_NAME,
     Key: attachment.r2Key,
   }));
   ```
3. **If R2 delete fails**: log error, **still delete DB record** (orphaned R2 objects can be cleaned up later by a maintenance job).
4. Delete the DB record (hard delete).
5. Audit log: action 'delete', entity_type 'attachment', changes: `{ "before": { ...attachment metadata }, "after": null }`.

### Response

- HTTP 200 OK
- Body: `{ "success": true, "data": { "id": "uuid", "deleted": true } }`

---

## Cascade Delete (when parent entity is deleted)

When an entity (contact, company, deal, meeting) is deleted and that entity has attachments:

1. **Service must delete R2 objects first**: For each attachment belonging to the entity, call `DeleteObjectCommand` on R2.
2. **If any R2 delete fails**: log the error, but **still proceed** with deleting the DB attachment records and the parent entity.
3. Delete all attachment DB records for the entity.
4. Continue with deleting the parent entity.

This ensures entity deletion is never blocked by R2 failures while still making a best-effort attempt to clean up storage.

---

## List (GET /api/v1/attachments?entityType=contact&entityId=xxx)

### Query Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| entityType | Yes | One of: 'contact', 'company', 'deal', 'meeting' |
| entityId | Yes | UUID of the entity |
| page | No | Page number (default: 1) |
| limit | No | Items per page (default: 25, max: 100) |

### Validation (Zod)

- entityType is required. Error: "entityType is required"
- entityId is required. Error: "entityId is required"
- Entity must exist. Error: "{entityType} not found"

### Response

- Section 4 canonical format with `meta.pagination`.
- Each attachment: id, filename, originalFilename, mimeType, sizeBytes, downloadUrl, uploadedBy (id, firstName, lastName), createdAt
- Sorted by createdAt DESC

---

---

# SECTION 6i: Core Features — Email Integration (Brevo)

## Brevo Integration

### Configuration

| Environment Variable | Required | Description |
|---------------------|----------|-------------|
| BREVO_API_KEY | Yes | Brevo API key (starts with "xkeysib-") |
| EMAIL_FROM_ADDRESS | Yes | Sender email address (must be verified in Brevo) |
| EMAIL_FROM_NAME | No | Sender name (default: value of CRM_NAME setting) |

### Graceful Degradation

- If `BREVO_API_KEY` is not set: log a warning at startup ("BREVO_API_KEY not configured — all email functions will be no-ops"), disable all email functionality, but do NOT prevent the application from starting.
- All email-sending methods must check if Brevo is configured before attempting to send. If not configured, log a warning and return silently.
- Email sending is ALWAYS async and best-effort. Failures must never block or roll back the primary operation.

### Rate Limiting

- **Global limit**: 50 emails per hour across all users and all email types.
- Tracked in-memory or via a simple DB counter.
- When limit reached: log warning, skip email send, log to email_log with status 'skipped' and error_message 'Rate limit exceeded'.

---

## Email Types

### 1. Welcome Email

- **Trigger**: Admin creates a new user via POST /api/v1/users
- **Recipient**: New user's email address
- **Template key**: `welcome`
- **Required variables**:
  - `{{first_name}}` — new user's first name
  - `{{last_name}}` — new user's last name
  - `{{email}}` — new user's email
  - `{{crm_name}}` — CRM name from settings
  - `{{login_url}}` — full URL to the login page
  - `{{set_password_url}}` — link to set initial password
- **Subject line default**: "Welcome to {{crm_name}}"
- **Behavior**: Sent once. If admin creates then immediately deletes user before email is sent, the queued email should still be sent (no recall mechanism).

### 2. Password Reset Email

- **Trigger**: User submits POST /api/v1/auth/forgot-password
- **Recipient**: The user's email address
- **Template key**: `password_reset`
- **Required variables**:
  - `{{first_name}}` — user's first name
  - `{{reset_url}}` — full URL with reset token, valid for 1 hour
  - `{{crm_name}}` — CRM name from settings
  - `{{expiry_time}}` — human-readable expiry ("1 hour")
- **Subject line default**: "Reset your {{crm_name}} password"
- **Security**: Send email even if email not found in DB (to prevent user enumeration). In that case, don't actually send — just return success to the client.
- **Rate limit**: Max 3 password reset emails per email address per hour.

### 3. Meeting Invitation

- **Trigger**: Meeting created with attendees (POST /api/v1/meetings)
- **Recipient**: Each attendee with an email address (excluding the meeting creator)
- **Template key**: `meeting_invitation`
- **Required variables**:
  - `{{attendee_name}}` — recipient's name
  - `{{meeting_title}}` — meeting title
  - `{{meeting_date}}` — formatted date (e.g., "Wednesday, March 25, 2026")
  - `{{meeting_time}}` — formatted time range (e.g., "2:00 PM – 3:00 PM EST")
  - `{{meeting_location}}` — location text, or "Online" if meetingUrl is set
  - `{{meeting_url}}` — meeting URL (if set)
  - `{{meeting_description}}` — meeting description (if set), truncated to 500 chars
  - `{{organizer_name}}` — meeting owner's full name
  - `{{crm_name}}` — CRM name
- **Subject line default**: "Meeting Invitation: {{meeting_title}}"
- **Rate limit**: max 20 emails per hour per user for meeting notifications

### 4. Meeting Update

- **Trigger**: Meeting details changed (PUT /api/v1/meetings/:id) — specifically when startTime, endTime, location, or meetingUrl changes
- **Recipient**: All attendees with email (excluding user who made the update)
- **Template key**: `meeting_update`
- **Required variables**:
  - All variables from meeting_invitation, plus:
  - `{{changes_summary}}` — human-readable list of what changed (e.g., "Time changed from 2:00 PM to 3:00 PM")
- **Subject line default**: "Updated: {{meeting_title}}"

### 5. Meeting Cancellation

- **Trigger**: Meeting deleted (DELETE /api/v1/meetings/:id) OR meeting status changed to 'cancelled'
- **Recipient**: All attendees with email (excluding user who cancelled)
- **Template key**: `meeting_cancellation`
- **Required variables**:
  - `{{attendee_name}}` — recipient's name
  - `{{meeting_title}}` — meeting title
  - `{{meeting_date}}` — original date
  - `{{meeting_time}}` — original time range
  - `{{cancelled_by}}` — name of user who cancelled
  - `{{crm_name}}` — CRM name
- **Subject line default**: "Cancelled: {{meeting_title}}"

---

## Email Template System

### Database Table: `email_templates`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| key | VARCHAR(100) | Unique identifier (e.g., 'welcome', 'password_reset') |
| name | VARCHAR(255) | Human-readable name for admin UI |
| subject | VARCHAR(500) | Email subject line (supports variable substitution) |
| body_html | TEXT | HTML email body (supports variable substitution) |
| body_text | TEXT | Plain text email body (supports variable substitution) |
| variables | JSONB | Array of available variable names for this template (for documentation/UI) |
| is_active | BOOLEAN | Whether this template is currently active (default true) |
| is_system | BOOLEAN | Whether this is a system template (cannot be deleted, only edited) |
| created_at | TIMESTAMP | Creation timestamp |
| updated_at | TIMESTAMP | Last update timestamp |

### Template Lookup

- Templates are looked up by `key` from the `email_templates` table.
- The email service loads the template by key, checks `is_active`, then performs variable substitution.

### Variable Substitution

- Variables use double curly brace syntax: `{{variable_name}}`
- Substitution is performed server-side before sending
- If a variable is referenced in the template but not provided in the data, replace with empty string (do not leave raw `{{...}}` in sent email)
- Variables are case-insensitive during substitution (but stored lowercase by convention)
- No nested variables or logic (no conditionals, no loops) — keep it simple

### Default Templates (Seeded)

On first deploy (or when running seed command), insert the following templates if they don't exist (check by `key`):

1. `welcome` — Welcome Email
2. `password_reset` — Password Reset Email
3. `meeting_invitation` — Meeting Invitation
4. `meeting_update` — Meeting Update
5. `meeting_cancellation` — Meeting Cancellation

All default templates have `is_system = true`. System templates **cannot be deleted** — only edited.

### Admin Template Management (Settings > Email Templates)

- **List view**: Table showing template name, key, subject, is_active, last updated
- **Edit view**:
  - Fields: name, subject, body_html (textarea or simple HTML editor), body_text (textarea)
  - Show available variables for this template (from the `variables` column) as a reference list
  - Click a variable name to insert it at cursor position in the active textarea
  - **Preview button**: Opens a modal showing the rendered email with sample data (both HTML and plain text versions; HTML preview uses an iframe to isolate styles)
  - **Test Send button**: Send a test email to the current admin's email address with sample data
  - Save button: validates subject and body_html are non-empty, then saves
- **Create new template**: Same form as edit, but key must be unique and is set by admin
- **Activate/Deactivate**: Toggle is_active. Deactivated templates are not sent (the email service checks is_active before sending and silently skips if inactive).
- **Delete**: Only non-system templates can be deleted. System templates (is_system=true) show delete button as disabled with tooltip "System templates cannot be deleted".

---

## Email Service Architecture

### File: `server/src/services/email.service.ts`

```typescript
class EmailService {
  private brevoClient: TransactionalEmailsApi | null;
  private isConfigured: boolean;

  constructor() {
    if (!process.env.BREVO_API_KEY) {
      this.isConfigured = false;
      logger.warn('BREVO_API_KEY not configured — all email functions will be no-ops');
      return;
    }
    // Initialize Brevo client
    this.isConfigured = true;
  }

  /**
   * Core send method. All other methods delegate to this.
   * 1. Check if Brevo is configured; if not, log warning and return
   * 2. Check global rate limit (50/hour); if exceeded, log warning and return
   * 3. Load template by key from DB; if not found or inactive, log warning and return
   * 4. Perform variable substitution on subject, body_html, body_text
   * 5. Call Brevo API to send transactional email
   * 6. Log result to email_log table (no body content in logs)
   * 7. If sending fails, log error but do NOT throw
   */
  async sendEmail(to: string, templateKey: string, variables: Record<string, string>): Promise<void>;

  async sendWelcomeEmail(user: User): Promise<void>;
  async sendPasswordResetEmail(user: User, resetUrl: string): Promise<void>;
  async sendMeetingInvitation(meeting: Meeting, attendees: Attendee[]): Promise<void>;
  async sendMeetingUpdate(meeting: Meeting, attendees: Attendee[], changes: string[]): Promise<void>;
  async sendMeetingCancellation(meeting: Meeting, attendees: Attendee[]): Promise<void>;
}
```

### Singleton Pattern

- Export a single instance: `export const emailService = new EmailService();`
- All controllers/services import and use this singleton.

### Email Sending Details (Brevo API Call)

```typescript
const sendSmtpEmail = {
  to: [{ email: recipientEmail, name: recipientName }],
  sender: { email: EMAIL_FROM_ADDRESS, name: EMAIL_FROM_NAME || crmName },
  subject: renderedSubject,
  htmlContent: renderedHtml,
  textContent: renderedText,
  tags: [templateKey], // For Brevo analytics
};
await apiInstance.sendTransacEmail(sendSmtpEmail);
```

---

## Email Logging

### Database Table: `email_log`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| to_email | VARCHAR(255) | Recipient email address |
| template_key | VARCHAR(100) | Template used |
| status | VARCHAR(20) | 'sent', 'failed', 'skipped' |
| error_message | TEXT | Error message if failed (null if sent) |
| contact_id | UUID | FK to contacts (null if not contact-related) |
| user_id | UUID | FK to users — the user who triggered the email |
| created_at | TIMESTAMP | When the email was sent/attempted |

### Rules

- Log EVERY email attempt, including failures and skips.
- Do NOT store the email body or subject in the log (privacy/storage).
- Do NOT store variable values in the log.
- 'skipped' status = template was inactive, Brevo not configured, or rate limit exceeded.
- Create an activity record (type='email') linked to the contact when a contact-related email is sent (meeting invitation to a contact, etc.).

### Retention

- Email logs are kept indefinitely by default.

---

---

# SECTION 6j: Core Features — API Keys & Integrations

## Frontend Route

- `/settings/api-keys` — API key management (admin only)

## API Key Management (Settings > API Keys)

### UI Location

- Settings page → "API Keys" tab (admin only)
- Tab shows: list of existing keys, "Generate New Key" button

---

### Generate API Key (POST /api/v1/api-keys)

#### Authorization

- Admin only. Non-admin users: 403 FORBIDDEN.

#### Request Body

```json
{
  "name": "string",
  "permissions": ["string"],
  "expiresAt": "string (ISO 8601) | null"
}
```

#### Validation (Zod)

| Field | Required | Constraints | Error Message |
|-------|----------|-------------|---------------|
| name | Yes | 1–255 characters, trimmed | "Name is required" / "Name must be between 1 and 255 characters" |
| permissions | Yes | Non-empty array; each element one of: 'read', 'write', 'delete', 'admin' | "At least one permission is required" / "Invalid permission: {value}. Allowed: read, write, delete, admin" |
| expiresAt | No | Valid ISO 8601 datetime, must be in the future; null = never expires | "Expiration date must be in the future" |

#### Process

1. Validate inputs with Zod.
2. Generate random key:
   ```typescript
   const rawKey = crypto.randomBytes(32).toString('hex'); // 64 hex chars
   const fullKey = `crm_${rawKey}`; // Total: 68 chars
   ```
3. Compute HMAC-SHA256 hash with server secret:
   ```typescript
   const keyHash = crypto.createHmac('sha256', process.env.API_KEY_SECRET)
     .update(fullKey)
     .digest('hex');
   ```
4. Extract prefix: `fullKey.substring(0, 8)` → e.g., "crm_a3b2"
5. Insert record into `api_keys` table:
   ```sql
   INSERT INTO api_keys (id, user_id, name, key_hash, key_prefix, permissions, expires_at, is_active, created_at)
   VALUES (uuid, currentUserId, name, keyHash, keyPrefix, permissions, expiresAt, true, NOW())
   ```
6. Audit log: action 'create', entity_type 'api_key', entity_id = new key ID (do NOT log the key or hash in audit).
7. Return response (HTTP 201):
   ```json
   {
     "success": true,
     "data": {
       "id": "uuid",
       "name": "My Integration Key",
       "key": "crm_a3b2c4d5e6f7...full key here...",
       "keyPrefix": "crm_a3b2",
       "permissions": ["read", "write"],
       "expiresAt": null,
       "createdAt": "2026-03-25T14:30:00Z"
     }
   }
   ```
   **CRITICAL**: The `key` field is returned ONLY in this response. It is never stored and cannot be retrieved again.

#### Frontend Behavior After Generation

- Display the full key in a modal dialog.
- Key displayed in a monospace font text field with a "Copy" button.
- Warning message (yellow/orange): "This API key will only be shown once. Copy it now and store it securely. You will not be able to see it again."
- "Done" button closes the modal.
- After closing, the key list refreshes showing the new key (with prefix only).

---

### List API Keys (GET /api/v1/api-keys)

#### Authorization

- Admin only.

#### Response

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "name": "My Integration Key",
      "keyPrefix": "crm_a3b2",
      "permissions": ["read", "write"],
      "isActive": true,
      "expiresAt": null,
      "lastUsedAt": "2026-03-24T10:15:00Z",
      "createdAt": "2026-03-20T09:00:00Z",
      "createdBy": {
        "id": "uuid",
        "firstName": "Admin",
        "lastName": "User"
      }
    }
  ],
  "meta": {
    "pagination": { "total": 5, "page": 1, "limit": 25, "totalPages": 1 }
  }
}
```

- **Never** return the key or key_hash.
- Show both active and inactive keys (inactive shown as grayed out with "Revoked" badge).
- Sorted by createdAt DESC.

---

### Revoke API Key (DELETE /api/v1/api-keys/:id)

#### Authorization

- Admin only.

#### Process

1. Find API key by ID. If not found: 404.
2. Set `is_active = false` and `revoked_at = NOW()` (soft delete — record is preserved for audit trail).
3. Audit log: action 'revoke', entity_type 'api_key', entity_id = key ID.
4. Return: HTTP 200 `{ "success": true, "data": { "id": "uuid", "revoked": true } }`

#### Frontend

- Revoke button with confirmation dialog: "Are you sure you want to revoke this API key? Any integrations using this key will immediately stop working."
- After revocation, the key row shows "Revoked" badge and revoke button is disabled.

---

### API Key Authentication (Middleware)

#### File: `server/src/middleware/auth.middleware.ts`

The auth middleware supports both session-based auth and API key auth. The check order:

1. Check for session cookie first (existing flow).
2. If no valid session, check for `Authorization: Bearer crm_xxxxx` header.
3. If neither: return 401 UNAUTHORIZED.

#### API Key Verification Process

1. Extract token from Authorization header:
   - Header format: `Bearer crm_xxxxxxxxxxxx`
   - If header present but doesn't start with "Bearer crm_": skip API key auth, fall through to 401

2. Extract prefix for lookup:
   ```typescript
   const keyPrefix = fullKey.substring(0, 8); // e.g., "crm_a3b2"
   ```

3. Look up candidate API key records by prefix:
   ```sql
   SELECT * FROM api_keys WHERE key_prefix = :keyPrefix
   ```

4. For each candidate, compute HMAC and compare using timing-safe equality:
   ```typescript
   const computedHash = crypto.createHmac('sha256', process.env.API_KEY_SECRET)
     .update(fullKey)
     .digest();
   const storedHash = Buffer.from(candidate.key_hash, 'hex');
   if (crypto.timingSafeEqual(computedHash, storedHash)) {
     // Match found
   }
   ```

5. Validate the matched key:
   a. Record exists? If not: 401 UNAUTHORIZED "Invalid API key"
   b. is_active = true? If not: 401 UNAUTHORIZED "API key has been revoked"
   c. expires_at is null OR expires_at > NOW()? If expired: 401 UNAUTHORIZED "API key has expired"
   d. Check permissions match requested action (see Permission Mapping below). If insufficient: 403 FORBIDDEN "API key does not have permission for this action"

6. Set request context:
   ```typescript
   req.user = /* load user from api_keys.user_id */;
   req.authMethod = 'api_key';
   req.apiKeyId = apiKey.id;
   ```

7. Update last_used_at (fire-and-forget, non-blocking):
   ```sql
   UPDATE api_keys SET last_used_at = NOW() WHERE id = :id
   ```

#### CSRF Exemption

- API key-authenticated requests are **exempt from CSRF checks**. They don't use cookies, so CSRF protection does not apply.

#### Permission Mapping

| Permission Level | Allowed HTTP Methods |
|-----------------|---------------------|
| `read` | GET only |
| `write` | GET, POST, PUT, PATCH |
| `delete` | GET, POST, PUT, PATCH, DELETE |
| `admin` | All methods + access to admin-only endpoints (users, settings, api-keys, audit-log) |

- Permissions are hierarchical for HTTP methods but NOT for admin endpoints.
- A key with `['delete']` permission can do GET + POST + PUT + PATCH + DELETE on non-admin endpoints, but CANNOT access admin endpoints.
- Only `admin` permission grants access to admin-only endpoints.
- If a key has `['read', 'admin']`, it can read everything including admin endpoints, but cannot create/update/delete on non-admin endpoints.

#### Rate Limiting for API Keys

- 100 requests per minute per API key (tracked by api_key.id).
- Use a sliding window counter (in-memory store or Redis).
- When rate limited: return 429 TOO_MANY_REQUESTS with headers:
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: 0`
  - `X-RateLimit-Reset: {unix timestamp when window resets}`
  - `Retry-After: {seconds until reset}`
- Rate limiting for session auth and API key auth are tracked independently.

---

---

# SECTION 6k: Core Features — Settings

## Frontend Routes

- `/settings` — Settings landing page (redirects to /settings/general)
- `/settings/general` — General settings
- `/settings/pipeline` — Deal pipeline stages
- `/settings/activity-types` — Activity type management
- `/settings/email-templates` — Email template management
- `/settings/api-keys` — API key management
- `/settings/audit-log` — Audit log viewer

## Settings Page (Admin only)

### Access Control

- Entire settings page and all settings API endpoints require role = 'admin'.
- Non-admin users: settings link is hidden from navigation; direct URL access returns 403.

### Settings Storage

- Settings stored in a `settings` table with JSONB value column.
- Schema: `id (UUID), key (VARCHAR(100) UNIQUE), value (JSONB), updated_at, updated_by (UUID FK users)`.
- All settings are cached in memory on server start and refreshed when updated.

---

### General Settings

| Setting Key | Type | Default | Validation | Description |
|------------|------|---------|------------|-------------|
| crm_name | string | "CRM" | 1–100 chars | Displayed in header, emails, page title |
| company_name | string | "" | 0–255 chars | Organization name |
| logo_url | string | "" | Valid URL or empty; max 500 chars | Logo image URL (displayed in header, emails) |
| primary_color | string | "#3B82F6" | Valid hex color (3 or 6 digit, with #) | Primary UI accent color |
| timezone | string | "UTC" | Valid IANA timezone string (e.g., "America/New_York") | Default timezone for date/time display |
| date_format | string | "MM/DD/YYYY" | One of: 'MM/DD/YYYY', 'DD/MM/YYYY', 'YYYY-MM-DD' | Date display format throughout the UI |
| currency | string | "USD" | Valid 3-letter ISO 4217 code | Default currency for deal values |

#### Frontend UI

- Form with labeled inputs for each setting.
- Color picker input for primary_color (with hex text input fallback).
- Timezone: searchable dropdown of IANA timezones.
- Date format: radio buttons or select dropdown with preview of today's date in each format.
- Currency: searchable dropdown of ISO 4217 currencies.
- "Save" button at bottom. On save, all settings are sent as a single PUT request.
- Show success toast on save. Show validation errors inline per field.

---

### Deal Pipeline Settings

#### Data Model

Deal stages are stored in the `deal_stages` table:

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| name | VARCHAR(100) | Stage name (e.g., "Lead") |
| order | INTEGER | Sort order (0-based, unique) |
| default_probability | INTEGER | 0–100; default win probability for deals in this stage |
| is_won | BOOLEAN | Exactly one stage must have is_won=true (e.g., "Closed Won") |
| is_lost | BOOLEAN | Exactly one stage must have is_lost=true (e.g., "Closed Lost") |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

#### Default Stages (Seeded)

| Order | Name | Probability | is_won | is_lost |
|-------|------|-------------|--------|---------|
| 0 | Lead | 10 | false | false |
| 1 | Qualified | 20 | false | false |
| 2 | Proposal | 40 | false | false |
| 3 | Negotiation | 60 | false | false |
| 4 | Closed Won | 100 | true | false |
| 5 | Closed Lost | 0 | false | true |

#### Validation Rules

- **Minimum 2 stages** must exist at all times.
- **Exactly one is_won stage**: There must be exactly one stage with is_won=true. Error: "Exactly one stage must be marked as 'Won'"
- **Exactly one is_lost stage**: There must be exactly one stage with is_lost=true. Error: "Exactly one stage must be marked as 'Lost'"
- **No duplicate names** (case-insensitive). Error: "Stage name '{name}' is already in use"
- **Probability 0–100** for each stage.
- **Cannot delete a stage that has existing deals**. If a stage has deals, show modal: "This stage has {N} deals. Please select a stage to move them to:" with a dropdown of other stages. On confirm, reassign all deals to the selected stage, then delete the original stage.

#### UI

- Drag-and-drop reorderable list of stages.
- Each row shows: drag handle, name (editable inline), probability (editable number input 0–100), is_won checkbox, is_lost checkbox, deal count badge, delete button.
- "Add Stage" button at bottom → inserts a new stage at the end with default name "New Stage" and probability 50.
- **Delete behavior**:
  - If stage has 0 deals: delete immediately with confirmation dialog.
  - If stage has deals: show reassignment modal as described above.
  - Cannot delete the is_won or is_lost stage (delete button disabled with tooltip).
- **Reorder**: On drag-drop, update the `order` field for all affected stages.
- "Save" button: sends PUT /api/v1/settings/deal-stages with the full array of stages (order determined by array position).

#### API: PUT /api/v1/settings/deal-stages

```json
{
  "stages": [
    { "id": "uuid-or-null", "name": "Lead", "defaultProbability": 10, "isWon": false, "isLost": false },
    { "id": "uuid-or-null", "name": "Qualified", "defaultProbability": 20, "isWon": false, "isLost": false },
    { "id": null, "name": "New Stage", "defaultProbability": 50, "isWon": false, "isLost": false }
  ]
}
```

- Array order determines `order` field (index 0 = order 0, etc.).
- If `id` is null: create new stage.
- If an existing stage ID is missing from the array: it's being deleted. Check for deals first.
- All validation rules above are enforced server-side.

---

### Activity Type Settings

#### Data Model

Activity types in `activity_types` table:

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| name | VARCHAR(100) | Type name |
| icon | VARCHAR(50) | Lucide icon name (e.g., "phone", "mail", "calendar") |
| color | VARCHAR(7) | Hex color |
| is_system | BOOLEAN | If true, cannot be deleted |
| is_active | BOOLEAN | If false, hidden from creation dropdowns but existing records preserved |
| created_at | TIMESTAMP | |

#### Default Types (Seeded)

| Name | Icon | Color | Is System |
|------|------|-------|-----------|
| Call | phone | #3B82F6 | true |
| Email | mail | #10B981 | true |
| Meeting | calendar | #8B5CF6 | true |
| Task | check-square | #F59E0B | true |
| Note | file-text | #6B7280 | true |

#### UI

- List of activity types with: icon preview, name (editable), icon selector (dropdown of Lucide icons), color picker, active toggle, delete button.
- **System types (is_system=true): cannot be deleted** (delete button disabled with tooltip "System type cannot be deleted"). Name, icon, and color are still editable.
- "Add Type" button → adds new type with defaults.
- Deactivating a type: existing activities of that type remain; type just won't appear in creation dropdowns.

---

### Contact/Company Source Options

#### Storage

Sources stored in settings table as a JSON array under setting key `contact_sources`:

```json
["Website", "Referral", "LinkedIn", "Cold Call", "Trade Show", "Other"]
```

#### UI

- Reorderable list of source values.
- Each row: drag handle, source name (editable inline), delete button.
- "Add Source" button.
- Delete: allowed even if contacts use this source (existing contacts keep the value; it just won't appear in dropdowns for new contacts).
- Validate: no empty strings, no duplicates (case-insensitive), max 50 sources.

---

### Email Template Management

See SECTION 6i for full details. Summary of settings UI:

- List all email templates in a table: name, key, subject, is_active, is_system, updated_at.
- Click to edit → edit page with subject, HTML body, text body, variable reference, preview, test send.
- "Create Template" button for custom templates.
- **System templates (is_system=true) cannot be deleted** — only edited.

---

### Audit Log Viewer

#### UI

- Read-only. No create, update, or delete operations on audit log entries.
- Table with columns: Timestamp, User, Action, Entity Type, Entity ID, IP Address.
- Click a row to expand and show: `changes` column (formatted JSON with `before`/`after` diff highlighting if both present).

#### Filters

| Filter | Type | Description |
|--------|------|-------------|
| userId | Select dropdown | Filter by user who performed the action |
| action | Select dropdown | Options: create, update, delete, login, logout, revoke |
| entityType | Select dropdown | Options: contact, company, deal, activity, meeting, note, attachment, user, api_key, setting, email_template |
| dateFrom | Date picker | Start of date range |
| dateTo | Date picker | End of date range |
| search | Text input | Search in entity_id or `changes` column (server-side JSONB search) |

#### Pagination

- `?page=1&limit=50` (default limit: 50)
- Sorted by createdAt DESC (most recent first).
- Total count displayed.

#### API: GET /api/v1/audit-log

- Query params: userId, action, entityType, dateFrom, dateTo, search, page, limit
- Response: Section 4 canonical format with `meta.pagination`
- Each entry: id, userId (with user name), action, entityType, entityId, oldValues, newValues, ipAddress, createdAt

---

---

# SECTION 6l: Core Features — Dashboard

## Frontend Route

- `/` — Dashboard (until built in M9, redirect to /contacts)

## Dashboard (Home page after login)

### Route

- Path: `/` (root)
- Requires authentication (redirect to /login if not authenticated)
- Available to all roles (admin, member)
- Until the dashboard is built in Milestone 9, the `/` route redirects to `/contacts`.

### Scope

- **Globally scoped**: Total Contacts, Total Companies, Open Deals Value, Deals Won This Month, Pipeline Overview, Recent Activities, Deal Forecast, Activity by Type — all show data across the entire CRM (not filtered by current user).
- **User-specific**: My Tasks and Upcoming Meetings are filtered to the current authenticated user.

### Date Range Picker

- A date range picker at the top of the dashboard controls the date range for applicable widgets (metrics, activity stats, deal forecast).
- Default: last 30 days.
- Presets: "Last 7 days", "Last 30 days", "Last 90 days", "This month", "This quarter", "This year"
- Custom date range picker (start date + end date).
- The selected range is passed as `?dateFrom=&dateTo=` query params to dashboard API endpoints.

### Data Loading

- Dashboard loads data from multiple API endpoints in parallel.
- Use Promise.all (or equivalent) to fetch all dashboard data concurrently.
- Show skeleton loaders for each card/widget while loading.
- If an individual widget fails to load, show an error state for that widget only (not the entire dashboard). Error state: "Failed to load. Retry" link.
- **Performance target**: < 1 second total load time for all dashboard data.
- Auto-refresh: no auto-refresh. User can manually refresh with a "Refresh" button in the top-right corner.

---

### Key Metrics Cards (Top Row)

Six cards displayed in a responsive grid (3 columns on desktop, 2 on tablet, 1 on mobile). **All metric cards are clickable** and link to filtered views.

**Responsive:** All metric cards stack in a 2×3 grid on tablet, single column on mobile. Charts resize to full width. Pipeline chart scrolls horizontally if needed. Recent activities and upcoming meetings sections stack vertically.

#### API: GET /api/v1/dashboard/metrics

Response:
```json
{
  "success": true,
  "data": {
    "totalContacts": {
      "count": 1250,
      "changeFromLastMonth": 45,
      "changePercent": 3.7
    },
    "totalCompanies": {
      "count": 340,
      "changeFromLastMonth": 12,
      "changePercent": 3.6
    },
    "openDealsValue": {
      "value": 2450000,
      "currency": "USD",
      "count": 87
    },
    "dealsWonThisMonth": {
      "count": 8,
      "value": 320000,
      "currency": "USD"
    },
    "tasksDueToday": {
      "count": 5,
      "overdueCount": 2
    },
    "meetingsToday": {
      "count": 3,
      "nextMeetingAt": "2026-03-25T14:00:00Z"
    }
  }
}
```

#### Card Rendering

1. **Total Contacts**:
   - Large number: 1,250 (formatted with thousands separator per locale)
   - Subtitle: "+45 from last month" (green if positive, red if negative, gray if zero)
   - Icon: users (Lucide)
   - Click → navigate to /contacts

2. **Total Companies**:
   - Large number: 340
   - Subtitle: "+12 from last month"
   - Icon: building (Lucide)
   - Click → navigate to /companies

3. **Open Deals Value**:
   - Large number: $2,450,000 (formatted with currency symbol and thousands separator)
   - Subtitle: "87 open deals"
   - Icon: dollar-sign (Lucide)
   - Click → navigate to /deals?status=open

4. **Deals Won This Month**:
   - Large number: 8
   - Subtitle: "$320,000 total value"
   - Icon: trophy (Lucide)
   - Click → navigate to /deals?status=won&closedAfter={first day of current month}

5. **Tasks Due Today** (user-specific):
   - Large number: 5
   - Subtitle: "2 overdue" (red text if overdue > 0)
   - Icon: check-square (Lucide)
   - Click → navigate to /activities?type=task&dueDate=today

6. **Meetings Today** (user-specific):
   - Large number: 3
   - Subtitle: "Next at 2:00 PM" (or "No more meetings today" if all past)
   - Icon: calendar (Lucide)
   - Click → navigate to /meetings?startAfter={today_start}&startBefore={today_end}

---

### Pipeline Overview

#### API: GET /api/v1/dashboard/pipeline

Response:
```json
{
  "success": true,
  "data": {
    "stages": [
      { "id": "uuid", "name": "Lead", "dealCount": 25, "totalValue": 500000, "order": 0 },
      { "id": "uuid", "name": "Qualified", "dealCount": 18, "totalValue": 720000, "order": 1 },
      { "id": "uuid", "name": "Proposal", "dealCount": 12, "totalValue": 480000, "order": 2 },
      { "id": "uuid", "name": "Negotiation", "dealCount": 8, "totalValue": 640000, "order": 3 }
    ],
    "totalOpenValue": 2340000
  }
}
```

#### Rendering

- Horizontal bar chart built with **pure SVG/CSS** (no external charting library).
- Each bar represents a pipeline stage (excluding Closed Won / Closed Lost).
- Bar width proportional to deal count or total value (toggle option).
- Bar label: stage name + deal count + total value.
- Color: use stage-specific colors or a gradient from primary color.
- Click on a bar/stage → navigate to /deals?stageId={stageId}.
- Below the chart: "Total pipeline: $2,340,000" summary.

---

### Recent Activities

#### API: GET /api/v1/dashboard/recent-activities

Response:
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "type": "call",
      "typeIcon": "phone",
      "typeColor": "#3B82F6",
      "subject": "Follow-up call with John",
      "entityType": "contact",
      "entityId": "uuid",
      "entityName": "John Smith",
      "user": { "id": "uuid", "firstName": "Jane", "lastName": "Doe", "avatar": null },
      "createdAt": "2026-03-25T13:45:00Z"
    }
  ]
}
```

- Limited to 10 most recent activities.
- Globally scoped — shows all CRM activities (for team visibility), not filtered by current user.

#### Rendering

- Vertical timeline list.
- Each item: type icon (colored circle with Lucide icon) + subject + "on {entityName}" (clickable link) + user name + relative time ("5 min ago", "2 hours ago").
- "View All Activities" link at bottom → navigate to /activities.

---

### Upcoming Meetings (User-specific)

#### API: GET /api/v1/dashboard/upcoming-meetings

- Returns next 5 meetings where start_time > NOW(), sorted by start_time ASC.
- **User-specific**: Only meetings where current user is owner or attendee.

Response:
```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "title": "Product Demo",
      "startTime": "2026-03-25T14:00:00Z",
      "endTime": "2026-03-25T15:00:00Z",
      "location": "Zoom",
      "meetingUrl": "https://zoom.us/j/123",
      "attendeeCount": 3,
      "contact": { "id": "uuid", "firstName": "John", "lastName": "Smith" }
    }
  ]
}
```

#### Rendering

- Card list, each showing:
  - Title (clickable → /meetings/:id)
  - Time: "Today at 2:00 PM" or "Tomorrow at 10:00 AM" or "Mar 27 at 3:00 PM"
  - Location or "Join" button if meetingUrl is present
  - Attendee count: "{N} attendees"
  - Contact name if linked (clickable)
- "View All Meetings" link → /meetings.

---

### My Tasks (User-specific)

#### API: GET /api/v1/dashboard/my-tasks

- **User-specific**: Returns tasks assigned to current user, grouped by urgency.
- Only tasks where `is_completed = false`.

Response:
```json
{
  "success": true,
  "data": {
    "overdue": [
      { "id": "uuid", "subject": "Send proposal", "dueDate": "2026-03-23", "priority": "high", "entityName": "Acme Corp" }
    ],
    "dueToday": [
      { "id": "uuid", "subject": "Follow up call", "dueDate": "2026-03-25", "priority": "medium", "entityName": "John Smith" }
    ],
    "upcoming": [
      { "id": "uuid", "subject": "Prepare deck", "dueDate": "2026-03-28", "priority": "low", "entityName": null }
    ]
  }
}
```

#### Rendering

- Three sections with colored headers:
  - **Overdue** (red header, red left border on items)
  - **Due Today** (yellow/amber header)
  - **This Week** (neutral/blue header)
- Each task shows: checkbox (for quick complete), subject, due date, priority badge, entity name (clickable).
- Quick complete: clicking checkbox sends PUT /api/v1/activities/:id with `{ "status": "completed" }`. On success, task fades out of the list.
- "View All Tasks" link → /activities?type=task&ownerId={currentUserId}.
- Max 5 items per section. If more, show "+N more" link.

---

### Deal Forecast

#### API: GET /api/v1/dashboard/deal-forecast

- Calculates expected revenue by month for the next 3 months.
- Formula per deal: `amount * (stage.default_probability / 100)`.
- Groups by expected_close_date month.
- Only includes deals where status is open (not won/lost).

Response:
```json
{
  "success": true,
  "data": {
    "months": [
      { "month": "2026-04", "label": "April 2026", "expectedValue": 450000, "dealCount": 15 },
      { "month": "2026-05", "label": "May 2026", "expectedValue": 320000, "dealCount": 10 },
      { "month": "2026-06", "label": "June 2026", "expectedValue": 180000, "dealCount": 7 }
    ],
    "total": 950000,
    "currency": "USD"
  }
}
```

#### Rendering

- Vertical bar chart with 3 bars (one per month).
- Built with **pure SVG/CSS** (no external charting library).
- Bar height proportional to expectedValue.
- Label on each bar: month name + formatted value.
- Below bar: deal count.
- Total line below chart: "3-month forecast: $950,000".

---

### Activity by Type (Last 30 days)

#### API: GET /api/v1/dashboard/activity-stats

Response:
```json
{
  "success": true,
  "data": {
    "byType": [
      { "type": "call", "label": "Calls", "count": 145, "color": "#3B82F6" },
      { "type": "email", "label": "Emails", "count": 230, "color": "#10B981" },
      { "type": "meeting", "label": "Meetings", "count": 42, "color": "#8B5CF6" },
      { "type": "task", "label": "Tasks", "count": 98, "color": "#F59E0B" },
      { "type": "note", "label": "Notes", "count": 67, "color": "#6B7280" }
    ],
    "total": 582,
    "period": "last_30_days"
  }
}
```

#### Rendering

- Donut chart built with **pure SVG** (no external charting library).
- Each segment colored by activity type color.
- Center of donut: total count "582 activities".
- Legend below/beside the chart: colored circle + type label + count.
- Click a segment → navigate to /activities?type={type}&createdAfter={30 days ago}.

---

### Dashboard Layout

```
+--------------------------------------------------------------+
|  [Metric Card] [Metric Card] [Metric Card]                   |
|  [Metric Card] [Metric Card] [Metric Card]                   |
+--------------------------------------------------------------+
|  [Pipeline Overview — Full Width]                             |
+-------------------------------+------------------------------+
|  [Recent Activities]          |  [Upcoming Meetings]          |
|                               |                               |
+-------------------------------+------------------------------+
|  [My Tasks]                   |  [Deal Forecast]              |
|                               |                               |
+-------------------------------+------------------------------+
|  [Activity by Type — Full Width or Half]                      |
+--------------------------------------------------------------+
```

- Responsive: on mobile, all widgets stack vertically in single column.
- On tablet: 2-column grid.
- On desktop: layout as shown above.

---

---

# SECTION 7: API Design

## Base URL

All API endpoints are prefixed with `/api/v1/`. This versioning allows future breaking changes to be introduced under `/api/v2/` without disrupting existing integrations.

- Development: `http://localhost:3000/api/v1/`
- Production: `https://{domain}/api/v1/`

---

## Response & Error Format

All responses follow the canonical format defined in **Section 4**. This section does not redefine it — refer to Section 4 for the full specification. Summary:

- **Success**: `{ "success": true, "data": { ... }, "meta": { ... } }`
- **Success (list)**: `{ "success": true, "data": [ ... ], "meta": { "pagination": { "total", "page", "limit", "totalPages" } } }`
- **Success (delete)**: `{ "success": true, "data": { "id": "uuid", "deleted": true } }`
- **Error**: `{ "success": false, "error": { "code": "ERROR_CODE", "message": "Human-readable message", "details": [ ... ] } }`

---

## Error Codes

| HTTP Status | Error Code | Description | When to Use |
|-------------|-----------|-------------|-------------|
| 400 | VALIDATION_ERROR | Input validation failed | Missing required fields, invalid formats, business rule violations |
| 401 | UNAUTHORIZED | Not authenticated | No session cookie or API key, expired session, invalid API key |
| 403 | FORBIDDEN | Insufficient permissions | User lacks role or permission for the action |
| 404 | NOT_FOUND | Resource not found | Entity ID doesn't exist or was deleted |
| 409 | CONFLICT | Conflict detected | Optimistic locking updatedAt mismatch, unique constraint violation (duplicate email, etc.) |
| 422 | UNPROCESSABLE_ENTITY | Semantic error | Request is syntactically valid but semantically wrong (e.g., trying to delete a deal stage that has deals without specifying reassignment) |
| 429 | RATE_LIMITED | Too many requests | Rate limit exceeded for IP, session, or API key |
| 500 | INTERNAL_ERROR | Server error | Unhandled exception (never expose stack trace or internal details to client) |

### Error Response Headers for Rate Limiting (429)

```
X-RateLimit-Limit: 200
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1711378800
Retry-After: 45
```

---

## Complete Endpoint List

### Auth

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| POST | /api/v1/auth/login | No | No | Login with email + password; returns session cookie |
| POST | /api/v1/auth/logout | Yes | No | Destroy session |
| GET | /api/v1/auth/me | Yes | No | Get current authenticated user profile |
| GET | /api/v1/auth/csrf-token | Yes | No | Get CSRF token (synchronizer token pattern) |
| POST | /api/v1/auth/change-password | Yes | No | Change own password (requires current password) |
| POST | /api/v1/auth/forgot-password | No | No | Request password reset email |
| POST | /api/v1/auth/reset-password | No | No | Reset password with valid token |
| PUT | /api/v1/auth/profile | Yes | No | Update own profile (name, avatar, etc.) |

### Users (Admin Only)

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/users | Yes | Yes | List all users with pagination/filtering |
| GET | /api/v1/users/:id | Yes | Yes | Get single user details |
| POST | /api/v1/users | Yes | Yes | Create new user |
| PUT | /api/v1/users/:id | Yes | Yes | Update user |
| DELETE | /api/v1/users/:id | Yes | Yes | Deactivate user (soft delete) |

### Contacts

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/contacts | Yes | No | List contacts with pagination/filtering/sorting |
| GET | /api/v1/contacts/:id | Yes | No | Get single contact with all relations |
| POST | /api/v1/contacts | Yes | No | Create new contact |
| PUT | /api/v1/contacts/:id | Yes | No | Update contact |
| DELETE | /api/v1/contacts/:id | Yes | No | Hard delete contact |
| POST | /api/v1/contacts/import | Yes | No | Import contacts from CSV |
| GET | /api/v1/contacts/export | Yes | No | Export contacts to CSV |
| GET | /api/v1/contacts/:id/export-data | Yes | Yes | Export all contact data (GDPR) |

### Companies

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/companies | Yes | No | List companies |
| GET | /api/v1/companies/:id | Yes | No | Get single company with relations |
| POST | /api/v1/companies | Yes | No | Create company |
| PUT | /api/v1/companies/:id | Yes | No | Update company |
| DELETE | /api/v1/companies/:id | Yes | No | Hard delete company |

### Deals

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/deals | Yes | No | List deals (list + Kanban support) |
| GET | /api/v1/deals/:id | Yes | No | Get single deal with relations |
| POST | /api/v1/deals | Yes | No | Create deal |
| PUT | /api/v1/deals/:id | Yes | No | Update deal (including stage moves) |
| DELETE | /api/v1/deals/:id | Yes | No | Hard delete deal |
| GET | /api/v1/deals/stats | Yes | No | Deal statistics (won/lost/pipeline value) |

### Activities

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/activities | Yes | No | List activities with filtering |
| GET | /api/v1/activities/:id | Yes | No | Get single activity |
| POST | /api/v1/activities | Yes | No | Create activity |
| PUT | /api/v1/activities/:id | Yes | No | Update activity |
| DELETE | /api/v1/activities/:id | Yes | No | Hard delete activity |

### Meetings

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/meetings | Yes | No | List meetings (list + calendar support) |
| GET | /api/v1/meetings/:id | Yes | No | Get single meeting with attendees |
| POST | /api/v1/meetings | Yes | No | Create meeting |
| PUT | /api/v1/meetings/:id | Yes | No | Update meeting |
| DELETE | /api/v1/meetings/:id | Yes | No | Hard delete meeting |

### Notes

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/notes | Yes | No | List notes (filtered by entity) |
| POST | /api/v1/notes | Yes | No | Create note |
| PUT | /api/v1/notes/:id | Yes | No | Update note |
| DELETE | /api/v1/notes/:id | Yes | No | Hard delete note |

### Attachments

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| POST | /api/v1/attachments/upload | Yes | No | Upload file attachment |
| GET | /api/v1/attachments/:id/download | Yes | No | Download attachment |
| DELETE | /api/v1/attachments/:id | Yes | No | Hard delete attachment |
| GET | /api/v1/attachments | Yes | No | List attachments for an entity |

### API Keys (Admin Only)

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/api-keys | Yes | Yes | List all API keys |
| POST | /api/v1/api-keys | Yes | Yes | Generate new API key |
| DELETE | /api/v1/api-keys/:id | Yes | Yes | Revoke API key |

### Settings (Admin Only)

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/settings | Yes | Yes | Get all settings |
| PUT | /api/v1/settings | Yes | Yes | Update settings |
| GET | /api/v1/settings/deal-stages | Yes | Yes | Get deal pipeline stages |
| PUT | /api/v1/settings/deal-stages | Yes | Yes | Update deal pipeline stages |
| GET | /api/v1/settings/activity-types | Yes | Yes | Get activity types |
| PUT | /api/v1/settings/activity-types | Yes | Yes | Update activity types |

### Email Templates (Admin Only)

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/email-templates | Yes | Yes | List all email templates |
| GET | /api/v1/email-templates/:id | Yes | Yes | Get single template |
| POST | /api/v1/email-templates | Yes | Yes | Create custom template |
| PUT | /api/v1/email-templates/:id | Yes | Yes | Update template |
| DELETE | /api/v1/email-templates/:id | Yes | Yes | Delete non-system template |

### Audit Log (Admin Only)

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/audit-log | Yes | Yes | Query audit log entries |

### Dashboard

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/dashboard/metrics | Yes | No | Key metric cards data |
| GET | /api/v1/dashboard/pipeline | Yes | No | Pipeline overview data |
| GET | /api/v1/dashboard/recent-activities | Yes | No | Latest 10 activities |
| GET | /api/v1/dashboard/upcoming-meetings | Yes | No | Next 5 meetings for current user |
| GET | /api/v1/dashboard/my-tasks | Yes | No | Current user's tasks grouped by urgency |
| GET | /api/v1/dashboard/deal-forecast | Yes | No | 3-month deal forecast |
| GET | /api/v1/dashboard/activity-stats | Yes | No | Activity distribution by type (30 days) |

### Tags

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/tags | Yes | No | List all tags |
| POST | /api/v1/tags | Yes | No | Create tag |
| PUT | /api/v1/tags/:id | Yes | No | Update tag |
| DELETE | /api/v1/tags/:id | Yes | No | Hard delete tag (remove from all entities) |

### Global Search

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/search | Yes | No | Search across contacts, companies, deals |

Query params:
- `q` (required): search query, min 2 chars
- `type` (optional): comma-separated list of entity types to search (default: all). Options: contacts, companies, deals
- `limit` (optional): max results per type (default: 5, max: 20)

Response:
```json
{
  "success": true,
  "data": {
    "contacts": [
      { "id": "uuid", "type": "contact", "title": "John Smith", "subtitle": "john@example.com", "url": "/contacts/uuid" }
    ],
    "companies": [
      { "id": "uuid", "type": "company", "title": "Acme Corp", "subtitle": "Technology", "url": "/companies/uuid" }
    ],
    "deals": [
      { "id": "uuid", "type": "deal", "title": "Enterprise License", "subtitle": "$50,000 — Negotiation", "url": "/deals/uuid" }
    ]
  }
}
```

### Health Check

| Method | Path | Auth Required | Admin Only | Description |
|--------|------|:------------:|:----------:|-------------|
| GET | /api/v1/health | No | No | Health check endpoint (returns 200 if server is running) |

---

## API Key Auth vs Session Auth

- All endpoints accept BOTH authentication methods.
- Session auth: validated via `crm.sid` cookie → session looked up in PostgreSQL sessions table.
- API key auth: validated via `Authorization: Bearer crm_xxxx` header → key prefix lookup, then HMAC-SHA256 comparison using `crypto.timingSafeEqual`.
- If both are present in a request, session auth takes precedence.
- API key permissions control which HTTP methods are allowed (see Section 6j for permission mapping).
- Rate limiting tracked independently: session-based requests are rate-limited per IP; API key requests are rate-limited per key.

---

## Pagination Convention

- **Mechanism**: Page-based pagination.
- **Query params**: `?page=1&limit=25`
- **Page**: 1-indexed (first page is 1, not 0). If page < 1, default to 1.
- **Limit**: Number of items per page.
  - Default: 25
  - Minimum: 1
  - Maximum: 100
  - If limit > 100, silently cap to 100 (do not error).
  - If limit < 1, default to 25.
- **Response**: Always include pagination metadata in `meta.pagination`:
  - `total`: total number of records matching the filter (before pagination)
  - `page`: current page number
  - `limit`: current page size
  - `totalPages`: `Math.ceil(total / limit)`
- **Empty results**: Return `data: []` with `total: 0, page: 1, limit: 25, totalPages: 0`.
- **Out of range page**: If `page > totalPages`, return `data: []` (do not error).

---

## Sorting Convention

- **Query params**: `?sort=fieldName&order=asc|desc`
- **Default sort**: `createdAt` DESC for all list endpoints (unless otherwise specified per module).
- **`sort` values**: camelCase field names in API (mapped to snake_case DB columns internally).
- **`order` values**: `asc` or `desc` (case-insensitive). Invalid values default to `desc`.
- **Invalid sort field**: If the sort field is not in the allowed list for that endpoint, ignore it and use the default sort (do not error).
- **Null handling**: Nulls sort last for ASC, first for DESC (PostgreSQL default NULLS LAST / NULLS FIRST).

---

## Filtering Convention

- **Query params**: Each filterable field has its own query parameter (camelCase).
- **Exact match**: `?status=active` — matches records where status equals 'active'.
- **Multi-value (OR)**: `?status=active,inactive` — comma-separated values; matches records where status is 'active' OR 'inactive'. Backend splits on comma and uses `WHERE status IN (...)`.
- **Date ranges**: `?createdAfter=2026-01-01&createdBefore=2026-03-31` — inclusive on both ends. Suffix convention: `After` (>=), `Before` (<=).
- **Search**: `?search=query` — performs case-insensitive partial match (ILIKE) across relevant text fields (defined per module).
- **Boolean filters**: `?isActive=true` or `?isActive=false` — string "true"/"false" parsed to boolean.
- **UUID filters**: Validated as UUID v4 format. Invalid UUID format returns 400 VALIDATION_ERROR.
- **Unknown filter params**: Silently ignored (do not error on unexpected query params).
- **Combining filters**: All filters are AND-combined. For example, `?status=active&ownerId=uuid` returns records where status is active AND owner_id matches.

---

---

# SECTION 8: Security

## Authentication Security

### Password Hashing
- Algorithm: bcrypt
- Cost factor: 12 (provides ~250ms hash time on modern hardware)
- Library: `bcryptjs` (pure JS, no native dependencies) or `bcrypt` (native, faster)
- Process: `const hash = await bcrypt.hash(plaintext, 12);`
- Verification: `const match = await bcrypt.compare(plaintext, hash);`
- NEVER store plaintext passwords, NEVER log passwords, NEVER include passwords in API responses.

### Password Requirements
- Minimum **10** characters
- Maximum **128** characters
- At least 1 uppercase letter (A-Z)
- At least 1 lowercase letter (a-z)
- At least 1 digit (0-9)
- At least 1 special character (!@#$%^&*()_+-=[]{}|;':",.<>?/`~)
- Error messages should specify which requirement is missing (e.g., "Password must contain at least one uppercase letter")
- Check on: user creation, password change, password reset
- Validation implemented with **Zod** (not express-validator)

### Login Rate Limiting
- 5 failed login attempts per email address per 15-minute window
- After 5 failures: return 429 with message "Too many login attempts. Please try again in {minutes} minutes."
- Track by email address (not by IP, to prevent lockout of shared IPs)
- Successful login resets the counter for that email
- Lockout duration: remainder of the 15-minute window (not a rolling extension)

### Session Management
- Sessions stored server-side in PostgreSQL using `connect-pg-simple`
- Session ID generated by `express-session` (cryptographically random)
- **24-hour rolling session**: session expiry resets on each authenticated request
- Cookie settings:
  - `name`: `crm.sid`
  - `httpOnly`: true (prevents JavaScript access)
  - `secure`: true in production (HTTPS only), false in development
  - `sameSite`: `lax` (prevents CSRF on top-level navigations, allows same-site requests)
  - `maxAge`: 86400000 (24 hours in milliseconds)
  - `path`: '/'
  - `domain`: not set (defaults to current host)
- Session regeneration: call `req.session.regenerate()` after successful login to prevent session fixation
- Session invalidation on password change: destroy all other sessions for the user when password is changed
- Admin session management: admin can view active sessions for any user and invalidate them (force logout)
- Session cleanup: `connect-pg-simple` automatically prunes expired sessions (configurable interval, default 15 minutes)

### CSRF Protection
- Method: **Synchronizer token pattern**
- Implementation:
  1. Client requests a CSRF token via `GET /api/v1/auth/csrf-token`
  2. Server generates a cryptographically random token, stores it in the session, and returns it in the response body
  3. Frontend includes the token in a custom header on all state-changing requests: `X-CSRF-Token`
  4. Backend middleware validates: session CSRF token === header value
  5. Apply to all state-changing requests (POST, PUT, PATCH, DELETE)
  6. **API key-authenticated requests are exempt** from CSRF (they don't use cookies)
  7. Exempt: /api/v1/auth/login (no session yet), /api/v1/auth/forgot-password, /api/v1/auth/reset-password
- CSRF token rotation: generate a new token on each login

---

## API Security

### API Key Storage
- Raw API keys are NEVER stored in the database
- Keys are hashed using **HMAC-SHA256** with the `API_KEY_SECRET` environment variable as the server secret:
  ```typescript
  const keyHash = crypto.createHmac('sha256', process.env.API_KEY_SECRET)
    .update(fullKey)
    .digest('hex');
  ```
- Key prefix (first 8 characters, e.g., "crm_a3b2") is stored separately for lookup/display
- The full key is returned exactly once: in the response to the key generation request

### API Key Lookup & Verification
1. Extract prefix from the provided key: `fullKey.substring(0, 8)`
2. Query database: `SELECT * FROM api_keys WHERE key_prefix = :prefix`
3. For each candidate record, compute HMAC and compare using **`crypto.timingSafeEqual`**:
   ```typescript
   const computedHash = crypto.createHmac('sha256', process.env.API_KEY_SECRET)
     .update(fullKey)
     .digest();
   const storedHash = Buffer.from(candidate.key_hash, 'hex');
   if (crypto.timingSafeEqual(computedHash, storedHash)) {
     // Match found
   }
   ```
4. This approach prevents timing attacks and is the correct way to verify HMAC-based key hashes.

### API Key Rate Limiting
- 100 requests per minute per API key
- Tracked separately from session-based rate limits
- Implementation: sliding window counter using in-memory store or Redis
- Response headers on every API key request:
  - `X-RateLimit-Limit`: maximum requests per window
  - `X-RateLimit-Remaining`: remaining requests in current window
  - `X-RateLimit-Reset`: Unix timestamp when window resets

### API Key Expiration
- Keys can optionally have an expiration date
- Expired keys are rejected with 401 UNAUTHORIZED
- No automatic notification before expiration (future enhancement)

---

## Input Security

### General Input Validation
- All input validation happens on the backend. Frontend validation is for UX only and must never be trusted.
- Use **Zod** for all request validation (not express-validator).
- Validate all request body fields, query parameters, and URL parameters.
- Reject requests with unexpected fields in the body (whitelist known fields, strip unknown fields).

### SQL Injection Prevention
- Use Knex.js query builder for ALL database queries — never construct raw SQL strings with concatenation.
- For the rare cases where raw SQL is needed, use Knex's `knex.raw('SELECT * FROM users WHERE id = ?', [userId])` with parameterized placeholders.
- NEVER interpolate user input into SQL strings.

### XSS Prevention
- Sanitize all user input before storage where appropriate (especially HTML-capable fields like note content, email templates).
- Use `sanitize-html` for server-side HTML sanitization.
- Escape output in responses — React handles this by default for JSX, but be careful with `dangerouslySetInnerHTML` (only used for rendered markdown, always sanitize with `sanitize-html` first).
- Content-Security-Policy header restricts inline scripts.

### File Upload Security
- **No SVG uploads**: `image/svg+xml` is explicitly blocked. SVG files can contain embedded JavaScript and pose XSS risks.
- **MIME type validation via magic bytes**: Do NOT trust the Content-Type header alone. Use the `file-type` library to detect MIME type from file buffer magic bytes. If the detected type differs from the declared type, use the detected type.
- **File extension validation**: Validate the file extension matches the detected MIME type.
- **File size limit**: 25MB maximum, enforced by multer AND double-checked in handler.
- **Filename sanitization**: Strip all characters except `a-zA-Z0-9._-`, replace spaces with hyphens, truncate to 100 chars, lowercase.
- **No disk storage**: Use multer memory storage only. Files go from memory buffer directly to R2. Never write uploaded files to the server's filesystem.
- **Path traversal prevention**: The R2 key is constructed server-side using sanitized filename + UUID prefix. The original filename is stored in the DB for display but never used in file paths.
- **Max 50 attachments per entity**: Enforced at upload time.

---

## HTTP Security Headers

Applied via the `helmet` middleware on all responses:

| Header | Value | Purpose |
|--------|-------|---------|
| Content-Security-Policy | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'` | Prevent XSS, clickjacking, and other injection attacks |
| X-Content-Type-Options | `nosniff` | Prevent MIME type sniffing |
| X-Frame-Options | `DENY` | Prevent clickjacking (redundant with CSP frame-ancestors but kept for older browsers) |
| X-XSS-Protection | `0` | Disabled because modern CSP is more effective; the legacy XSS filter can introduce vulnerabilities |
| Strict-Transport-Security | `max-age=31536000; includeSubDomains` (production only) | Force HTTPS |
| Referrer-Policy | `strict-origin-when-cross-origin` | Limit referrer information leakage |
| Permissions-Policy | `camera=(), microphone=(), geolocation=()` | Disable unnecessary browser features |

---

## CORS (Cross-Origin Resource Sharing)

### Configuration

```typescript
// server/src/config/cors.ts
const corsOptions = {
  origin: process.env.FRONTEND_URL || 'http://localhost:5173',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-CSRF-Token'],
  exposedHeaders: ['X-RateLimit-Limit', 'X-RateLimit-Remaining', 'X-RateLimit-Reset', 'Retry-After'],
  maxAge: 86400, // Preflight cache: 24 hours
};
```

### Rules
- **Production**: Only the exact frontend origin is whitelisted. No wildcards (`*`).
- **Development**: Allow `http://localhost:5173` (Vite default) and `http://localhost:3000`.
- **credentials**: `true` — required for session cookies to be sent cross-origin.
- **No wildcard with credentials**: CORS spec forbids `Access-Control-Allow-Origin: *` when `credentials: true`. Always specify the exact origin.

---

## Rate Limiting

### Implementation

Use `express-rate-limit` middleware with appropriate stores.

| Context | Limit | Window | Key | Store |
|---------|-------|--------|-----|-------|
| Global (per IP) | 200 requests | 1 minute | IP address | Memory (or Redis in production) |
| Auth endpoints | 5 requests | 15 minutes | IP address | Memory (or Redis) |
| Login (per email) | 5 attempts | 15 minutes | Email address | Memory (or Redis) |
| API key (per key) | 100 requests | 1 minute | API key ID | Memory (or Redis) |
| File upload (per user) | 10 uploads | 1 minute | User ID | Memory (or Redis) |
| Export (per user) | 5 exports | 1 hour | User ID | Memory (or Redis) |
| Password reset (per email) | 3 requests | 1 hour | Email address | Memory (or Redis) |
| Meeting emails (per user) | 20 emails | 1 hour | User ID | Memory (or Redis) |
| Global email | 50 emails | 1 hour | Global | Memory (or Redis) |

### Rate Limit Response

- HTTP 429 Too Many Requests
- Body: `{ "success": false, "error": { "code": "RATE_LIMITED", "message": "Too many requests. Please try again later." } }`
- Headers:
  - `X-RateLimit-Limit`: maximum requests per window
  - `X-RateLimit-Remaining`: remaining requests in current window
  - `X-RateLimit-Reset`: Unix timestamp when window resets
  - `Retry-After`: seconds until reset

### Rate Limit Headers on All Responses

Rate limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) are included on **all** API responses (not just 429 responses) so clients can proactively manage their request rate.

### Rate Limit Bypass

- No bypass mechanism. Even admin users are rate-limited.
- If needed in the future, a rate limit exemption list can be added (by IP or user ID).

---

## Audit & Compliance

### Audit Log Table: `audit_log`

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Primary key |
| user_id | UUID | FK to users (who performed the action) |
| action | VARCHAR(50) | 'create', 'update', 'delete', 'login', 'logout', 'login_failed', 'revoke', 'export', 'import' |
| entity_type | VARCHAR(50) | 'contact', 'company', 'deal', 'activity', 'meeting', 'note', 'attachment', 'user', 'api_key', 'setting', 'email_template' |
| entity_id | UUID | ID of the affected entity (nullable for login/logout) |
| changes | JSONB | Single column with `{ "before": {...}, "after": {...} }` format. For creates: `before` is null. For deletes: `after` is null. For updates: both present with only changed fields. |
| ip_address | VARCHAR(45) | Client IP address (supports IPv6) |
| user_agent | VARCHAR(500) | Client user agent string |
| created_at | TIMESTAMP | When the action occurred (indexed) |

### Audit Log Rules

- **Immutability**: Audit log entries are **INSERT-only**. There is no UPDATE or DELETE endpoint for audit logs. The database user should ideally not have DELETE permission on this table in production.
- **Sensitive data**: Do NOT log passwords, API keys (raw or hashed), or email template content in `changes`. Redact these fields before logging.
- **Performance**: Use async logging where possible (fire-and-forget) so audit logging does not slow down the primary operation. If async logging fails, log the failure to application logs but do not fail the primary operation.
- **Retention**: Audit logs are retained indefinitely by default. A future admin setting can configure auto-purge of logs older than N days.

### Authorization Model Reference

The CRM uses two roles: `admin` (full access including settings, user management, API keys, audit log) and `member` (read/write access to all CRM entities; cannot access settings, users, or API keys).

Additional authorization rules:
- Notes: only creator or admin can edit/delete
- Attachments: only uploader or admin can delete
- Meetings: only owner or admin can delete
- All entities: any authenticated user can read

### GDPR Considerations

- **Data export**: Export all data related to a contact (contact record + all notes, activities, meetings, attachments metadata, email logs). Endpoint: `GET /api/v1/contacts/:id/export-data` (returns JSON). Admin-only.
- **Data deletion**: Delete all data for a contact (hard delete of contact and all related records, including R2 objects for attachments). Endpoint: `DELETE /api/v1/contacts/:id?gdpr=true` (performs hard delete cascading to all related entities). Admin-only.
- **Right to rectification**: Standard PUT /api/v1/contacts/:id handles this.

---

## Environment Variables Security

### Required Variables (validated at startup)

The server MUST check that all required environment variables are set when the application starts. If any required variable is missing, the server MUST log a clear error message identifying the missing variable(s) and exit with a non-zero exit code.

```typescript
// server/src/config/index.ts
const requiredVars = [
  'DATABASE_URL',
  'SESSION_SECRET',
  'API_KEY_SECRET',     // Required for HMAC-SHA256 API key hashing
];

const optionalVars = [
  'BREVO_API_KEY',       // Email disabled if missing (no-ops with log warnings)
  'R2_ACCOUNT_ID',       // Attachments disabled if missing
  'R2_ACCESS_KEY_ID',
  'R2_SECRET_ACCESS_KEY',
  'R2_BUCKET_NAME',
];
```

### Security Rules

- `.env` file MUST be listed in `.gitignore` — never committed to version control.
- Provide a `.env.example` file with all variable names and placeholder values (no real secrets).
- NEVER log environment variable values (especially secrets). Only log the variable name if validation fails ("Missing required environment variable: SESSION_SECRET").
- Use `process.env` reads centralized in a single config module. Never read `process.env` directly in route handlers or services.
- Session secret: minimum 32 characters, randomly generated. Use `crypto.randomBytes(32).toString('hex')`.
- API key secret: minimum 32 characters, randomly generated. Use `crypto.randomBytes(32).toString('hex')`.

---

## Session Security

### Server-Side Sessions

- **Storage**: PostgreSQL via `connect-pg-simple`.
- **Table**: `sessions` (auto-created by connect-pg-simple or via provided SQL schema).
- **Why not JWT**: Server-side sessions allow immediate revocation (logout, password change, admin force-logout). JWTs cannot be revoked without additional infrastructure (blocklists). For a CRM with sensitive data, session revocation is critical.

### Session Regeneration

- After successful login: call `req.session.regenerate()` to create a new session ID. This prevents session fixation attacks.
- Implementation:
  ```typescript
  req.session.regenerate((err) => {
    if (err) { /* handle error */ }
    req.session.userId = user.id;
    req.session.save((err) => {
      if (err) { /* handle error */ }
      // Send response
    });
  });
  ```

### Session Invalidation

- **On logout**: `req.session.destroy()` — removes session from PostgreSQL.
- **On password change**: Destroy all sessions for the user except the current one:
  ```sql
  DELETE FROM session WHERE sess->>'userId' = :userId AND sid != :currentSid
  ```
- **Admin force-logout**: Admin can invalidate all sessions for a specific user:
  ```sql
  DELETE FROM session WHERE sess->>'userId' = :targetUserId
  ```
- **Account deactivation**: When admin deactivates a user, all sessions for that user are destroyed immediately.

### Session Cleanup

- `connect-pg-simple` runs a cleanup job at a configurable interval (default: every 15 minutes).
- Cleanup deletes expired sessions from the `sessions` table.

### Session Data

The session stores minimal data:

```typescript
interface SessionData {
  userId: string;      // UUID of authenticated user
  createdAt: number;   // Timestamp when session was created
  ip: string;          // IP address at login (for audit)
  csrfToken: string;   // CSRF synchronizer token
}
```

- Do NOT store sensitive data (passwords, tokens) in the session.
- Do NOT store large objects in the session (user profiles, permissions). Look up fresh data from DB on each request using userId.
# SECTION 9: Milestones (Execution Plan)

Define exactly what gets built in each milestone, in what order. Each milestone must be fully testable and shippable independently. Milestones are ordered so that configurable entities (deal stages, activity types) are seeded before the features that consume them.

**Universal rules for every milestone:**
1. Detailed deliverables list with specific acceptance criteria
2. Specific test scenarios (not generic — every scenario names inputs and expected outputs)
3. Clear exit criteria describing what must be true before advancing
4. Regression requirement: "After this milestone's tests pass, run ALL previous milestone test suites to confirm nothing broke"
5. "What the user sees" — a plain-language description of the app's state after this milestone ships

---

## Milestone 0: Deployment & Scaffolding

**Goal:** Deploy bare-bones project infrastructure so it is live and accessible from the start. All subsequent development and testing happens against the live deployment.

**Deliverables:**

1. Use the **deploy-project** skill to set up the project infrastructure:
   - Create project directory
   - Set up PostgreSQL database (Docker container)
   - Configure web server (nginx) with SSL
   - Set up process management (PM2)
   - Allocate ports
   - Generate strong credentials for all services
   - Create `.env` with all required environment variables
2. Scaffold a bare-bones project:
   - Vite + React frontend showing a "CRM Coming Soon" placeholder page
   - Express backend with a single `GET /api/v1/health` endpoint returning `{ success: true, data: { status: "ok" } }`
   - Database connection verified (Knex connects to PostgreSQL successfully)
   - Frontend proxies API requests to backend
3. Verify the project is live at its public URL
4. Register the project and create Development Guidelines document per the deploy-project skill

**Testing:**

- Public URL loads the placeholder page over HTTPS
- `GET /api/v1/health` returns 200 with expected response
- Database connection works (health endpoint can optionally verify this)
- Process restarts automatically if killed (PM2 restart test)

**Exit Criteria:** The project is live and accessible at its public URL, showing a basic page. The health check endpoint works. The database is connected. All subsequent milestones build on top of this live deployment.

**What the user sees:** A "CRM Coming Soon" page at the project URL. The health endpoint responds.


## Milestone 1: Project Setup & Infrastructure

**Goal:** Project scaffolding, tooling, database connection, core configuration, and validation middleware.

**Deliverables:**

1. Initialize monorepo structure: `client/` (Vite + React + TypeScript), `server/` (Express + TypeScript), `shared/`
2. Configure TypeScript for all three packages (strict mode)
3. Set up Vite with React, Tailwind CSS, shadcn/ui
4. Set up Express with TypeScript compilation (ts-node-dev for dev)
5. Configure PostgreSQL connection with Knex
6. Set up centralized config module (`server/src/config/index.ts`) — validates all env vars at startup using Zod schemas
7. Create `.env.example` with ALL required and optional variables:
   ```
   DATABASE_URL=postgresql://user:pass@localhost:5432/crm
   SESSION_SECRET=<random-64-chars>
   API_KEY_SECRET=<random-64-chars>
   ADMIN_EMAIL=admin@example.com
   ADMIN_PASSWORD=<min-10-chars>
   BREVO_API_KEY=<optional>
   R2_ACCOUNT_ID=<optional>
   R2_ACCESS_KEY_ID=<optional>
   R2_SECRET_ACCESS_KEY=<optional>
   R2_BUCKET_NAME=<optional>
   R2_PUBLIC_URL=<optional>
   FRONTEND_URL=http://localhost:5173
   NODE_ENV=development
   PORT=3000
   ```
8. Set up winston logger with structured JSON logging
9. Configure helmet, CORS, express-rate-limit
10. Set up initial migration: create sessions table (connect-pg-simple)
11. Set up seed infrastructure
12. Configure build scripts: dev, build, start for both client and server
13. Set up shared types package
14. Create base API response utility functions
15. Create base error classes (`AppError`, `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`)
16. Create error handling middleware
17. Create request logging middleware
18. Create health check endpoint (`GET /api/v1/health`)
19. Set up Vitest for both frontend and backend (config files, test scripts in package.json)
20. Create Zod-based validation middleware for Express (`server/src/middleware/validate.ts`) — accepts a Zod schema, validates `req.body`/`req.query`/`req.params`, returns 400 with structured error details on failure

**Test Scenarios:**

- `GET /api/v1/health` returns `{ status: "ok", timestamp: <ISO string> }` with HTTP 200
- Server starts without errors when all required env vars are present
- Server fails to start with a clear error message when `DATABASE_URL` is missing
- Server fails to start with a clear error message when `SESSION_SECRET` is missing
- Server fails to start when `ADMIN_PASSWORD` is provided but shorter than 10 characters
- Server starts successfully when optional vars (`BREVO_API_KEY`, `R2_*`) are absent
- Database connection works: health check queries the DB and reports status
- CORS headers present on responses (check `Access-Control-Allow-Origin`)
- Rate limiting returns 429 after exceeding threshold
- Logger outputs structured JSON to stdout (verify format includes timestamp, level, message)
- Validation middleware returns 400 with `{ error: "Validation failed", details: [...] }` when request body fails schema
- Validation middleware passes request through when body matches schema
- Vitest runs and reports zero tests (baseline — confirms test infrastructure works)
- `npm run build` succeeds for both client and server without errors

**Exit Criteria:** Project compiles, server runs, connects to DB, health check works, validation middleware operational, Vitest configured and runnable.

**What the user sees:** Nothing visible in a browser yet — this milestone is pure infrastructure. Running `npm run dev` in the server starts Express on port 3000, and `GET http://localhost:3000/api/v1/health` returns a JSON success response.

---

## Milestone 2: Authentication & User Management

**Goal:** Complete auth system with login, logout, session management, password flows, CSRF protection, escalating lockout, user CRUD, and the app shell.

**Deliverables:**

1. Migration: create `users` table
2. Migration: create `audit_log` table
3. Auth middleware: `isAuthenticated`, `isAdmin`
4. CSRF protection middleware using synchronizer token pattern with `GET /api/v1/auth/csrf-token` endpoint
5. Auth routes + controller + service: login, logout, me, change-password, forgot-password, reset-password, update-profile
6. User routes + controller + service + validator: CRUD (admin only)
7. Password hashing with bcrypt (cost factor 12)
8. Password policy: minimum 10 characters
9. Session management with connect-pg-simple — cookie name `crm.sid`, 24-hour rolling expiry, regenerate session ID on login
10. Escalating lockout: 5 failed attempts = 1 min lock, 10 = 5 min, 15 = 15 min, 20+ = 60 min (tracked per email, reset on successful login)
11. Audit logging for all auth + user actions
12. Seed: create default admin user (email/password from env vars: `ADMIN_EMAIL`, `ADMIN_PASSWORD`)
13. Brevo email service setup (welcome email, password reset email) — graceful no-op if `BREVO_API_KEY` not set
14. Frontend: Login page (email + password form, error display, "Forgot Password" link)
15. Frontend: Forgot Password page (email input, success message)
16. Frontend: Reset Password page (new password + confirm, token from URL)
17. Frontend: App shell layout (sidebar navigation, header with user info/logout, main content area)
18. Frontend: Auth store (Zustand) — manages user state, auth checks, CSRF token
19. Frontend: `ProtectedRoute` wrapper (redirects to `/login` if not authenticated)
20. Frontend: `AdminRoute` wrapper (redirects to `/contacts` if not admin)
21. Frontend: User Management page (admin only — list users, create modal, edit modal, deactivate toggle)
22. Frontend: Profile page (edit own name, avatar URL)
23. Frontend: Change Password page (current password + new password + confirm)
24. After successful login, redirect to `/contacts` (dashboard is not built until Milestone 9)

**Test Scenarios:**

- Login with valid credentials returns 200, sets `crm.sid` cookie, session stored in DB
- Login with wrong password returns 401 with generic "Invalid email or password"
- Login with non-existent email returns 401 with same generic message (no user enumeration)
- After 5 failed logins for same email, 6th attempt returns 429 with "Account temporarily locked. Try again in 1 minute."
- After 10 failed logins, lock duration is 5 minutes
- Successful login resets the failed attempt counter
- Session persists across requests: `GET /api/v1/auth/me` returns user data with valid session
- Session expires after 24 hours of inactivity (set cookie max-age, verify expiry)
- Logout destroys session: subsequent `GET /api/v1/auth/me` returns 401
- Session ID is regenerated after login (compare session ID before and after)
- `isAuthenticated` middleware returns 401 for requests without session cookie
- `isAdmin` middleware returns 403 for authenticated non-admin users
- `GET /api/v1/auth/csrf-token` returns a token; POST without that token returns 403
- POST with valid CSRF token succeeds
- Admin can create a user with email, name, role; new user can login
- Admin can list all users, edit user details, deactivate a user
- Non-admin `POST /api/v1/users` returns 403
- Change password with incorrect current password returns 400
- Change password with valid current password succeeds; old sessions invalidated
- Password shorter than 10 characters rejected with validation error
- Forgot password with valid email triggers Brevo API call (mock or spy); returns 200
- Forgot password with invalid email still returns 200 (no user enumeration)
- Reset password with valid token changes password; token becomes single-use
- Reset password with expired token (>1 hour) returns 400
- Admin cannot deactivate themselves — returns 400
- Profile update changes name and avatar URL; returns updated user
- Audit log entries created for: login, logout, failed login, password change, user create, user update, user deactivate
- When `BREVO_API_KEY` is absent, email functions log `[EMAIL DISABLED] Would have sent: {template} to {email}` and do not throw

**Exit Criteria:** Full auth flow works end-to-end. Admin can manage users. CSRF protection active on all state-changing endpoints. Escalating lockout functional. App shell renders with sidebar navigation.

**Regression:** Run all Milestone 1 tests — health check, config validation, error handling.
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

**What the user sees:** Opening `http://localhost:5173` shows a login page. After logging in with admin credentials, the user sees the app shell with a sidebar (navigation links for Contacts, Companies, Deals, Activities, Meetings, Settings — most are placeholder pages for now). The header shows the logged-in user's name and a logout button. Admin can navigate to User Management and create/edit/deactivate users.


#### Smoke Test Commands
```bash
# Login
curl -s -X POST http://localhost:3000/api/v1/auth/login -H "Content-Type: application/json" -d '{"email":"admin@example.com","password":"AdminPass123!"}' -c cookies.txt

# Get current user
curl -s http://localhost:3000/api/v1/auth/me -b cookies.txt

# Get CSRF token  
curl -s http://localhost:3000/api/v1/auth/csrf-token -b cookies.txt

# List users (admin)
curl -s http://localhost:3000/api/v1/users -b cookies.txt

# Create user
curl -s -X POST http://localhost:3000/api/v1/users -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"email":"test@example.com","firstName":"Test","lastName":"User","role":"member","password":"TestPass123!"}'

# Unauthenticated request → 401
curl -s http://localhost:3000/api/v1/users

# Logout
curl -s -X POST http://localhost:3000/api/v1/auth/logout -H "X-CSRF-Token: {token}" -b cookies.txt
```

---

## Milestone 3: Contacts Module

**Goal:** Full CRUD, list with advanced filtering/sorting/pagination, detail view, import/export, bulk operations.

**Deliverables:**

1. Migration: create `contacts` table with all indexes
2. Migration: create `tags` table
3. Seed: contact sources in settings (e.g., "Website", "Referral", "Cold Call", "LinkedIn", "Trade Show", "Other")
4. Contact routes + controller + service + validator
5. Duplicate detection (warn on same email — do not block, just return warning flag)
6. Import endpoint (CSV parsing with BOM and CRLF handling, row-level validation, bulk create)
7. Export endpoint (CSV generation with currently-applied filters)
8. Bulk operations endpoint (change owner, change status, add/remove tags, delete)
9. Optimistic locking on updates (version column or `updated_at` check)
10. Frontend: Contacts list page (table, search bar, filters, sort, pagination, bulk action bar)
11. Frontend: Reusable `DataTable` component (generic, sortable columns, selectable rows, pagination controls)
12. Frontend: Reusable `FilterPanel` component (used across all entity list pages)
13. Frontend: Contact create/edit form (React Hook Form + Zod validation, all fields)
14. Frontend: Contact detail page (info section, tabs for activities, notes, attachments, deals)
15. Frontend: Import modal (CSV upload, preview parsed rows with error highlighting, confirm)
16. Frontend: Export button (exports with current filters applied)
17. Frontend: Bulk action bar (appears when rows selected, disappears when deselected)
18. Frontend: Tag management (inline add/remove on contact, create new tags on the fly)

**Test Scenarios:**

- Create contact with all fields returns 201 with full contact object
- Create contact with only required fields (first_name, last_name, email) returns 201
- Create contact missing required fields returns 400 with field-level errors
- Create contact with invalid email format returns 400
- Create contact with duplicate email returns 201 but response includes `warnings: ["A contact with this email already exists"]`
- `GET /api/v1/contacts` returns paginated list: default page=1, limit=25
- `GET /api/v1/contacts?page=2&limit=10` returns correct offset
- Search by partial name: `?search=john` matches "John Smith" and "Johnny Doe"
- Search by email: `?search=john@` matches contacts with that email prefix
- Search by phone: `?search=555` matches contacts with that phone substring
- Filter by status: `?status=active` returns only active contacts
- Filter by company: `?company_id=5` returns only contacts linked to company 5
- Filter by owner: `?owner_id=2` returns only contacts owned by user 2
- Filter by source: `?source=referral` returns only referral contacts
- Filter by tags: `?tags=vip,enterprise` returns contacts with either tag
- Filter by date range: `?created_after=2026-01-01&created_before=2026-03-01`
- Sort by `last_name` ascending (default), `created_at` descending, `last_contacted_at`
- Bulk change owner: select 3 contacts, change owner to user 2 — all 3 updated
- Bulk change status: select 5 contacts, set to "inactive" — all 5 updated
- Bulk add tag: select 2 contacts, add tag "VIP" — both tagged
- Bulk delete: select 2 contacts, delete — both soft-deleted or hard-deleted per design
- Import valid CSV (10 rows, all fields) — 10 contacts created, response: `{ imported: 10, errors: [] }`
- Import CSV with BOM (byte order mark) — parses correctly, no garbled first column
- Import CSV with CRLF line endings — parses correctly
- Import CSV with 3 valid rows and 2 invalid rows — response: `{ imported: 3, errors: [{ row: 4, field: "email", message: "Invalid email" }, { row: 5, ... }] }`
- Export CSV includes all contacts matching current filters; CSV opens correctly in Excel
- Contact detail page shows all related records (activities, notes, deals linked to this contact)
- Optimistic locking: update contact, then try updating with stale version — returns 409 Conflict
- Audit log entries for create, update, delete, import, bulk operations

**Exit Criteria:** Contacts fully functional — CRUD, list with all filters/sorts/search, import (handles BOM and CRLF), export, bulk operations, detail page with tabs, tag management.

**Regression:** Run all Milestone 1 and 2 test suites.
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

**What the user sees:** Clicking "Contacts" in the sidebar shows a list page with a search bar, filter panel, and data table. The user can create contacts via a form, import from CSV, export to CSV, select multiple contacts for bulk actions, click into a contact to see its detail page with tabs. Tags can be added/removed inline.


#### Smoke Test Commands
```bash
# Create contact
curl -s -X POST http://localhost:3000/api/v1/contacts -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"firstName":"John","lastName":"Doe","email":"john@example.com","status":"lead"}'

# List contacts
curl -s "http://localhost:3000/api/v1/contacts?page=1&limit=10" -b cookies.txt

# Search contacts
curl -s "http://localhost:3000/api/v1/contacts?search=john" -b cookies.txt

# Filter contacts
curl -s "http://localhost:3000/api/v1/contacts?status=lead,active" -b cookies.txt

# Get single contact
curl -s http://localhost:3000/api/v1/contacts/{id} -b cookies.txt

# Update contact
curl -s -X PUT http://localhost:3000/api/v1/contacts/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"firstName":"Jane","updatedAt":"{timestamp}"}'

# Export contacts
curl -s "http://localhost:3000/api/v1/contacts/export" -b cookies.txt -o contacts.csv

# Delete contact
curl -s -X DELETE http://localhost:3000/api/v1/contacts/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Verify deleted
curl -s http://localhost:3000/api/v1/contacts/{id} -b cookies.txt  # expect 404
```

---

## Milestone 4: Companies Module

**Goal:** Full CRUD, list with filtering/sorting, detail view, contact association, proper delete cascade behavior.

**Deliverables:**

1. Migration: create `companies` table with all indexes
2. Company routes + controller + service + validator
3. Contact-company association (set `company_id` on contacts)
4. Delete behavior: deleting a company sets `contacts.company_id = NULL` and `deals.company_id = NULL`; direct children (notes, attachments, activities on the company) CASCADE delete
5. Company metrics on detail view (number of contacts, number of deals, total deal value, average deal size)
6. Duplicate detection on domain (warn, do not block)
7. Optimistic locking
8. Frontend: Companies list page (reuse `DataTable`, reuse `FilterPanel`)
9. Frontend: Company create/edit form (all fields)
10. Frontend: Company detail page (info section, tabs for contacts, deals, activities, notes, attachments)
11. Frontend: Associate/disassociate contacts from company detail page

**Test Scenarios:**

- Create company with all fields returns 201
- Create company with only required fields (name) returns 201
- Create company with duplicate domain returns 201 with warning
- List companies with pagination, search by name, filter by industry, sort by name/created_at
- Company detail page shows correct metrics: create 3 contacts and 2 deals linked to company, verify counts and totals
- Associate contact with company: set `company_id` on contact, verify contact appears in company detail
- Disassociate contact: set `company_id = NULL`, verify contact removed from company detail
- Delete company with 2 contacts and 1 deal: contacts and deal get `company_id = NULL` (not deleted), notes/attachments on the company are deleted
- Delete company with no associations: deletes cleanly
- Optimistic locking: concurrent update returns 409
- Audit log entries for all CRUD operations

**Exit Criteria:** Companies fully functional with contact association, proper delete cascades, and detail page with metrics.

**Regression:** Run all Milestone 1, 2, and 3 test suites.

**What the user sees:** Clicking "Companies" in the sidebar shows a list page with search, filters, and data table. Users can create companies, click into a company to see its detail page with contacts, deals, and other tabs. Contacts can be linked/unlinked from the company detail view. Deleting a company orphans contacts and deals rather than deleting them.


#### Smoke Test Commands
```bash
# Create company
curl -s -X POST http://localhost:3000/api/v1/companies -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"name":"Acme Corp","industry":"Technology","website":"https://acme.com"}'

# List companies
curl -s "http://localhost:3000/api/v1/companies?page=1&limit=10" -b cookies.txt

# Search companies
curl -s "http://localhost:3000/api/v1/companies?search=acme" -b cookies.txt

# Get single company
curl -s http://localhost:3000/api/v1/companies/{id} -b cookies.txt

# Update company
curl -s -X PUT http://localhost:3000/api/v1/companies/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"name":"Acme Corporation","updatedAt":"{timestamp}"}'

# Delete company
curl -s -X DELETE http://localhost:3000/api/v1/companies/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Verify deleted
curl -s http://localhost:3000/api/v1/companies/{id} -b cookies.txt  # expect 404
```
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

---

## Milestone 5: Deals & Pipeline Module

**Goal:** Full CRUD, table + kanban views, deal stages (seeded from migration), stage change logic, statistics.

**Deliverables:**

1. Migration: create `deals` table with all indexes
2. Migration: create `deal_stages` table + seed default stages (Lead, Qualified, Proposal, Negotiation, Closed Won, Closed Lost) with display_order and default_probability
3. Deal routes + controller + service + validator
4. Deal statistics endpoint (total pipeline value, win rate, average deal size, deals by stage, forecast)
5. Stage change logic: moving to a stage auto-sets probability from `deal_stages.default_probability`; moving to Closed Won requires `closed_at` date and sets probability to 100; moving to Closed Lost requires `closed_at` date and `lost_reason`, sets probability to 0
6. Frontend: Deals list page — table view (reuse `DataTable`)
7. Frontend: Deals kanban view (drag-and-drop columns by stage)
8. Frontend: View toggle (table/kanban) with preference persisted in localStorage
9. Frontend: Deal create/edit form (contact, company, stage, value, expected close date, etc.)
10. Frontend: Deal detail page (pipeline stage visual showing progress, info, related records tabs)
11. Frontend: Deal stage change from kanban (drag card between columns) and from detail page (click stage in visual)
12. Frontend: Deal statistics display (cards + charts on deals list page)

**Test Scenarios:**

- Create deal with all fields returns 201; stage defaults to first stage if not specified
- Create deal with only required fields returns 201
- List deals in table view with pagination, filter by stage, owner, company, value range, expected close date range
- Sort by value descending, expected_close_date ascending, created_at
- Kanban view renders one column per active stage, cards in correct columns
- Drag card from "Lead" to "Qualified": deal's stage updates, probability auto-set to Qualified's default_probability
- Drag card to "Closed Won": prompt appears for closed_at date; after confirming, probability set to 100
- Drag card to "Closed Lost": prompt appears for closed_at and lost_reason; after confirming, probability set to 0
- Click stage in deal detail pipeline visual: same stage-change logic applies
- Statistics endpoint: seed 10 deals across stages — verify total pipeline value, win rate (closed_won / (closed_won + closed_lost)), average deal size
- Deal linked to contact and company: deleting the contact sets `deal.contact_id = NULL`; deleting the company sets `deal.company_id = NULL`
- Audit log entries for create, update, delete, stage changes

**Exit Criteria:** Deals module fully functional with both table and kanban views, drag-and-drop stage changes, auto-probability, closed handling prompts, and statistics.

**Regression:** Run all Milestone 1-4 test suites.

**What the user sees:** Clicking "Deals" in the sidebar shows deals in either table or kanban view (toggle in top-right). The kanban shows columns for each pipeline stage with deal cards that can be dragged between columns. Moving a deal to Closed Won or Closed Lost prompts for additional information. Statistics cards show pipeline value, win rate, and forecast above the deal list.


#### Smoke Test Commands
```bash
# Create deal
curl -s -X POST http://localhost:3000/api/v1/deals -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"title":"Big Deal","value":50000,"contactId":"{contactId}","companyId":"{companyId}"}'

# List deals
curl -s "http://localhost:3000/api/v1/deals?page=1&limit=10" -b cookies.txt

# Filter deals by stage
curl -s "http://localhost:3000/api/v1/deals?stage=lead,qualified" -b cookies.txt

# Get single deal
curl -s http://localhost:3000/api/v1/deals/{id} -b cookies.txt

# Update deal (stage change)
curl -s -X PUT http://localhost:3000/api/v1/deals/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"stageId":"{stageId}","updatedAt":"{timestamp}"}'

# Get deal statistics
curl -s http://localhost:3000/api/v1/deals/stats -b cookies.txt

# Delete deal
curl -s -X DELETE http://localhost:3000/api/v1/deals/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Verify deleted
curl -s http://localhost:3000/api/v1/deals/{id} -b cookies.txt  # expect 404
```
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

---

## Milestone 6: Activities & Meetings Module

**Goal:** Activity feed, task management, meeting CRUD with calendar view, side effects on related entities, email notifications.

**Deliverables:**

1. Migration: create `activities` table
2. Migration: create `activity_types` table + seed defaults (Call, Email, Meeting, Note, Task, Follow-up)
3. Migration: create `meetings` table + `meeting_attendees` table
4. Activity routes + controller + service + validator
5. Meeting routes + controller + service + validator
6. Activity side effects: creating an activity of type Call, Email, or Meeting on a contact updates that contact's `last_contacted_at` to the activity's date
7. Meeting email notifications (create, update, cancel) via Brevo — graceful no-op if Brevo unavailable: logs `[EMAIL DISABLED] Would have sent: meeting_{action} to {email}`
8. Frontend: Global activity feed page (all activities, filterable by type, user, date range, entity)
9. Frontend: Activity timeline component (reusable — embedded in contact, company, deal detail pages)
10. Frontend: Log Activity modal (type selector from activity_types, form fields, link to entity)
11. Frontend: My Tasks page (sections: Overdue, Today, Upcoming, Completed; task = activity of type "Task" with due date)
12. Frontend: Task quick-complete toggle (checkbox marks task as completed with timestamp)
13. Frontend: Meetings list page
14. Frontend: Meeting calendar view (month grid showing meeting dots, click day to see day detail, click meeting to navigate to detail)
15. Frontend: Meeting create/edit form (title, date/time, duration, location, description, attendee management with user search)
16. Frontend: Meeting detail page (info, attendee list, related entity links)

**Test Scenarios:**

- Create activity of type "Call" on contact ID 5: returns 201, contact 5's `last_contacted_at` updated to activity date
- Create activity of type "Task" with due_date: returns 201, appears in My Tasks under correct section
- Create activity of type "Email" with no contact link: returns 201, no side effect triggered
- List activities for a contact: `GET /api/v1/contacts/5/activities` returns only activities linked to contact 5
- Filter activities by type: `?type=call` returns only calls
- Filter activities by date range: `?from=2026-03-01&to=2026-03-31`
- Mark task as completed: `PATCH /api/v1/activities/:id` with `{ completed: true }` — sets `completed_at` to now
- My Tasks page: create 1 overdue task (due yesterday), 1 due today, 1 due tomorrow, 1 completed — verify each appears in correct section
- Create meeting with 3 attendees: returns 201, Brevo API called 3 times (one per attendee) or logs disabled message
- Update meeting time: Brevo notified with update template
- Cancel meeting: Brevo notified with cancellation template
- Calendar view: create meetings on March 10 and March 20 — both dates show indicators, clicking date shows meetings
- Meeting detail shows all attendees with their response status
- Attendee management: add attendee to existing meeting, remove attendee — both work correctly
- Audit log entries for activity CRUD, meeting CRUD, task completion

**Exit Criteria:** Activities and meetings fully functional. Activity feed renders on entity detail pages. Tasks manageable from My Tasks. Calendar renders meetings. Email notifications work or gracefully degrade.

**Regression:** Run all Milestone 1-5 test suites.

**What the user sees:** Contact/company/deal detail pages now show an activity timeline tab with logged activities. A "Log Activity" button opens a modal to record calls, emails, tasks, etc. "My Tasks" in the sidebar shows a task dashboard organized by urgency. "Meetings" in the sidebar shows a list and calendar view. Creating a meeting sends email notifications to attendees (or logs a message if Brevo is not configured).


#### Smoke Test Commands
```bash
# Create activity
curl -s -X POST http://localhost:3000/api/v1/activities -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"type":"call","subject":"Follow-up call","contactId":"{contactId}","description":"Discussed next steps"}'

# List activities
curl -s "http://localhost:3000/api/v1/activities?page=1&limit=10" -b cookies.txt

# Filter activities by type
curl -s "http://localhost:3000/api/v1/activities?type=call" -b cookies.txt

# Get single activity
curl -s http://localhost:3000/api/v1/activities/{id} -b cookies.txt

# Update activity (mark task complete)
curl -s -X PUT http://localhost:3000/api/v1/activities/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"completed":true}'

# Delete activity
curl -s -X DELETE http://localhost:3000/api/v1/activities/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Create meeting
curl -s -X POST http://localhost:3000/api/v1/meetings -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"title":"Product Demo","startTime":"2026-04-01T10:00:00Z","endTime":"2026-04-01T11:00:00Z","location":"Zoom","attendeeIds":["{userId}"]}'

# List meetings
curl -s "http://localhost:3000/api/v1/meetings?page=1&limit=10" -b cookies.txt

# Get single meeting
curl -s http://localhost:3000/api/v1/meetings/{id} -b cookies.txt

# Update meeting
curl -s -X PUT http://localhost:3000/api/v1/meetings/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"title":"Updated Demo","updatedAt":"{timestamp}"}'

# Delete meeting
curl -s -X DELETE http://localhost:3000/api/v1/meetings/{id} -H "X-CSRF-Token: {token}" -b cookies.txt
```
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

---

## Milestone 7: Notes & Attachments Module

**Goal:** Notes with markdown support and pinning, attachments with R2 storage (graceful degradation without R2).

**Deliverables:**

1. Migration: create `notes` table
2. Migration: create `attachments` table
3. Note routes + controller + service + validator
4. Attachment routes + controller + service + validator
5. R2 storage service (upload, download via presigned URL, delete) — graceful degradation: if R2 credentials not set, all file operations return `{ error: "File storage not configured" }` with HTTP 503
6. Multer configuration for file uploads (memory storage, pipe to R2)
7. File type and size validation: server rejects files > 25MB, rejects disallowed MIME types (executables, scripts); client validates before upload
8. Maximum 50 attachments per entity enforced server-side
9. Frontend: Notes component (reusable — embedded in contact, company, deal detail pages)
10. Frontend: Inline note creation with markdown editor (basic: bold, italic, headings, lists, links, code blocks)
11. Frontend: Note edit/delete inline (only creator or admin can edit/delete)
12. Frontend: Pin/unpin notes (pinned notes sort to top)
13. Frontend: Attachments component (reusable — embedded in contact, company, deal detail pages)
14. Frontend: Upload button with drag-and-drop zone
15. Frontend: Attachment list (filename, size, type icon, upload date, uploader, download button, delete button)
16. Frontend: Upload progress indicator
17. Frontend: Markdown rendering for notes (rendered view with raw toggle)

**Test Scenarios:**

- Create note on contact: `POST /api/v1/contacts/5/notes` with `{ content: "## Hello\n**bold** text" }` returns 201
- Get notes for contact: returns notes array, pinned notes first, then by created_at desc
- Edit note as creator: returns 200 with updated content
- Edit note as different non-admin user: returns 403
- Edit note as admin (not creator): returns 200 (admins can edit any note)
- Delete note as creator: returns 204
- Pin note: `PATCH /api/v1/notes/:id` with `{ is_pinned: true }` — note appears first in list
- Unpin note: sets `is_pinned: false` — note returns to chronological position
- Markdown rendering: note with `## Heading` renders as `<h2>`, `**bold**` renders as `<strong>`
- Upload 5MB PDF to contact: returns 201 with attachment record including presigned download URL
- Upload 30MB file: returns 400 "File size exceeds 25MB limit"
- Upload .exe file: returns 400 "File type not allowed"
- Download attachment: presigned URL returns file with correct content-type
- Delete attachment: removes from R2 and database; subsequent download returns 404
- Upload 51st attachment to an entity that already has 50: returns 400 "Maximum 50 attachments per entity"
- When R2 is not configured: upload returns 503 `{ error: "File storage not configured" }`
- When R2 is not configured: CRM is otherwise fully functional (all other pages work, attachment upload shows clear "not configured" message)
- Audit log entries for note and attachment operations

**Exit Criteria:** Notes and attachments fully functional. Markdown renders correctly. File operations work with R2 or gracefully degrade without it.

**Regression:** Run all Milestone 1-6 test suites.

**What the user sees:** Contact, company, and deal detail pages now have Notes and Attachments tabs. Users can write markdown notes, pin important ones to the top, and edit/delete their own notes. Users can upload files via button or drag-and-drop, see upload progress, download files, and delete attachments. If R2 is not configured, the attachments section shows a clear message that file storage is not set up.


#### Smoke Test Commands
```bash
# Create note on contact
curl -s -X POST http://localhost:3000/api/v1/notes -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"entityType":"contact","entityId":"{contactId}","content":"## Meeting Notes\n**Key takeaways:** discussed pricing"}'

# List notes for entity
curl -s "http://localhost:3000/api/v1/notes?entityType=contact&entityId={contactId}" -b cookies.txt

# Update note
curl -s -X PUT http://localhost:3000/api/v1/notes/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"content":"Updated note content","updatedAt":"{timestamp}"}'

# Pin note
curl -s -X PATCH http://localhost:3000/api/v1/notes/{id} -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"isPinned":true}'

# Delete note
curl -s -X DELETE http://localhost:3000/api/v1/notes/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Upload attachment
curl -s -X POST http://localhost:3000/api/v1/attachments/upload -H "X-CSRF-Token: {token}" -b cookies.txt -F "file=@test.pdf" -F "entityType=contact" -F "entityId={contactId}"

# List attachments for entity
curl -s "http://localhost:3000/api/v1/attachments?entityType=contact&entityId={contactId}" -b cookies.txt

# Download attachment
curl -s http://localhost:3000/api/v1/attachments/{id}/download -b cookies.txt -o downloaded_file.pdf

# Delete attachment
curl -s -X DELETE http://localhost:3000/api/v1/attachments/{id} -H "X-CSRF-Token: {token}" -b cookies.txt
```
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

---

## Milestone 8: Settings, API Keys & Email Templates

**Goal:** Admin settings panel with deal stage and activity type management, API key generation/authentication, email template management, audit log viewer.

**Deliverables:**

1. Migration: create `api_keys` table
2. Migration: create `email_templates` table + seed defaults (welcome, password_reset, meeting_invite, meeting_update, meeting_cancel)
3. Migration: create `email_log` table
4. Settings routes + controller + service (CRUD for settings key-value pairs)
5. Deal stages management: reorder, add, edit, delete (with guard: minimum 2 stages, cannot delete stage with existing deals)
6. Activity types management: add, edit, deactivate (cannot delete type with existing activities — deactivate instead)
7. Email template management: CRUD with variable preview (e.g., `{{user_name}}`, `{{reset_link}}`)
8. API key routes + controller + service: generate (returns key ONCE), list (masked), revoke
9. API key authentication middleware: `Authorization: Bearer <api_key>` — validates key, attaches user context, respects key permissions
10. Audit log viewer with filtering by action, user, entity type, date range
11. Frontend: Settings page with tabs — General, Deal Stages, Activity Types, Sources, Email Templates, API Keys, Audit Log
12. Frontend: Deal Stages tab (drag-to-reorder list, add/edit/delete with confirmation, shows deal count per stage)
13. Frontend: Activity Types tab (list, add/edit/deactivate)
14. Frontend: Email Templates tab (list, edit with variable insertion, preview rendered output)
15. Frontend: API Keys tab (generate button, list with masked keys, revoke with confirmation)
16. Frontend: API key generation modal (shows full key ONCE with copy button, warns it cannot be shown again)
17. Frontend: Audit Log tab (filterable table of audit entries)

**Test Scenarios:**

- Get all settings: returns key-value pairs
- Update setting: `PUT /api/v1/settings/company_name` with `{ value: "Acme Corp" }` returns 200
- Deal stages: reorder stages (swap position of stage 1 and stage 2) — verify new order persists
- Deal stages: add new stage "Discovery" at position 2 — verify it appears between existing stages
- Deal stages: attempt to delete stage with 3 existing deals — returns 400 "Cannot delete stage with existing deals. Move or delete the 3 deals first."
- Deal stages: attempt to delete when only 2 stages remain — returns 400 "Minimum 2 stages required"
- Deal stages: delete stage with 0 deals when 3+ stages exist — succeeds
- Activity types: add new type "Demo" — appears in activity type dropdown in Log Activity modal
- Activity types: deactivate type "Follow-up" — no longer appears in dropdown, existing activities retain their type label
- API key generation: `POST /api/v1/api-keys` with `{ name: "Integration", permissions: ["contacts:read", "contacts:write"] }` returns `{ id, key: "crm_...", name, permissions, created_at }` — key is a 64-char random string prefixed with `crm_`
- API key authentication: `GET /api/v1/contacts` with `Authorization: Bearer crm_<key>` returns 200 with contacts
- API key without required permission: key with only `contacts:read` tries `POST /api/v1/contacts` — returns 403
- API key revocation: revoke key, then use it — returns 401 "API key revoked or invalid"
- API key rate limiting: API keys have separate rate limit (configurable, default 100 req/min)
- Email template CRUD: edit "password_reset" template body, verify updated content returned
- Email template preview: `POST /api/v1/email-templates/preview` with template ID and sample variables — returns rendered HTML
- Audit log viewer: filter by action="login" returns only login entries; filter by date range works; pagination works
- Email log: after sending an email, entry appears in email_log with status, recipient, template used

**Exit Criteria:** All admin features functional. Deal stages and activity types manageable from UI. API keys enable external programmatic access. Email templates editable. Audit log browsable with filters.

**Regression:** Run all Milestone 1-7 test suites.

**What the user sees:** The Settings page (admin only) has tabs for managing deal stages (drag to reorder), activity types, contact sources, email templates (edit with live preview), API keys (generate, copy, revoke), and a full audit log. Non-admin users see a simplified settings page without admin tabs.


#### Smoke Test Commands
```bash
# Get settings
curl -s http://localhost:3000/api/v1/settings -b cookies.txt

# Update setting
curl -s -X PUT http://localhost:3000/api/v1/settings/company_name -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"value":"Acme Corp"}'

# Get deal stages
curl -s http://localhost:3000/api/v1/settings/deal-stages -b cookies.txt

# Update deal stages
curl -s -X PUT http://localhost:3000/api/v1/settings/deal-stages -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"stages":[{"id":1,"name":"Lead","displayOrder":1},{"id":2,"name":"Discovery","displayOrder":2}]}'

# Generate API key
curl -s -X POST http://localhost:3000/api/v1/api-keys -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"name":"Integration","permissions":["contacts:read","contacts:write"]}'

# List API keys
curl -s http://localhost:3000/api/v1/api-keys -b cookies.txt

# Use API key to access contacts
curl -s http://localhost:3000/api/v1/contacts -H "Authorization: Bearer {apiKey}"

# Revoke API key
curl -s -X DELETE http://localhost:3000/api/v1/api-keys/{id} -H "X-CSRF-Token: {token}" -b cookies.txt

# Get email templates
curl -s http://localhost:3000/api/v1/email-templates -b cookies.txt

# Preview email template
curl -s -X POST http://localhost:3000/api/v1/email-templates/preview -H "Content-Type: application/json" -H "X-CSRF-Token: {token}" -b cookies.txt -d '{"templateId":1,"variables":{"user_name":"Test User"}}'

# Get audit log
curl -s "http://localhost:3000/api/v1/audit-log?page=1&limit=10" -b cookies.txt
```
- **Regression:** Re-run all smoke test commands from previous milestones to verify no regressions

---

## Milestone 9: Dashboard, Global Search & Polish

**Goal:** Dashboard with metrics and charts, global search, UI polish across all pages, keyboard shortcuts, performance optimization, comprehensive E2E and regression testing.

**Deliverables:**

1. Dashboard routes + controller + service (aggregation queries across all entities)
2. Frontend: Dashboard page as the post-login landing page (update redirect from `/contacts` to `/dashboard`)
   - Metrics cards: total contacts, total companies, open deals (count + value), tasks due today, overdue tasks
   - Pipeline chart: bar or funnel chart showing deal count and value per stage
   - Recent activities feed (last 10 across all entities)
   - Upcoming meetings (next 7 days)
   - My open tasks (due soonest first)
   - Deal forecast: expected revenue by month for next 3 months (sum of deal values weighted by probability)
   - Activity stats: activities logged per day/week for the last 30 days
3. Frontend: Simple chart components (bar chart, donut chart — lightweight, SVG-based or canvas-based, no heavy chart library)
4. Global search endpoint: `GET /api/v1/search?q=<term>` searches contacts (name, email), companies (name, domain), deals (title) — returns categorized results, ranked: exact match > starts with > contains
5. Frontend: Global search bar in header (always visible, `Ctrl+K` shortcut to focus)
   - Debounced input (300ms)
   - Dropdown with categorized results (Contacts, Companies, Deals sections)
   - Click result to navigate to detail page
   - Enter on highlighted result navigates
6. Frontend: Search results page (full page, tabbed by entity type, for when user presses Enter in search bar)
7. UI polish across ALL pages:
   - Loading states: skeleton loaders on all list pages and detail pages
   - Empty states: helpful messages with action buttons ("No contacts yet. Create your first contact.")
   - Error states: friendly error messages with retry buttons
8. UI polish: responsive design — sidebar collapses to hamburger on mobile, tables become card layouts on small screens
9. UI polish: toast notifications for all CUD operations (success: green, error: red, auto-dismiss after 5s)
10. UI polish: confirmation modals for all destructive actions (delete contact, revoke API key, etc.)
11. UI polish: breadcrumbs on all pages (e.g., Contacts > John Smith)
12. Keyboard shortcuts: `Ctrl+K` opens search, `Esc` closes any open modal/dropdown
13. Performance: review all database queries, add missing indexes, optimize N+1 queries
14. Performance: target < 200ms response time for all list endpoints with 1000 records
15. Comprehensive E2E test suite (see scenarios below)
16. Full regression test suite: run ALL milestone test suites in sequence

**Test Scenarios:**

- Dashboard: seed database with 50 contacts, 20 companies, 30 deals (various stages), 100 activities, 10 meetings — verify all metrics cards show correct numbers
- Dashboard: pipeline chart shows correct deal counts per stage
- Dashboard: forecast calculation matches sum of (deal_value * probability / 100) grouped by expected_close_date month
- Dashboard: recent activities shows last 10 activities in chronological order
- Dashboard: upcoming meetings shows meetings in next 7 days only
- Global search: `?q=john` returns contacts named John, companies named "Johnson Inc", deals titled "Johnson Deal"
- Global search: exact match "John Smith" ranked higher than "Johnny" which is ranked higher than "Smithson Johnson"
- Global search: empty query returns empty results (not all records)
- Global search: minimum 2 characters required, returns 400 for single character
- `Ctrl+K` focuses the search bar from any page
- `Esc` closes the search dropdown, closes modals, closes mobile sidebar
- Every list page shows skeleton loader during fetch
- Every list page shows empty state when no records exist
- Every list page shows error state with retry button on API failure (simulate with network error)
- Mobile viewport (375px width): sidebar becomes hamburger menu, tables become card layouts, forms are full-width
- Toast appears on contact create (green "Contact created"), contact delete (green "Contact deleted"), API error (red with error message)
- Confirmation modal appears before: deleting any entity, revoking API key, deactivating user, bulk delete
- Breadcrumbs: Contacts page shows "Contacts", contact detail shows "Contacts > John Smith", edit shows "Contacts > John Smith > Edit"
- E2E flow — full lifecycle: create company "Acme" > create contact "John" linked to Acme > create deal "Big Deal" linked to John and Acme > log activity "Call" on deal > add note on contact > upload attachment on deal > verify dashboard shows updated metrics > search for "Acme" and find company, contact's company, and deal
- E2E flow — deal lifecycle: create deal > move through Lead > Qualified > Proposal > Negotiation > Closed Won (provide close date) > verify probability is 100, dashboard win rate updated
- E2E flow — user lifecycle: admin creates user "Sales Rep" > sales rep logs in > changes password > creates a contact > logs an activity > admin deactivates sales rep > sales rep cannot login
- E2E flow — API key: admin generates API key with contacts:read permission > use key to `GET /api/v1/contacts` (200) > try `POST /api/v1/contacts` (403) > admin revokes key > `GET /api/v1/contacts` returns 401
- Performance: with 1000 contacts in DB, `GET /api/v1/contacts` responds in < 200ms
- Performance: with 500 deals in DB, `GET /api/v1/deals` responds in < 200ms
- Performance: dashboard endpoint with full dataset responds in < 500ms
- Performance: global search responds in < 300ms

**Exit Criteria:** CRM is complete, polished, and performant. Dashboard provides actionable overview. Global search works across all entities. All pages have loading/empty/error states. Responsive on mobile. All E2E flows pass. All regression tests pass.

**Regression:** Run ALL test suites from Milestones 1-8 as a final regression gate.

**What the user sees:** After login, the user lands on a dashboard with metrics cards, pipeline chart, recent activities, upcoming meetings, and tasks. A search bar in the header (also activated with `Ctrl+K`) provides instant search across contacts, companies, and deals. All pages have polished loading states, empty states with helpful CTAs, and error states with retry. Toast notifications confirm every action. Destructive actions require confirmation. The app works on mobile with a collapsible sidebar and responsive layouts. The CRM is production-ready.


#### Smoke Test Commands
```bash
# Dashboard metrics
curl -s http://localhost:3000/api/v1/dashboard/metrics -b cookies.txt

# Dashboard pipeline
curl -s http://localhost:3000/api/v1/dashboard/pipeline -b cookies.txt

# Dashboard recent activities
curl -s http://localhost:3000/api/v1/dashboard/recent-activities -b cookies.txt

# Dashboard upcoming meetings
curl -s http://localhost:3000/api/v1/dashboard/upcoming-meetings -b cookies.txt

# Dashboard my tasks
curl -s http://localhost:3000/api/v1/dashboard/my-tasks -b cookies.txt

# Dashboard deal forecast
curl -s http://localhost:3000/api/v1/dashboard/deal-forecast -b cookies.txt

# Dashboard activity stats
curl -s http://localhost:3000/api/v1/dashboard/activity-stats -b cookies.txt

# Global search
curl -s "http://localhost:3000/api/v1/search?q=acme" -b cookies.txt

# Global search - minimum query length
curl -s "http://localhost:3000/api/v1/search?q=a" -b cookies.txt  # expect 400

# Health check (regression)
curl -s http://localhost:3000/api/v1/health
```
- **Regression:** Re-run ALL smoke test commands from ALL previous milestones to verify no regressions

---
---

# SECTION 10: AI Agent Execution Instructions

## Role

You are a **Project Manager Agent**. You are responsible for executing this PRD from start to finish by delegating work to specialized agents and ensuring quality at every step.

## Execution Model

You will use the **Delegation Skill** to delegate milestones to Developer agents. You manage, review, test, and advance through milestones sequentially.

## Before You Start — User Customization Phase

**CRITICAL: Before executing any milestone, you MUST complete the customization phase.**

1. Read the "Before You Start" section (Section 1) of this PRD
2. Present ALL customization questions to the user
3. Wait for user responses
4. Update the PRD context with user's answers (store in your session memory)
5. Adjust any milestone details based on user customization (e.g., custom fields, stages, branding)
6. Confirm with user: "I've captured your customizations. Here's what I'll build: [summary]. Shall I proceed?"
7. Only after user confirms -> begin Milestone 1

After user customization is complete, the FIRST execution step is **Milestone 0 (Deployment & Scaffolding)**. Use the **deploy-project** skill to set up infrastructure. The project must be live and accessible at its public URL before any feature development begins. All subsequent testing happens against this live deployment, allowing both agents and the user to see the CRM come to life progressively.

## Conflict Resolution Rules

If you encounter contradictory information between sections of this PRD, follow this priority order:

1. **Section 5 (Database Schema)** — canonical for all table/column definitions, data types, constraints, and relationships
2. **Section 8 (Security)** — canonical for all security parameters (session config, password policy, rate limits, CSRF, lockout)
3. **Section 7 (API Design)** — canonical for endpoint paths, HTTP methods, request/response formats, and status codes
4. **Section 4 (Coding Best Practices)** — canonical for patterns, conventions, file structure, and naming
5. **Section 6 (Features)** — canonical for business logic, UI behavior, and user-facing functionality
6. **Section 3 (Tech Stack)** — canonical for technology choices, libraries, and versions
7. **Section 1 (User Customization)** — canonical for user-specific overrides (branding, custom fields, custom stages)

Document any conflict resolution decisions in a `DECISIONS.md` file in the project root. Each entry should include: the conflict found, which sections disagreed, which section's definition was used, and why.

## Testing Framework

All milestones must follow these testing standards:

- **Test runner:** Vitest for all tests (backend and frontend)
- **Backend integration tests:** Use `supertest` to test Express routes. Use a separate test database (e.g., `crm_test`). Wrap each test in a transaction and roll back after — no test data persists between tests.
- **Frontend tests:** Use `@testing-library/react` for component tests. Test user interactions, not implementation details.
- **E2E tests (Milestone 9):** Test scripts that call API endpoints with `supertest` to validate full user flows end-to-end. These run against the full application stack with a test database.
- **Test file naming:** `*.test.ts` colocated with the source file (e.g., `contact.service.ts` and `contact.service.test.ts` in the same directory)
- **Coverage target:** Each milestone must achieve > 80% test coverage on service files (`*.service.ts`). Use Vitest's `--coverage` flag to verify.
- **Test data:** Use factory functions or fixtures to create test data. Never rely on specific database IDs — always create needed data in setup.

## Commit Strategy

Follow this commit discipline throughout the build:

- **Granularity:** One commit per logical unit of work within a milestone (e.g., "migration + model", "routes + controller + service", "frontend list page", "tests for contacts CRUD"). Avoid giant single-commit milestones.
- **Commit message format:** Use conventional commits:
  - `feat(contacts): add CRUD routes and service`
  - `fix(auth): correct session regeneration on login`
  - `test(deals): add integration tests for stage change logic`
  - `chore(config): add Vitest configuration`
- **Milestone tags:** After all tests pass for a milestone, tag the commit: `git tag milestone-N-complete` (e.g., `git tag milestone-1-complete`)
- **No broken commits:** Every commit should leave the application in a buildable, runnable state. Never commit code that does not compile.

## Error Recovery

If a delegation fails or produces incorrect code:

1. **Read the error details carefully.** Understand whether it is a syntax error, logic error, missing dependency, or misunderstood requirement.
2. **Send a targeted fix request** to the developer agent with:
   - The exact error message or test failure output
   - The file(s) that need to change
   - Specific instructions on what to fix (not "fix the bug" — say "the `findById` method in `contact.service.ts` is not awaiting the Knex query; add `await` on line 45")
3. **Three-strike rule:** If 3 attempts fail on the same issue, stop attempting that specific task. Document the issue in `DECISIONS.md` with the error details, what was tried, and move on to the next task. Return to the blocked task after completing the current milestone's other deliverables, with fresh context.
4. **Never delete or rewrite entire modules.** Fix incrementally. If a service file has a bug, fix the specific function — do not regenerate the entire file, which risks losing other correct work.
5. **Check prerequisites before blaming code:** Is the database running? Are migrations applied? Are dependencies installed? Is the test database created?

## External Service Handling

Brevo (email) and R2 (file storage) are OPTIONAL external services. The CRM must be fully functional without them.

**When Brevo credentials (`BREVO_API_KEY`) are not provided:**
- All email-sending functions become no-ops
- They log: `[EMAIL DISABLED] Would have sent: {template_name} to {recipient_email}` at the `info` log level
- They return success (do not throw or return errors) — the calling code should not need to know whether email is configured
- The email_log table still gets entries with `status: "disabled"`
- All features that trigger emails (password reset, welcome email, meeting notifications) work normally from the user's perspective — they just don't send actual emails

**When R2 credentials (`R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`) are not provided:**
- Upload endpoint returns HTTP 503 with `{ error: "File storage not configured. Set R2 environment variables to enable file uploads." }`
- Download endpoint returns HTTP 503 with same message
- Delete endpoint returns HTTP 503 with same message
- Frontend attachment component shows a clear message: "File storage is not configured. Contact your administrator to enable file uploads."
- All other CRM features work normally — the app does not crash or show errors on pages that include the attachment component

**Testing implication:** The test suite must pass with AND without these credentials. Tests for email and file features should use mocks/spies when credentials are absent and verify the graceful degradation behavior.

## Progress Tracking

Before starting execution, create a master checklist:

```markdown
# CRM Build Progress

## Milestone 0: Deployment & Scaffolding
- [ ] Deploy project infrastructure (deploy-project skill)
- [ ] Set up PostgreSQL database (Docker container)
- [ ] Configure web server (nginx) with SSL
- [ ] Set up process management (PM2)
- [ ] Scaffold bare-bones project (Vite + React placeholder, Express health endpoint)
- [ ] Database connection verified (Knex connects to PostgreSQL)
- [ ] Project live at public URL
- [ ] Register project and create Development Guidelines
- [ ] Milestone 0 COMPLETE

## Milestone 1: Project Setup & Infrastructure
- [ ] Project scaffolding (monorepo: client/, server/, shared/)
- [ ] TypeScript configuration (strict mode, all packages)
- [ ] Vite + React + Tailwind + shadcn setup
- [ ] Express + TypeScript setup
- [ ] PostgreSQL + Knex setup
- [ ] Centralized config module (Zod-validated env vars)
- [ ] .env.example with all variables documented
- [ ] Logger setup (winston, structured JSON)
- [ ] Security middleware (helmet, CORS, rate limit)
- [ ] Session table migration
- [ ] Seed infrastructure
- [ ] Build scripts (dev, build, start)
- [ ] Shared types package
- [ ] Base API response utilities
- [ ] Error classes + error handling middleware
- [ ] Request logging middleware
- [ ] Health check endpoint
- [ ] Vitest setup (frontend + backend)
- [ ] Zod validation middleware
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Milestone 1 COMPLETE

## Milestone 2: Authentication & User Management
- [ ] Users table migration
- [ ] Audit log table migration
- [ ] Auth middleware (isAuthenticated, isAdmin)
- [ ] CSRF protection (synchronizer token, GET /api/v1/auth/csrf-token)
- [ ] Auth routes + controller + service
- [ ] User routes + controller + service + validator
- [ ] Password hashing (bcrypt, cost 12, min 10 chars)
- [ ] Session management (crm.sid, 24h rolling, regenerate on login)
- [ ] Escalating lockout (5/10/15/20+ failures)
- [ ] Audit logging for auth + user actions
- [ ] Seed: default admin user
- [ ] Brevo email service (graceful no-op)
- [ ] Frontend: Login page
- [ ] Frontend: Forgot Password + Reset Password pages
- [ ] Frontend: App shell (sidebar, header, main content)
- [ ] Frontend: Auth store (Zustand)
- [ ] Frontend: ProtectedRoute + AdminRoute
- [ ] Frontend: User Management page (admin)
- [ ] Frontend: Profile page
- [ ] Frontend: Change Password page
- [ ] Post-login redirect to /contacts
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1 tests pass
- [ ] Milestone 2 COMPLETE

## Milestone 3: Contacts Module
- [ ] Contacts table migration with indexes
- [ ] Tags table migration
- [ ] Seed: contact sources in settings
- [ ] Contact routes + controller + service + validator
- [ ] Duplicate detection (warn on same email)
- [ ] Import endpoint (CSV with BOM/CRLF handling)
- [ ] Export endpoint (CSV with filters)
- [ ] Bulk operations endpoint
- [ ] Optimistic locking
- [ ] Frontend: Contacts list page
- [ ] Frontend: Reusable DataTable component
- [ ] Frontend: Reusable FilterPanel component
- [ ] Frontend: Contact create/edit form
- [ ] Frontend: Contact detail page with tabs
- [ ] Frontend: Import modal
- [ ] Frontend: Export button
- [ ] Frontend: Bulk action bar
- [ ] Frontend: Tag management
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-2 tests pass
- [ ] Milestone 3 COMPLETE

## Milestone 4: Companies Module
- [ ] Companies table migration with indexes
- [ ] Company routes + controller + service + validator
- [ ] Contact-company association
- [ ] Delete behavior (NULL contacts/deals, CASCADE direct children)
- [ ] Company metrics on detail view
- [ ] Duplicate detection on domain
- [ ] Optimistic locking
- [ ] Frontend: Companies list page
- [ ] Frontend: Company create/edit form
- [ ] Frontend: Company detail page with tabs
- [ ] Frontend: Associate/disassociate contacts
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-3 tests pass
- [ ] Milestone 4 COMPLETE

## Milestone 5: Deals & Pipeline Module
- [ ] Deals table migration with indexes
- [ ] Deal stages table migration + seed defaults
- [ ] Deal routes + controller + service + validator
- [ ] Deal statistics endpoint
- [ ] Stage change logic (auto-probability, closed handling)
- [ ] Frontend: Deals table view
- [ ] Frontend: Deals kanban view (drag-and-drop)
- [ ] Frontend: View toggle (table/kanban)
- [ ] Frontend: Deal create/edit form
- [ ] Frontend: Deal detail page (pipeline visual)
- [ ] Frontend: Stage change (kanban drag + detail click)
- [ ] Frontend: Deal statistics display
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-4 tests pass
- [ ] Milestone 5 COMPLETE

## Milestone 6: Activities & Meetings Module
- [ ] Activities table migration
- [ ] Activity types table migration + seed defaults
- [ ] Meetings + meeting_attendees table migration
- [ ] Activity routes + controller + service + validator
- [ ] Meeting routes + controller + service + validator
- [ ] Activity side effects (last_contacted_at updates)
- [ ] Meeting email notifications (graceful degradation)
- [ ] Frontend: Global activity feed page
- [ ] Frontend: Activity timeline component (reusable)
- [ ] Frontend: Log Activity modal
- [ ] Frontend: My Tasks page (overdue/today/upcoming/completed)
- [ ] Frontend: Task quick-complete toggle
- [ ] Frontend: Meetings list page
- [ ] Frontend: Meeting calendar view
- [ ] Frontend: Meeting create/edit form
- [ ] Frontend: Meeting detail page
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-5 tests pass
- [ ] Milestone 6 COMPLETE

## Milestone 7: Notes & Attachments Module
- [ ] Notes table migration
- [ ] Attachments table migration
- [ ] Note routes + controller + service + validator
- [ ] Attachment routes + controller + service + validator
- [ ] R2 storage service (graceful degradation)
- [ ] Multer configuration
- [ ] File type + size validation (client + server)
- [ ] Max 50 attachments per entity
- [ ] Frontend: Notes component (reusable)
- [ ] Frontend: Markdown editor + renderer
- [ ] Frontend: Note edit/delete (creator or admin)
- [ ] Frontend: Pin/unpin notes
- [ ] Frontend: Attachments component (reusable)
- [ ] Frontend: Upload with drag-and-drop + progress
- [ ] Frontend: Attachment list (download, delete)
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-6 tests pass
- [ ] Milestone 7 COMPLETE

## Milestone 8: Settings, API Keys & Email Templates
- [ ] API keys table migration
- [ ] Email templates table migration + seed defaults
- [ ] Email log table migration
- [ ] Settings routes + controller + service
- [ ] Deal stages management (reorder, add, edit, delete with guards)
- [ ] Activity types management (add, edit, deactivate)
- [ ] Email template management (CRUD + preview)
- [ ] API key routes + controller + service
- [ ] API key authentication middleware
- [ ] Audit log viewer with filtering
- [ ] Frontend: Settings page with tabs
- [ ] Frontend: Deal Stages tab (drag-reorder, guards)
- [ ] Frontend: Activity Types tab
- [ ] Frontend: Email Templates tab (edit + preview)
- [ ] Frontend: API Keys tab (generate, list masked, revoke)
- [ ] Frontend: API key generation modal (show once, copy)
- [ ] Frontend: Audit Log tab
- [ ] Tests pass (>80% service coverage)
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Regression: Milestone 1-7 tests pass
- [ ] Milestone 8 COMPLETE

## Milestone 9: Dashboard, Global Search & Polish
- [ ] Dashboard routes + controller + service
- [ ] Frontend: Dashboard page (metrics, charts, widgets)
- [ ] Frontend: Chart components (SVG-based)
- [ ] Update post-login redirect to /dashboard
- [ ] Global search endpoint
- [ ] Frontend: Global search bar (debounced, categorized dropdown)
- [ ] Frontend: Search results page (tabbed)
- [ ] UI polish: loading states (skeleton loaders)
- [ ] UI polish: empty states (with action CTAs)
- [ ] UI polish: error states (with retry)
- [ ] UI polish: responsive design (mobile sidebar, card layouts)
- [ ] UI polish: toast notifications
- [ ] UI polish: confirmation modals
- [ ] UI polish: breadcrumbs
- [ ] Keyboard shortcuts (Ctrl+K, Esc)
- [ ] Performance optimization (indexes, N+1 queries)
- [ ] Performance validation (<200ms lists, <500ms dashboard)
- [ ] E2E test suite (all critical flows)
- [ ] Full regression: ALL milestone test suites pass
- [ ] Verify responsive layout at 320px, 768px, and 1024px viewports for all pages created in this milestone
- [ ] Milestone 9 COMPLETE
```

Update this checklist in your session memory as you complete each item. This is your source of truth.

## Execution Steps Per Milestone

For EACH milestone, follow this exact process:

### Step 1: Plan

- Review the milestone deliverables in this PRD
- Break down into concrete implementation tasks
- Identify dependencies between tasks
- Prepare a detailed task description for the developer agent

### Step 2: Delegate to Developer

Use the Delegation Skill to delegate the milestone work to a Developer agent. Structure your delegation as follows:

```
Delegate to a Developer agent with a detailed task description containing:
- All relevant PRD sections for this milestone
- Database schema details (table definitions, columns, types, constraints, indexes)
- API endpoint specifications (method, path, request body, response shape, status codes)
- Frontend component requirements (component name, props, behavior, layout)
- Validation rules (field-level, form-level, server-side)
- Error handling expectations (what errors to catch, what messages to return)
- Coding best practices to follow (project conventions, naming, file structure)
- Reference to project structure conventions established in Milestone 1
- Testing requirements: Vitest, >80% service coverage, specific test scenarios from this PRD
```

**IMPORTANT:** Include in the task description:

- All relevant PRD sections for this milestone
- Database schema details
- API endpoint specifications
- Frontend component requirements
- Validation rules
- Error handling expectations
- Coding best practices to follow
- Reference to project structure conventions
- The exact test scenarios listed in this PRD for this milestone

### Step 3: Monitor

- Wait for developer to complete
- Check delegation status periodically if no response

### Step 4: Comprehensive Smoke Test

After the developer reports completion, delegate a **comprehensive smoke test** to a tester agent. The tester MUST:

1. **Start the server** and verify it's running on the expected port
2. **API endpoint verification:** For EVERY API endpoint created in this milestone, execute:
   - At least one successful request (valid data, proper auth) — verify correct status code and response shape
   - At least one expected-error request (missing auth → 401, invalid data → 400, not found → 404) — verify error response shape
   - Use `curl` with the session cookie or API key for authentication
3. **Complete user flow testing:** For EVERY entity/feature in this milestone, test the FULL lifecycle:
   - Create → List (verify appears) → View detail → Edit (change a field) → View detail (verify edit persisted) → Delete → List (verify removed)
4. **Frontend verification:** Load every page created in this milestone in a browser/headless check. Verify no console errors, page renders content, navigation works.
5. **Edge case testing:** Run all specific test scenarios listed in the milestone's testing section
6. **Regression testing (Milestone 2+):** Re-run the smoke test curl commands from ALL previous milestones to verify nothing broke
7. **Report results** in a structured format:

| Endpoint/Flow | Test | Status | Error Details |
|---|---|---|---|
| POST /api/v1/contacts | Create with valid data | PASS | — |
| POST /api/v1/contacts | Create with missing firstName | PASS (400) | — |
| GET /api/v1/contacts/export | Export as CSV | FAIL | 500 Internal Server Error |

**Do NOT proceed to the next milestone if ANY test fails.** Fix failures first, then re-test.

Additional test categories per milestone:

1. **API Integration Tests**: Hit every endpoint with valid data, invalid data, missing auth, wrong permissions
2. **E2E Tests**: Execute full user flows through the application
3. **Edge Cases**: Empty states, max length inputs, special characters, concurrent operations
4. **Security Tests**: Auth bypass attempts, injection attempts, CSRF, rate limiting, lockout escalation
5. **Regression Tests**: Run ALL previous milestone test suites

### Step 5: Fix Issues

If tests reveal issues:

- Delegate the specific failures back to a Developer agent with exact error details
- Provide the exact error message, the file and line number, and what the correct behavior should be
- Wait for fixes
- Re-test ONLY the failed scenarios + regression test previously passing scenarios
- Repeat until all tests pass
- Follow the three-strike rule (see Error Recovery)

### Step 6: Confirm & Advance

Once all tests pass for a milestone:

- Update progress checklist: mark milestone COMPLETE
- Tag the commit: `git tag milestone-N-complete`
- Document any deviations from PRD in `DECISIONS.md`
- **DO NOT STOP** — immediately proceed to the next milestone
- Start Step 1 for the next milestone

## Critical Rules

1. **Never skip testing.** Every milestone must be fully tested before moving on.
2. **Never skip a milestone.** They must be executed in order (0 -> 1 -> 2 -> 3 -> ... -> 9).
3. **Keep working until ALL milestones (0-9) are complete.** Do not pause or wait unless explicitly blocked.
4. **Track everything.** Update session memory after every significant action.
5. **Be thorough in delegations.** Include ALL relevant context from this PRD. The developer agent does not have this PRD — you must give them everything they need.
6. **Fix forward, not backward.** If a design decision needs adjustment, make the adjustment in the current milestone, don't redo previous milestones (unless it's a breaking change).
7. **Test with real data.** Create seed data that exercises all features. Don't just test with minimal data.
8. **Check the UI.** After frontend milestones, verify the UI renders correctly, navigation works, forms submit.
9. **Verify external service degradation.** Confirm the app works fully without Brevo and R2 credentials — email functions log instead of sending, file operations return clear error messages.
10. **Run regression tests.** After every milestone, run ALL previous milestone test suites. A new milestone must never break existing functionality.

## Cross-Milestone Integration Points

Pay special attention to these integration points between milestones:

- **Milestone 2 -> 3, 4, 5, 6, 7, 8**: Auth middleware must protect all new routes
- **Milestone 3 -> 4**: Contacts reference companies via `company_id` foreign key
- **Milestone 3, 4 -> 5**: Deals reference both contacts and companies
- **Milestone 5**: Deal stages seeded in deal_stages table — consumed by deals CRUD and kanban
- **Milestone 3, 4, 5 -> 6**: Activities and meetings reference contacts, companies, and deals
- **Milestone 6**: Activity types seeded in activity_types table — consumed by activity CRUD and Log Activity modal
- **Milestone 3, 4, 5, 6 -> 7**: Notes and attachments are polymorphic (attach to any entity)
- **Milestone 2 -> 8**: Audit log (created in M2) is viewed in Settings (M8)
- **Milestone 5 -> 8**: Deal stages (created in M5) are managed in Settings (M8)
- **Milestone 6 -> 8**: Activity types (created in M6) are managed in Settings (M8)
- **Milestone ALL -> 9**: Dashboard aggregates data from all entity types; global search spans contacts, companies, and deals
- **Milestone 2 -> 9**: Post-login redirect changes from `/contacts` to `/dashboard`

## When You're Done

After all milestones (0-9) are complete:

1. Run a final comprehensive test across all modules (`npx vitest run --coverage`)
2. Verify all features work together (cross-module interactions)
3. Check for any console errors or warnings
4. Verify all environment variables are documented in `.env.example`
5. Verify the app works without Brevo and R2 credentials (graceful degradation)
6. Confirm the following cross-module flows work end-to-end:
   - Create a company -> add contacts to it -> create a deal linked to company and contact -> log activities -> add notes -> attach files -> view everything on dashboard -> search for company name and find all related records
   - Admin creates user -> user logs in -> user creates contact -> user logs activity -> user views dashboard -> admin deactivates user -> user cannot login
   - Generate API key with `contacts:read` -> use key to GET contacts (200) -> try POST contacts (403) -> revoke key -> GET contacts returns 401
   - Forgot password flow -> reset link email logged (or sent) -> reset password -> login with new password
7. Report to user: milestone summary, what was built, any deviations documented in `DECISIONS.md`, how to start the CRM (`npm run dev` in both client and server directories)
