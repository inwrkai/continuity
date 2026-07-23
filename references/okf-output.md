# OKF Output Format

Rules for writing Continuity pipeline output as a portable knowledge bundle on disk under `inwrk/`.

The layout is self-contained markdown with YAML frontmatter. It is designed to be compatible with [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) where useful; agents should follow the rules in this file even without consulting the external spec.

## Bundle layout

Default output directory: `inwrk/` in the current workspace (user may override). If `inwrk/` is missing but a legacy `okf/` directory exists, rename `okf/` → `inwrk/` before reading or writing (e.g. `mv okf inwrk`).

```
inwrk/
├── index.md                # Root index with okf_version and object config
├── schema.md               # Current workspace schema (authoritative when present)
├── schema/
│   └── vN.md               # Immutable snapshots per confirmed schema version
├── log.md                  # Chronological update history (human-readable)
├── events.jsonl            # Append-only structured event log (machine-readable)
├── automations.md          # Draft/confirmed event → action automations
├── notifications.md        # Notify / Remind / Escalate inbox
├── lessons.md              # Persistent lesson set (YAML in body)
├── records/
│   ├── index.md            # Directory listing of records
│   └── <slug>.md           # One concept per record
└── runs/
    └── <date>-<slug>.md    # Per-run summary (keep last 20)
```

`schema.md` and `schema/` are created by [schema setup](schema-setup.md). Bundles that only use default object types may omit them.

`events.jsonl`, `automations.md`, and `notifications.md` are created when first needed (first mutation under Continuity ≥ 0.7, first automation, or first notify action). See [events.md](events.md) and [automations.md](automations.md).

## Workspace schema

When present and `status: confirmed`, `schema.md` is the **authoritative** definition of objects, fields, relationships, and rules for extract/review. Keep `index.md` `object_types` / `object_schemas` in sync as a compatibility mirror.

### schema.md

```yaml
---
schema_version: 1
status: confirmed          # draft | confirmed
updated: 2026-07-20
sources:
  - type: csv
    name: customers.csv
  - type: google_sheet
    name: Pipeline Q3
---
```

Body sections (markdown):

#### Objects

For each object type:

```markdown
## Objects

### Company
- **Description**: Customer or partner organization
- **Identity keys**: `name` (normalized)
- **Status**: (none)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | Text | yes | Display name |
| `domain` | Text | no | Website domain |

### Deal
- **Description**: Sales opportunity
- **Identity keys**: `id` when present; else `name` + `company`
- **Status**: `New`, `Qualified`, `Won`, `Lost`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | Text | yes | Deal name |
| `company` | Text | yes | Related Company name or id |
| `amount` | Currency | no | |
| `status` | Enum | yes | See status list above |
```

Prefer property types from [schema-vocabulary.md](schema-vocabulary.md) (`Text`, `Enum`, `Currency`, `Date`, …).

#### Relationships

```markdown
## Relationships

| From | Field | To | Cardinality | Optional |
|------|-------|----|-------------|----------|
| Deal | `company` | Company | many-to-one | no |
| Task | `project` | Project | many-to-one | yes |
```

#### Rules

```markdown
## Rules

### Validation
- Deal.`status` must be one of the Deal status enum
- Deal.`company` is required

### Deduplication
- Company: match on normalized `name` (and `domain` when both present)
- Deal: match on external `id` when present; else `name` + related Company

### Policy
- (optional) Soft constraints as guidance text — Continuity does not execute schema Policy text as an automation engine
```

Executable Trigger / Schedule / Condition / Action rules belong in `automations.md`, not here. See [automations.md](automations.md).

Label rule kinds using vocabulary names when helpful (`Policy`, `Condition`, `Audit`, …). See [schema-vocabulary.md](schema-vocabulary.md).

#### Open questions

Only while `status: draft` or during discovery. Clear or omit after confirm.

#### Changelog

```markdown
## Changelog

### v1 — 2026-07-20
- Initial schema from customers.csv and Pipeline Q3 sheet
```

### schema/vN.md snapshots

On each confirmed version (initial confirm or approved bump):

