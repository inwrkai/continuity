# Object Catalog

Default object types and field schemas for the ops-brain extraction pipeline. Use these when the user does not specify a custom scope.

## Default object types

| Object | Description |
|--------|-------------|
| `Task` | Action items, todos, work assignments with status and due dates |
| `Thread` | Conversation topics, discussions, issues spanning multiple messages |
| `Attendance` | Employee check-in/check-out, work location, hours worked |
| `Appointment` | Meetings, calls, scheduled events |
| `Lead` | Sales prospects, referrals, business opportunities |
| `Expense` | Purchases, reimbursements, financial transactions |
| `Decision` | Choices agreed in conversation — what was decided and by whom |
| `Blocker` | Impediments preventing progress on work |
| `Question` | Open questions raised in discussion that need answers |

The agent may scope to a subset (e.g., only `Task` and `Appointment`) when the user requests it.

---

## Shared field rules

### Date and time fields

Any field that holds a date or datetime (`due_date`, `date_time`, `start_time`, `end_time`, `date`, `reported_at`, etc.):

1. Resolve relative phrases ("Thursday", "tomorrow at 10") against the transcript/run `anchor_date` into ISO 8601
2. Preserve the original phrasing in citations
3. If no anchor date is available, keep the raw phrase and set `as_of_date` to the run date

