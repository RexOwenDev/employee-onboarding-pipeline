# Setup Guide — Employee Onboarding Pipeline

> **Implementation requires:** Postgres admin access, Google Workspace Super Admin, Slack workspace
> admin, n8n Cloud account, and organizational knowledge about the client's department structure,
> provisioning tools, and onboarding process. This is not a self-service installation.

---

## Prerequisites

| Requirement | Why Needed |
|---|---|
| n8n Cloud account (or self-hosted n8n ≥ 1.40) | Hosts all 10 workflows |
| Google Workspace Super Admin | GSuite Service Account domain-wide delegation |
| Slack workspace Admin | Bot token with users:write.invites scope |
| Supabase project (or any Postgres instance) | 6 tables for config, audit, scheduling |
| BambooHR account with Webhook access | Trigger source for new hire events |
| Anthropic API key | Claude Haiku — role classification + welcome email |
| ClickUp workspace Admin | Create tasks across team spaces |
| Notion workspace Admin | Create member onboarding pages |
| Google account with Sheets + Gmail | Audit log + welcome email sending |

---

## Phase 1: Postgres Schema

Run the following SQL in your Supabase (or Postgres) instance. Execute in order.

```sql
-- 1. Canonical hire record
CREATE TABLE new_hires (
  id               SERIAL PRIMARY KEY,
  hire_id          TEXT NOT NULL UNIQUE,
  full_name        TEXT NOT NULL,
  personal_email   TEXT NOT NULL,
  job_title        TEXT NOT NULL,
  department       TEXT,
  role_group       TEXT,
  employment_type  TEXT DEFAULT 'FULL_TIME',
  start_date       DATE NOT NULL,
  manager_email    TEXT,
  location         TEXT,
  client_slug      TEXT NOT NULL,
  source_system    TEXT DEFAULT 'bamboohr',
  status           TEXT DEFAULT 'PENDING',
  it_approved_by   TEXT,
  it_approved_at   TIMESTAMPTZ,
  created_at       TIMESTAMPTZ DEFAULT NOW(),
  updated_at       TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Multi-tenant company config
CREATE TABLE company_config (
  id                    SERIAL PRIMARY KEY,
  slug                  TEXT NOT NULL UNIQUE,
  name                  TEXT NOT NULL,
  email_domain          TEXT,
  slack_hr_channel      TEXT,
  slack_it_channel      TEXT,
  slack_ops_channel     TEXT,
  slack_finance_channel TEXT,
  it_manager_slack_id   TEXT,
  hr_rep_slack_id       TEXT,
  active                BOOLEAN DEFAULT true,
  created_at            TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Role group → provisioning plan
--    NOTE: keywords TEXT[] is used by Workflow 02's Postgres keyword fallback.
--    See Phase 2b for per-role keyword guidance.
CREATE TABLE role_groups (
  id                    SERIAL PRIMARY KEY,
  client_id             INTEGER REFERENCES company_config(id),
  role_group            TEXT NOT NULL,
  tools                 TEXT[],
  gsuite_ou             TEXT,
  slack_channels        TEXT[],
  notion_page_id        TEXT,
  checklist_template_id TEXT,
  keywords              TEXT[]
);

-- 4. Onboarding task templates
CREATE TABLE onboarding_templates (
  id               SERIAL PRIMARY KEY,
  client_id        INTEGER REFERENCES company_config(id),
  role_group       TEXT NOT NULL,  -- or 'ALL' for universal tasks
  task_name        TEXT NOT NULL,
  task_description TEXT,
  assignee_type    TEXT,
  due_offset_days  INTEGER,
  priority         TEXT DEFAULT 'normal'
);

-- 5. Event audit log (append-only)
CREATE TABLE onboarding_events (
  id           SERIAL PRIMARY KEY,
  hire_id      TEXT REFERENCES new_hires(hire_id),
  event_type   TEXT NOT NULL,
  status       TEXT NOT NULL,
  details      JSONB,
  performed_by TEXT,
  occurred_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 6. Scheduled check-in queue
--    NOTE: The composite UNIQUE constraint on (hire_id, event_type) is required.
--    Workflow 07 uses ON CONFLICT (hire_id, event_type) DO NOTHING to ensure
--    each check-in type (DAY_7, DAY_14, DAY_30) is only scheduled once per hire,
--    even if the pipeline is accidentally re-triggered.
CREATE TABLE scheduled_events (
  id            SERIAL PRIMARY KEY,
  hire_id       TEXT REFERENCES new_hires(hire_id),
  event_type    TEXT NOT NULL,
  scheduled_at  TIMESTAMPTZ NOT NULL,
  status        TEXT DEFAULT 'PENDING',
  completed_at  TIMESTAMPTZ,
  UNIQUE(hire_id, event_type)
);
```