1. Write or overwrite `schema.md` with the new content and `schema_version: N`
2. Copy the full confirmed content to `schema/vN.md` (immutable snapshot — do not edit later versions in place)
3. Sync `index.md` (below)
4. Append a log entry noting the schema confirm or bump

### Syncing index.md from schema

Whenever `schema.md` is confirmed or bumped:

1. Set `object_types` to the list of object names from the schema
2. Set `object_schemas` to a compact field map per object (field name → type comment / enum notes), sufficient for extract guidance
3. Optionally add `schema_version` and a body link to `/schema.md`

Do not leave `index.md` object config stale relative to a confirmed schema.

## Root index.md

The bundle root `index.md` is the only `index.md` that may have frontmatter.

**With workspace schema** (preferred when setup has run):

```yaml
---
okf_version: "0.1"
title: <Bundle title>
description: <One-line summary of this knowledge bundle>
schema_version: 1
object_types:
  - Company
  - Deal
  - Task
object_schemas:
  Company:
    name: string
    domain: string
  Deal:
    name: string
    company: string
    amount: number
    status: string  # New, Qualified, Won, Lost
  Task:
    name: string
    status: string
    due_date: string
    project: string
---
```

**Fallback without schema** (defaults from [objects.md](objects.md)):

```yaml
---
okf_version: "0.1"
title: <Bundle title>
description: <One-line summary of this knowledge bundle>
object_types:
  - Task
  - Thread
  - Attendance
  - Appointment
  - Lead
  - Expense
  - Decision
  - Blocker
  - Question
---
```

Body: directory listing with links to `schema.md` (when present), `records/`, `lessons.md`, `log.md`, and recent `runs/`.

Omit `object_schemas` only when using default types with no custom or discovered fields. When a confirmed schema exists, always include the synced `object_schemas` mirror.

## log.md

Chronological history, newest first. Date headings use ISO 8601 `YYYY-MM-DD`.

```markdown
# Bundle Update Log

## 2026-07-11
* **Creation**: Added [Prepare demo](/records/prepare-demo.md) (Task).
* **Update**: Refreshed [Client call with Priya](/records/client-call-priya.md) (Appointment).
* **Deletion**: Removed [Stale blocker](/records/stale-blocker.md) (Blocker).
* **Automation**: Fired `Won deal → invoice task` on [Acme enterprise deal](/records/acme-enterprise-deal.md); created [Send invoice for Acme enterprise deal](/records/send-invoice-for-acme-enterprise-deal.md).

## 2026-07-10
* **Initialization**: Created inwrk bundle structure.
* **Schema**: Confirmed workspace schema v1 ([schema.md](/schema.md)).
```

Entry types: `**Creation**`, `**Update**`, `**Deletion**`, `**Deprecation**`, `**Schema**`, `**Automation**`.

Append entries for every run or canvas apply. When a record is updated in place, log an **Update** entry referencing the record concept. When a record file is removed, log a **Deletion** entry with the former title and type. Log **Schema** when a schema is confirmed or bumped (include version number). Log **Automation** when a confirmed automation fires (include automation name and resulting record links when applicable).

Also append matching lines to `events.jsonl` per [events.md](events.md). Canvas-sourced writes follow the same entry types; note in the record provenance when applied via [canvas-update.md](canvas-update.md).

## lessons.md

A concept document with type `Lessons`. Stores the persistent YAML lesson set in the body.

```markdown
---
type: Lessons
title: Extraction Lessons
description: Persistent rules for record extraction and grouping.
timestamp: 2026-07-11T10:00:00Z
---

# Lessons

```yaml
- object: extraction
  when: "message is a greeting or social chat"
  do: "do not extract a record"
- object: grouping
  when: "two tasks share the same due date but different owners"
  do: "keep them as separate records"
```
```

On first run, create with an empty YAML list. On subsequent runs, load existing lessons before extract and overwrite with the updated set after the lessons step (when corrections warrant an update).

## events.jsonl, automations.md, notifications.md