See [stages.md — Date anchoring](stages.md#date-anchoring).

### Confidence

Records use categorical confidence: `high`, `medium`, or `low`. Do not write numeric confidence scores.

When reading legacy bundles that still have numeric `confidence`, map:

| Numeric | Categorical |
|---------|-------------|
| ≥ 0.8 | `high` |
| ≥ 0.5 | `medium` |
| < 0.5 | `low` |

---

## Task

Action items and work assignments extracted from context.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Short task name (3–8 words) |
| `description` | string | no | Brief context (5–15 words) |
| `status` | enum | yes | One of: `New`, `In Progress`, `Done`, `Closed` |
| `reason` | string | no | Why the task exists or its current state |
| `due_date` | string | no | Due date as ISO 8601 when resolvable |
| `as_of_date` | string | no | Run/transcript date when `due_date` could not be resolved |
| `last_message` | string | no | Most recent message referencing the task |
| `relevant_messages` | string | no | Other messages that mention the task |

**Extraction notes:**

- Use past tense for completed tasks ("was completed", "has been resolved")
- Use imperative language only for pending or newly assigned tasks
- Do not extract items you are not confident are actual tasks
- Resolve relative due dates (e.g., "Thursday" → `2026-07-16`)

---

## Thread

Distinct conversation topics, discussions, or issues.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Thread title |
| `description` | string | no | What the thread is about |
| `status` | string | yes | One of: `Open`, `Closed`, `Resolved`, `Ongoing` |
| `participants` | string[] | no | People involved in the discussion |
| `start_time` | string | no | When the discussion started (ISO 8601 when resolvable) |
| `end_time` | string | no | When the discussion ended (ISO 8601 when resolvable) |
| `priority` | string | no | One of: `High`, `Medium`, `Low` |
| `category` | string | no | e.g., `Issue`, `Discussion`, `Question`, `Update`, `Feature Request` |
| `resolution` | string | no | How it was resolved (if closed) |

**Extraction notes:**

- Identify when discussions start and end
- Track distinct topics that span multiple messages
- Capture status updates on ongoing issues

---

## Attendance

Employee attendance records from check-in/check-out messages.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `employee_name` | string | yes | Employee name |
| `log_in_time` | string | no | Login time (ISO 8601 when resolvable) |
| `log_out_time` | string | no | Logout time (ISO 8601 when resolvable) |
| `working_from_home` | boolean | no | `true` if WFH/remote |
| `total_time_worked` | string | no | Duration (e.g., "8h 30m") |

**Extraction notes:**

- Use the latest message for the same employee when multiple updates exist
- Messages may reference dates different from the message timestamp
- Include all employees mentioned, including proxy updates ("logged out for X")

---

## Appointment

Meetings, calls, and scheduled events.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Appointment or meeting title |
| `date_time` | string | no | Scheduled date and time as ISO 8601 when resolvable |
| `as_of_date` | string | no | Run/transcript date when `date_time` could not be resolved |
| `location` | string | no | Physical address or virtual platform; omit if absent |
| `participants` | string[] | no | Attendees |
| `description` | string | no | Purpose or agenda |
| `status` | string | no | One of: `Scheduled`, `Confirmed`, `Cancelled`, `Completed` |

**Extraction notes:**

- Include both virtual and in-person meetings
- For virtual meetings, note the platform (Zoom, Meet, etc.) in `location`
- Capture status updates (confirmed, cancelled, rescheduled)
- Resolve "tomorrow at 10" against `anchor_date`

---

## Lead

Sales prospects and business opportunities.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Contact name |
| `company` | string | no | Company name |
| `email` | string | no | Email address; omit if absent |
| `phone` | string | no | Phone number; omit if absent |
| `source` | string | no | How found (referral, website, cold outreach, etc.) |
| `status` | string | no | One of: `New`, `Contacted`, `Qualified`, `Unqualified`, `Converted` |
| `notes` | string | no | Additional context |
| `interest_level` | string | no | One of: `High`, `Medium`, `Low` |
| `product_interest` | string | no | Product or service of interest |
| `budget` | string | no | Budget information |
| `timeline` | string | no | Decision timeline (ISO 8601 date when resolvable) |

**Extraction notes:**

- Include direct leads and referrals
- Extract contact info when present; omit missing optional fields (do not invent "Not provided")

---

## Expense

Purchases, reimbursements, and financial transactions.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | yes | What the expense is for |
| `amount` | number | yes | Monetary amount |
| `currency` | string | no | Currency code (default to context locale if unspecified) |
| `category` | string | no | e.g., `Travel`, `Meals`, `Office Supplies`, `Equipment`, `Software`, `Marketing` |
| `date` | string | no | Date of expense as ISO 8601 when resolvable |
| `as_of_date` | string | no | Run/transcript date when `date` could not be resolved |
| `vendor` | string | no | Vendor or merchant |
| `paid_by` | string | no | Person who paid |
| `status` | string | no | One of: `Pending`, `Approved`, `Rejected`, `Reimbursed` |
| `receipt` | string | no | Receipt number or reference; omit if absent |
| `notes` | string | no | Additional details |

**Extraction notes:**

- Include business and personal expenses needing reimbursement
- Extract amounts from receipts, invoices, and images when present in context
- Do not invent amounts not supported by the source

---

## Decision

Choices agreed during conversation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `decision` | string | yes | What was decided (concise statement) |
| `decided_by` | string | no | Person or group who decided |
| `rationale` | string | no | Why the decision was made |
| `status` | string | yes | One of: `Proposed`, `Agreed`, `Rejected`, `Revisited` |
| `date` | string | no | When decided (ISO 8601 when resolvable) |
| `related_to` | string | no | Related task, project, or topic name |

**Extraction notes:**

- Extract only clear decisions, not exploratory brainstorming
- Prefer `Agreed` when the group explicitly confirms
- Capture "we decided to…" / "going with option B" style statements

---

## Blocker

Impediments preventing progress.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `summary` | string | yes | What is blocked |
| `owner` | string | no | Person responsible or affected |
| `blocking_reason` | string | no | Why progress is blocked |
| `severity` | string | no | One of: `Critical`, `High`, `Medium`, `Low` |
| `status` | string | yes | One of: `Open`, `Mitigating`, `Resolved` |
| `unblocks` | string | no | What work resumes when resolved |

**Extraction notes:**

- Extract explicit blockers ("blocked on…", "waiting for…", "can't proceed until…")
- Do not invent severity; omit if not stated
- Update status when a later message resolves the blocker

---

## Question

Open questions that need answers.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `question` | string | yes | The open question |
| `asked_by` | string | no | Who asked |
| `directed_to` | string | no | Who should answer |
| `answer` | string | no | Answer if provided later in context |
| `status` | string | yes | One of: `Open`, `Answered`, `Deferred` |

**Extraction notes:**

- Prefer actionable or decision-relevant questions over rhetorical ones
- If the same conversation answers the question, set `answer` and `status: Answered`
- Do not extract every interrogative sentence — only meaningful open questions

---

## Custom object types

Users may define additional object types for a specific bundle. Store the configuration in the bundle root `index.md` frontmatter:

```yaml
---
okf_version: "0.1"
title: My Project Knowledge
object_types:
  - Task
  - Appointment
  - Incident
object_schemas:
  Incident:
    title: string
    severity: string  # Critical, High, Medium, Low
    status: string    # Open, Investigating, Resolved
    assignee: string
    reported_at: string
---
```

### Custom type guidelines

1. **Name** — PascalCase, singular (e.g., `Incident`, `Decision`, `Milestone`)
2. **Fields** — define only fields you want extracted; mark required fields in the schema
3. **Reuse** — on subsequent runs, read `object_types` and `object_schemas` from the existing bundle's `index.md`
4. **Subset** — user can request a subset of default types without defining custom schemas
5. **Validation** — the agent uses schemas as extraction guidance, not strict validation

### When to add custom types

- Domain-specific entities not covered by defaults (e.g., `Bug`, `Feature`, `Contract`)
- Team-specific terminology that maps poorly to default types
- Scoped extractions where a single object type is sufficient

When custom types are defined, include them in the `object_types` list passed to the extract stage.