---

## Phase 2: Seed Data (Requires Organizational Knowledge)

### 2a. Insert company_config row

This requires knowing the client's Slack channel IDs and Slack user IDs for IT manager and HR rep.

```sql
INSERT INTO company_config (
  slug, name, email_domain,
  slack_hr_channel, slack_it_channel, slack_ops_channel, slack_finance_channel,
  it_manager_slack_id, hr_rep_slack_id
)
VALUES (
  '[CLIENT_SLUG]',               -- e.g., 'acme'
  '[COMPANY_NAME]',              -- e.g., 'Acme Corp'
  '[EMAIL_DOMAIN]',              -- e.g., 'acme.com'
  '[SLACK_HR_CHANNEL_ID]',       -- e.g., 'C01ABC123'
  '[SLACK_IT_CHANNEL_ID]',
  '[SLACK_OPS_CHANNEL_ID]',
  '[SLACK_FINANCE_CHANNEL_ID]',
  '[IT_MANAGER_SLACK_USER_ID]',  -- Slack user ID (Uxxxxxxxx format)
  '[HR_REP_SLACK_USER_ID]'
);
```

### 2b. Insert role_groups rows

**This requires interviewing department heads and IT.** There is no generic default.

The `keywords` column drives Workflow 02's Postgres keyword fallback — when Claude's role
classification cannot confidently determine a role group, the workflow falls back to a Postgres
`keywords` array match against the hire's job title. Include realistic synonyms that your client's
HR system actually uses.

Example keywords per role:

| Role Group | Example keywords |
|---|---|
| ENGINEERING | `engineer`, `developer`, `software`, `backend`, `frontend`, `fullstack`, `devops`, `sre`, `data` |
| MARKETING | `marketing`, `brand`, `content`, `seo`, `growth`, `copywriter`, `social`, `campaign` |
| SALES | `sales`, `account executive`, `ae`, `bdr`, `sdr`, `revenue`, `business development` |
| FINANCE | `finance`, `accounting`, `accountant`, `controller`, `fp&a`, `payroll`, `bookkeeper` |
| HR | `hr`, `human resources`, `people ops`, `recruiter`, `talent`, `hrbp`, `onboarding` |
| OPERATIONS | `operations`, `ops`, `project manager`, `pm`, `coordinator`, `logistics`, `supply chain` |
| EXECUTIVE | `vp`, `director`, `chief`, `cto`, `cfo`, `ceo`, `head of`, `president` |

For each department the client has:

```sql
INSERT INTO role_groups (
  client_id, role_group, tools, gsuite_ou, slack_channels,
  notion_page_id, checklist_template_id, keywords
)
VALUES (
  (SELECT id FROM company_config WHERE slug = '[CLIENT_SLUG]'),
  'ENGINEERING',
  ARRAY['gsuite','slack','notion','github','linear'],
  '/Engineering',
  ARRAY['[SLACK_ENG_CHANNEL_ID]', '[SLACK_GENERAL_CHANNEL_ID]'],
  '[NOTION_ENG_TEAM_PAGE_ID]',
  '[CLICKUP_CHECKLIST_TEMPLATE_ID]',
  ARRAY['engineer','developer','software','backend','frontend','fullstack','devops','sre','data']
);
-- Repeat for MARKETING, SALES, FINANCE, HR, OPERATIONS, EXECUTIVE, DEFAULT
-- DEFAULT role_group catches anything not matched; set keywords = ARRAY['default'] or similar
```