- **`events.jsonl`** — append-only JSON Lines audit of mutations. Required once any Continuity ≥ 0.7 write has occurred. Format and emission points: [events.md](events.md).
- **`automations.md`** — YAML automations + `event_cursor` frontmatter. Create when the user adds an automation or when evaluation first runs. Format: [automations.md](automations.md).
- **`notifications.md`** — table inbox for Notify / Remind / Escalate. Create on first notify action.

Body of root `index.md` may link to these files when present.

## Record concepts

Each finalized record becomes one markdown file under `records/`. Filename: slug derived from `title` (lowercase, hyphens, no special characters).

### Slug collisions

If `records/<slug>.md` already exists for a **different** `record_id`, append a short suffix from the new record's UUID (first 8 hex chars):

```
prepare-demo.md          # existing record
prepare-demo-a1b2c3d4.md # new record with colliding title
```

If the file exists for the **same** `record_id`, update that file in place (do not create a duplicate).

### Frontmatter

```yaml
---
type: Task
title: Prepare demo
description: Prepare demo for Thursday client meeting
tags: [demo, client]
timestamp: 2026-07-11T10:00:00Z
record_id: "550e8400-e29b-41d4-a716-446655440000"
confidence: high
name: Prepare demo
status: New
due_date: "2026-07-16"
---
```

**Required fields:**

- `type` — object type (e.g., `Task`, `Appointment`, `Decision`)
- `title` — short display name
- `description` — full resolved text from the draft

**Recommended fields:**

- `tags` — cross-cutting labels
- `timestamp` — ISO 8601 last-modified time

**Producer-defined fields (Continuity):**

- `record_id` — stable UUID; reused on updates
- `confidence` — `high`, `medium`, or `low` (see legacy mapping in [objects.md](objects.md))
- Object-specific fields — promoted to frontmatter when scalar; complex fields stay in the body table

Promote simple scalar fields (strings, numbers, booleans) to frontmatter. Keep arrays and long text in the body `# Fields` table.

### Body

```markdown
# Fields

| Field | Value |
|-------|-------|
| `name` | Prepare demo |
| `status` | New |
| `due_date` | 2026-07-16 |
| `participants` | John, Sarah |

# Related

- Part of discussion in [Q4 planning thread](/records/q4-planning-thread.md)

# Citations

[1] "Let's prepare the demo for Thursday" — meeting transcript, 2026-07-11
[2] "John confirmed the demo date" — meeting transcript, 2026-07-11
```

**`# Fields`** — table of structured fields not fully represented in frontmatter.

**`# Related`** (optional) — bundle-relative links to related record concepts.

**`# Citations`** — source excerpts from the transcript that support this record. Use numbered entries with the exact citation text from the draft.

### Update behavior

When a draft has `updates_record_id`:

1. Find the existing record file by matching `record_id` in frontmatter
2. Diff fields for the event `changes` object before overwriting
3. Edit that file in place (do not create a duplicate)
4. Refresh frontmatter, body fields, citations, and `timestamp`
5. Append an **Update** entry to `log.md`
6. Append `record.updated` to `events.jsonl` per [events.md](events.md)

When creating a new record:

1. Generate a new UUID for `record_id`
2. Create `records/<slug>.md` (apply slug-collision rule if needed)
3. Append a **Creation** entry to `log.md`
4. Append `record.created` to `events.jsonl`

When deleting a record (canvas apply or explicit user request):

1. Find the record file by matching `record_id` in frontmatter
2. Remove `records/<slug>.md`
3. Append a **Deletion** entry to `log.md` with the former title and object type
4. Append `record.deleted` to `events.jsonl`
5. Regenerate `records/index.md`

After a batch of creates/updates (extract or canvas): evaluate automations per [automations.md](automations.md).

Canvas-sourced creates, updates, and deletes follow the same rules. See [canvas-update.md](canvas-update.md).

### records/index.md

No frontmatter. Auto-generate or update after each run:

```markdown
# Records

* [Prepare demo](prepare-demo.md) - Prepare demo for Thursday client meeting (Task)
* [Client call with Priya](client-call-priya.md) - Schedule call with Priya tomorrow at 10 (Appointment)
* [Ship without feature flag](ship-without-feature-flag.md) - Agreed to ship without feature flag (Decision)
```

