# Object Catalog

Default object types and field schemas for the okf-brain extraction pipeline. Use these when the user does not specify a custom scope.

## Default object types

| Object | Description |
|--------|-------------|
| `Task` | Action items, todos, work assignments with status and due dates |
| `Thread` | Conversation topics, discussions, issues spanning multiple messages |
| `Attendance` | Employee check-in/check-out, work location, hours worked |
| `Appointment` | Meetings, calls, scheduled events |
| `Lead` | Sales prospects, referrals, business opportunities |
| `Expense` | Purchases, reimbursements, financial transactions |

The agent may scope to a subset (e.g., only `Task` and `Appointment`) when the user requests it.

---

## Task

Action items and work assignments extracted from context.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Short task name (3–8 words) |
| `description` | string | no | Brief context (5–15 words) |
| `status` | enum | yes | One of: `New`, `In Progress`, `Done`, `Closed` |
| `reason` | string | no | Why the task exists or its current state |
| `due_date` | string | no | Due date as stated in context |
| `last_message` | string | no | Most recent message referencing the task |
| `relevant_messages` | string | no | Other messages that mention the task |

**Extraction notes:**

- Use past tense for completed tasks ("was completed", "has been resolved")
- Use imperative language only for pending or newly assigned tasks
- Do not extract items you are not confident are actual tasks

---

## Thread

Distinct conversation topics, discussions, or issues.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Thread title |
| `description` | string | no | What the thread is about |
| `status` | string | yes | One of: `Open`, `Closed`, `Resolved`, `Ongoing` |
| `participants` | string[] | no | People involved in the discussion |
| `start_time` | string | no | When the discussion started |
| `end_time` | string | no | When the discussion ended (or "Not specified") |
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
| `log_in_time` | string | no | Login time |
| `log_out_time` | string | no | Logout time |
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
| `date_time` | string | no | Scheduled date and time |
| `location` | string | no | Physical address or virtual platform; use "Not specified" if absent |
| `participants` | string[] | no | Attendees |
| `description` | string | no | Purpose or agenda |
| `status` | string | no | One of: `Scheduled`, `Confirmed`, `Cancelled`, `Completed` |

**Extraction notes:**

- Include both virtual and in-person meetings
- For virtual meetings, note the platform (Zoom, Meet, etc.) in `location`
- Capture status updates (confirmed, cancelled, rescheduled)

---

## Lead

Sales prospects and business opportunities.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Contact name |
| `company` | string | no | Company name |
| `email` | string | no | Email address; "Not provided" if absent |
| `phone` | string | no | Phone number; "Not provided" if absent |
| `source` | string | no | How found (referral, website, cold outreach, etc.) |
| `status` | string | no | One of: `New`, `Contacted`, `Qualified`, `Unqualified`, `Converted` |
| `notes` | string | no | Additional context |
| `interest_level` | string | no | One of: `High`, `Medium`, `Low` |
| `product_interest` | string | no | Product or service of interest |
| `budget` | string | no | Budget information |
| `timeline` | string | no | Decision timeline |

**Extraction notes:**

- Include direct leads and referrals
- Extract contact info when present; mark missing fields as "Not provided"

---

## Expense

Purchases, reimbursements, and financial transactions.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | yes | What the expense is for |
| `amount` | number | yes | Monetary amount |
| `currency` | string | no | Currency code (default to context locale if unspecified) |
| `category` | string | no | e.g., `Travel`, `Meals`, `Office Supplies`, `Equipment`, `Software`, `Marketing` |
| `date` | string | no | Date of expense |
| `vendor` | string | no | Vendor or merchant |
| `paid_by` | string | no | Person who paid |
| `status` | string | no | One of: `Pending`, `Approved`, `Rejected`, `Reimbursed` |
| `receipt` | string | no | Receipt number or reference; "Not provided" if absent |
| `notes` | string | no | Additional details |

**Extraction notes:**

- Include business and personal expenses needing reimbursement
- Extract amounts from receipts, invoices, and images when present in context
- Do not invent amounts not supported by the source

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

When custom types are defined, include them in the `object_types` list passed to the signals and bundles stages.