### 2c. Insert onboarding_templates rows

**This requires knowing the client's actual onboarding process.** Start with universals, then add
role-specific tasks.

```sql
-- Universal tasks (role_group = 'ALL')
INSERT INTO onboarding_templates
  (client_id, role_group, task_name, assignee_type, due_offset_days, priority)
VALUES
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', 'Complete HR paperwork (I-9, W-4, direct deposit)', 'hire', 0, 'urgent'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', 'IT equipment setup and account login verification', 'it_rep', 1, 'urgent'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', 'Review company handbook and sign acknowledgement', 'hire', 1, 'high'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', 'Complete benefits enrollment (deadline: Day 3)', 'hire', 3, 'urgent'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', 'Manager 1:1 check-in (Day 7)', 'manager', 7, 'high'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ALL', '30-day performance review', 'manager', 30, 'high');

-- Role-specific tasks — examples for ENGINEERING
INSERT INTO onboarding_templates
  (client_id, role_group, task_name, assignee_type, due_offset_days, priority)
VALUES
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ENGINEERING', 'Complete dev environment setup', 'hire', 1, 'urgent'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ENGINEERING', 'Attend architecture walkthrough with tech lead', 'hire', 3, 'high'),
  ((SELECT id FROM company_config WHERE slug='[CLIENT_SLUG]'),
   'ENGINEERING', 'Complete first pull request (good-first-issue)', 'hire', 7, 'normal');
-- Add role-specific tasks for each role_group your client uses
```

---

## Phase 3: n8n Credential Setup

Create each credential in n8n → Credentials → New:

1. **Postgres** — Host: `[SUPABASE_HOST]`, Port: 5432, Database: postgres, User: postgres,
   Password: `[SUPABASE_PASSWORD]`, SSL: required
2. **Slack** — Bot Token: `xoxb-...`
   Required OAuth scopes: `channels:read`, `chat:write`, `users:read.email`, `users:write.invites`
3. **Gmail OAuth2** — Client ID + Secret from GCP Console → OAuth2 Credentials → Gmail API scope
4. **Google Sheets OAuth2** — same GCP project; add Google Sheets API scope
5. **HTTP Header Auth (Anthropic)** — Header Name: `x-api-key`, Value: `sk-ant-...`

After creating each credential, note the credential ID. Reference them by name in each workflow's
node configuration.

---

## Phase 4: GSuite Service Account

1. GCP Console → IAM & Admin → Service Accounts → Create
2. Name: `n8n-onboarding-sa`
3. Create key → JSON → download → minify to one line (no newlines in the JSON string)
4. Google Workspace Admin Console → Security → API Controls → Domain-wide Delegation → Add new:
   - Client ID: `[SA_CLIENT_ID from the downloaded JSON]`
   - OAuth Scopes: `https://www.googleapis.com/auth/admin.directory.user`
5. Copy minified JSON → n8n Settings → Variables → `GSUITE_SERVICE_ACCOUNT_JSON`
6. Set `GSUITE_ADMIN_EMAIL` n8n variable to a Super Admin email (used as the impersonation subject)

> **Security note:** `GSUITE_SERVICE_ACCOUNT_JSON` is a high-privilege credential with
> domain-wide delegation. Restrict access to n8n admins only. Do not expose in logs or workflow
> debug output.

### §4.1 GitHub Adapter

Implement in `04-account-provisioning` → github case branch:

```javascript
const resp = await fetch(
  `https://api.github.com/orgs/${ORG}/teams/${TEAM_SLUG}/memberships/${username}`,
  {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer [GITHUB_PAT]`,
      'Accept': 'application/vnd.github+json'
    },
    body: JSON.stringify({ role: 'member' })
  }
);
if (!resp.ok) throw new Error(`GitHub invite failed: ${resp.status} ${await resp.text()}`);
```

Required: GitHub Personal Access Token with `admin:org` scope. Store in n8n env as
`GITHUB_PAT`.

### §4.2 Linear Adapter

```javascript
const resp = await fetch('https://api.linear.app/graphql', {
  method: 'POST',
  headers: {
    'Authorization': '[LINEAR_API_KEY]',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    query: `mutation { memberInvite(email: "EMAIL", teamId: "TEAM_ID") { success } }`
  })
});
const result = await resp.json();
if (!result.data?.memberInvite?.success) throw new Error('Linear invite failed');
```

Required: Linear API key with admin access. Store as `LINEAR_API_KEY`.

### §4.3 HubSpot Adapter

Follow the stub pattern in `04-account-provisioning`. Use HubSpot's Users API
(`POST /settings/v3/users`) to invite by email. Required scope: `crm.objects.users.write`.
Store token as `HUBSPOT_ACCESS_TOKEN`.

### §4.4 Canva Adapter

Follow the stub pattern in `04-account-provisioning`. Use Canva's Team Users API to invite.
Canva requires an Enterprise plan for API-based user provisioning. Store token as
`CANVA_API_TOKEN`.

---

## Phase 5: Google Sheets Audit Log

Create a Google Sheet. Add a tab named exactly: **Onboarding Audit Log**

Add these 22 column headers in row 1 (exact order, exact spelling):

| # | Column |
|---|---|
| 1 | hire_id |
| 2 | full_name |
| 3 | personal_email |
| 4 | job_title |
| 5 | department |
| 6 | role_group |
| 7 | employment_type |
| 8 | start_date |
| 9 | manager_email |
| 10 | source_system |
| 11 | it_approved_by |
| 12 | it_approved_at |
| 13 | gsuite_status |
| 14 | slack_status |
| 15 | notion_status |
| 16 | additional_tools_status |
| 17 | welcome_email_sent |
| 18 | manager_notified |
| 19 | clickup_checklist_created |
| 20 | check_ins_scheduled |
| 21 | pipeline_started_at |
| 22 | pipeline_completed_at |

After creating the sheet, copy the Sheet ID from the URL
(`https://docs.google.com/spreadsheets/d/[SHEET_ID]/edit`) and set it in the
`08-audit-log` workflow's Google Sheets node.

> **Implementation note:** The append node uses `cellFormat: "RAW"` to prevent formula
> injection. Do not change this to `USER_ENTERED` — hire names or email values containing
> `=` would be interpreted as formulas.

---

## Phase 6: n8n Workflow Import

Import workflow files in this exact order. Workflow `00` must be imported first — all other
workflows reference its error-handler workflow ID via `[00_WORKFLOW_ID]` tokens.

| Import order | File | Notes |
|---|---|---|
| 1 | `00-error-handler.json` | Import first; copy workflow ID from URL after save |
| 2 | `08-audit-log.json` | Sub-workflow; used by all provisioning steps |
| 3 | `07-checkin-scheduler.json` | Schedules DAY_7 / DAY_14 / DAY_30 events |
| 4 | `07b-checkin-runner.json` | Schedule Trigger — polls `scheduled_events` |
| 5 | `06-clickup-checklist.json` | Sub-workflow; creates ClickUp task lists |
| 6 | `05-stakeholder-notifications.json` | Sub-workflow; sends emails + Slack alerts |
| 7 | `04-account-provisioning.json` | Sub-workflow; provisions all tool accounts |
| 8 | `03-it-approval-gate.json` | Slack sendAndWait approval gate |
| 9 | `02-role-classifier.json` | Claude Haiku classification + Postgres fallback |
| 10 | `01-hris-intake-bamboohr.json` | Webhook entry point; activate last |

After importing each workflow:

1. Copy the workflow ID from the n8n URL bar (`/workflow/[ID]`)
2. Replace the corresponding `[NN_WORKFLOW_ID]` placeholder token in all referencing workflow JSON
   files before importing them
3. Re-import (or update via n8n API) any workflow where a referenced ID changed

**Activation order** (activate after all imports and ID replacements are complete):

```
00 → 07b → 08 → 07 → 06 → 05 → 04 → 03 → 02 → 01
```

Activate `01-hris-intake-bamboohr` last — this is the entry point and enabling it before the
sub-workflows are active will cause failures on first trigger.

---

## Phase 7: IT Approval Gate Constraint

The Slack "Send and Wait for Approval" node in `03-it-approval-gate` uses n8n's built-in
`sendAndWait` operation, which sends interactive Slack message buttons and suspends workflow
execution until the IT manager responds.

**Known constraint: button labels are fixed.**

n8n's `sendAndWait` node does not support custom button label text. The approval buttons are
always labelled:

- **Approve** (primary style)
- **Disapprove** (secondary style)

You cannot rename these to "Yes / No", "Provision / Hold", or any other text. This is a
platform limitation — not a configuration option.

The response value is binary: `approved: true` or `approved: false`. The workflow branches on
this value at the "Check — Approved or Disapproved?" IF node.

**If reason capture is required for disapproved hires:**

The current implementation records `'Disapproved in Slack'` as the reason. If the client requires
a freeform hold reason, two options exist:

1. After the Disapprove branch fires, send a follow-up `chat.postMessage` to the same IT manager
   as a thread reply, requesting the reason. Parse their reply via a separate webhook or polling
   mechanism.
2. Replace `sendAndWait` with a custom community node or an external approval page (e.g., a small
   n8n Form trigger) that supports richer input collection.

Option 1 is lower effort but requires additional workflow logic. Option 2 requires a community
node or frontend build.

---

## Phase 8: BambooHR Webhook Configuration

BambooHR → Settings → Webhooks → New Webhook:

| Field | Value |
|---|---|
| URL | `https://[YOUR_N8N_URL]/webhook/bamboohr-intake` |
| Events | Employee Added, Employee Data Changed |
| Format | JSON |
| Secret | Same value as `BAMBOOHR_HMAC_SECRET` in n8n environment variables |

Set the `BAMBOOHR_HMAC_SECRET` n8n environment variable before activating `01-hris-intake-bamboohr`.
The intake workflow verifies this signature on every request.

---

## Phase 9: End-to-End Test

Use this test payload against the webhook URL with a correct HMAC-SHA256 signature.

```bash
# Step 1 — Generate test HMAC
echo -n '{"employeeId":"TEST001","firstName":"Jane","lastName":"Test","personalEmail":"jane@personal.com","jobTitle":"Software Engineer","department":"Engineering","hireDate":"2026-05-01","supervisorEmail":"manager@company.com"}' \
  | openssl dgst -sha256 -hmac "[BAMBOOHR_HMAC_SECRET]"

# Step 2 — Send test webhook
curl -X POST https://[N8N_URL]/webhook/bamboohr-intake \
  -H "Content-Type: application/json" \
  -H "X-BambooHR-Signature: [HMAC_FROM_STEP_1]" \
  -d '{"employeeId":"TEST001","firstName":"Jane","lastName":"Test","personalEmail":"jane@personal.com","jobTitle":"Software Engineer","department":"Engineering","hireDate":"2026-05-01","supervisorEmail":"manager@company.com"}'
```

**Expected result:** HTTP 200 `{"status":"accepted"}` + IT approval Slack DM arrives within
30 seconds.

**Verify end-to-end:**
- [ ] Postgres `new_hires` row inserted with `status = 'PENDING'`
- [ ] IT manager received Slack DM with Approve / Disapprove buttons
- [ ] After clicking Approve: `new_hires.status` → `'APPROVED'`; provisioning sub-workflows fire
- [ ] `onboarding_events` table has records for each step (INTAKE, APPROVAL_GRANTED, GSUITE, SLACK, etc.)
- [ ] `scheduled_events` has 3 rows for TEST001: DAY_7, DAY_14, DAY_30
- [ ] Audit log row appended to Google Sheet
- [ ] Welcome email delivered to `jane@personal.com`

**HMAC rawBody caveat:**

The HMAC verification code in Workflow 01's "Verify HMAC + Normalize Payload" Code node
attempts to extract the raw request body using this priority order:

1. `item.binary.data.data` — base64-decoded binary body (most accurate; available when n8n
   sends the webhook with raw body capture enabled via `rawBody: true`)
2. `item.json.rawBody` — string rawBody from the parsed JSON item
3. Fallback: re-serializes `item.json.body` via `JSON.stringify` — **not byte-perfect** vs. the
   original bytes sent by BambooHR

The fallback is acceptable for BambooHR webhooks because BambooHR sends JSON-only payloads.
JSON re-serialization is deterministic for the key ordering BambooHR uses in practice.
However: if BambooHR changes its key ordering or adds whitespace, the fallback HMAC will
mismatch. The preferred production path is binary body extraction (path 1) — verify that
n8n's Webhook node has `rawBody` enabled.

---

## Phase 10: Production Operational Notes

### Monitoring

- **Monitor `scheduled_events` queue size daily.** A growing `status = 'PENDING'` backlog that
  doesn't shrink indicates that `07b-checkin-runner` is not firing or is erroring. Check n8n
  execution history for Workflow 07b.

- **Review `onboarding_events` table weekly for FAILED events.** Query:

  ```sql
  SELECT hire_id, event_type, details, occurred_at
  FROM onboarding_events
  WHERE status = 'FAILED'
  ORDER BY occurred_at DESC
  LIMIT 50;
  ```

  Failed events should trigger investigation — a failed GSuite provisioning means the hire has no
  email account.

### Credential Rotation Schedule

Rotate the following annually (or immediately upon suspected compromise):

| Credential | n8n Location |
|---|---|
| `BAMBOOHR_HMAC_SECRET` | n8n env variable + BambooHR webhook config (must match) |
| `ANTHROPIC_API_KEY` | n8n HTTP Header Auth credential |
| `SLACK_BOT_TOKEN` | n8n Slack credential |
| `NOTION_INTEGRATION_TOKEN` | n8n HTTP Header Auth credential |
| `CLICKUP_API_TOKEN` | n8n HTTP Header Auth credential |

When rotating `BAMBOOHR_HMAC_SECRET`, update both the n8n env variable and the BambooHR
webhook secret in the same maintenance window — mismatched values will cause all incoming
webhooks to fail HMAC verification.

### Cost Controls

- Set an Anthropic usage alert at **$10/month**.
  Claude Haiku costs approximately $0.003 per hire (role classification + welcome email
  generation). At that rate, you need 3,000+ hires per month before hitting $10. The alert
  exists to catch runaway executions or misconfigured retry loops — not normal usage.

- Set a Postgres connection alert if using Supabase free tier (connection limit: 60).
  At high volume, consider enabling PgBouncer in Supabase settings.

### Access Control

- The `GSUITE_SERVICE_ACCOUNT_JSON` n8n variable contains a private key with Google
  Workspace domain-wide delegation. This is the highest-privilege credential in the stack —
  it can read and write all user accounts in the Google Workspace org.
  **Restrict n8n admin access accordingly.** Do not grant n8n editor access to anyone who
  should not have GSuite Super Admin equivalent capability.