Include the `description` and object type from each record. This index is the **first** place to look when loading prior records or answering queries.

## Prior-record loading (summary-first)

When loading an existing bundle for a new run:

1. Read `records/index.md` into **prior_record_summaries** (title, description, object/type, path)
2. Do **not** load every `records/*.md` file up front
3. During extract, when a draft may update an existing item, load only the candidate record file(s) to obtain `record_id` and fields
4. Pass loaded prior records for `updates_record_id` linkage

If `records/index.md` is missing or empty, fall back to scanning `records/*.md` (excluding `index.md`).

Loaded prior record shape:

```json
{
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "object": "Task",
  "title": "Prepare demo",
  "description": "Prepare demo for Thursday client meeting",
  "fields": { "name": "Prepare demo", "status": "New", "due_date": "2026-07-16" },
  "confidence": "high"
}
```

Legacy numeric `confidence` values map to categorical per [objects.md](objects.md).

## Run traceability

Each pipeline run MAY write a concept under `runs/` for auditability.

Filename: `<YYYY-MM-DD>-<short-slug>.md` (e.g., `2026-07-11-meeting-extract.md`).

Run files are **short summaries** — do not dump full intermediate JSON.

```markdown
---
type: Pipeline Run
title: Meeting transcript extraction
description: Extract → review → write run on 2026-07-11.
timestamp: 2026-07-11T10:30:00Z
---

# Input

Brief description of context source (file names, MCP tools used, paste length).
Anchor date: 2026-07-11

# Summary

- Drafts extracted: 6
- Drafts after review: 4
- Created: 3
- Updated: 1
- User corrections: 1 (merged two Task drafts)
- Lessons updated: 1

# Records produced

- [Prepare demo](/records/prepare-demo.md) (created, Task, high)
- [Client call with Priya](/records/client-call-priya.md) (updated, Appointment, high)
- [Ship without feature flag](/records/ship-without-feature-flag.md) (created, Decision, medium)
- [Blocked on API keys](/records/blocked-on-api-keys.md) (created, Blocker, high)

# Schema misfits

Facts that did not fit the current workspace schema (omit section if none):

- Mention of "warranty claim #8821" — no Claim object in schema — cited: "opened warranty claim 8821"

# Schema proposals

Suggested schema changes from recurring or strong misfits (omit if none). Do not apply without user approval:

- Add object `Claim` with fields `id`, `status`, `related_account` (seen in this run; confirm if it recurs)
```

Include `# Schema misfits` whenever extract finds information that cannot map cleanly onto the confirmed schema. Include `# Schema proposals` when proposing a bump; apply only after the user approves (see [schema-setup.md](schema-setup.md)).

### Retention

Keep the **most recent 20** run files under `runs/`. After writing a new run file, if more than 20 exist, delete the oldest by filename date (then slug). Note deletions briefly in the chat summary when they occur.

## Conformance checklist

A bundle produced by Continuity is well-formed when:

1. Every non-reserved `.md` file has parseable YAML frontmatter with a non-empty `type` (except directory indexes and `schema.md` / `schema/vN.md`, which use schema frontmatter)
2. Root `index.md` declares `okf_version: "0.1"`
3. `index.md` and `log.md` follow reserved-file conventions (root index may have frontmatter; directory indexes do not)
4. Cross-links use bundle-relative paths (`/records/slug.md`, `/schema.md`)
5. New writes use categorical `confidence` (`high` | `medium` | `low`)
6. When `schema.md` exists with `status: confirmed`, `index.md` `object_types` / `object_schemas` match that schema, and `schema/vN.md` exists for the current `schema_version`
7. When mutations have been written under Continuity ≥ 0.7, `events.jsonl` exists and stays in sync with `log.md` for those mutations
8. Confirmed automations live only in `automations.md` (not in `schema.md`); draft automations never execute

## Chat summary

After writing the bundle, report to the user:

- Number of drafts extracted and kept after review
- Records created vs updated (list titles with object types)
- Path to the inwrk bundle
- Any low-confidence records flagged for review
- Lessons changes, if any
- Schema misfits and any schema proposals awaiting approval
- Automations fired and any external actions awaiting approval
