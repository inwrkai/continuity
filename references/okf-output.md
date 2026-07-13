# OKF Output Format

Rules for writing ops-brain pipeline output as a portable knowledge bundle on disk.

The layout is self-contained markdown with YAML frontmatter. It is designed to be compatible with [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) where useful; agents should follow the rules in this file even without consulting the external spec.

## Bundle layout

Default output directory: `okf/` in the current workspace (user may override).

```
okf/
├── index.md                # Root index with okf_version and object config
├── log.md                  # Chronological update history
├── lessons.md              # Persistent lesson set (YAML in body)
├── records/
│   ├── index.md            # Directory listing of records
│   └── <slug>.md           # One concept per record
└── runs/
    └── <date>-<slug>.md    # Per-run summary (keep last 20)
```

## Root index.md

The bundle root `index.md` is the only `index.md` that may have frontmatter.

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

Body: directory listing with links to `records/`, `lessons.md`, `log.md`, and recent `runs/`.

Omit `object_schemas` unless custom types are defined (see [objects.md](objects.md)).

## log.md

Chronological history, newest first. Date headings use ISO 8601 `YYYY-MM-DD`.

```markdown
# Bundle Update Log

## 2026-07-11
* **Creation**: Added [Prepare demo](/records/prepare-demo.md) (Task).
* **Update**: Refreshed [Client call with Priya](/records/client-call-priya.md) (Appointment).

## 2026-07-10
* **Initialization**: Created OKF bundle structure.
```

Entry types: `**Creation**`, `**Update**`, `**Deprecation**`.

Append entries for every run. When a record is updated in place, log an **Update** entry referencing the record concept.

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

**Producer-defined fields (ops-brain):**

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
2. Edit that file in place (do not create a duplicate)
3. Refresh frontmatter, body fields, citations, and `timestamp`
4. Append an **Update** entry to `log.md`

When creating a new record:

1. Generate a new UUID for `record_id`
2. Create `records/<slug>.md` (apply slug-collision rule if needed)
3. Append a **Creation** entry to `log.md`

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
```

### Retention

Keep the **most recent 20** run files under `runs/`. After writing a new run file, if more than 20 exist, delete the oldest by filename date (then slug). Note deletions briefly in the chat summary when they occur.

## Conformance checklist

A bundle produced by ops-brain is well-formed when:

1. Every non-reserved `.md` file has parseable YAML frontmatter with a non-empty `type`
2. Root `index.md` declares `okf_version: "0.1"`
3. `index.md` and `log.md` follow reserved-file conventions (root index may have frontmatter; directory indexes do not)
4. Cross-links use bundle-relative paths (`/records/slug.md`)
5. New writes use categorical `confidence` (`high` | `medium` | `low`)

## Chat summary

After writing the bundle, report to the user:

- Number of drafts extracted and kept after review
- Records created vs updated (list titles with object types)
- Path to the OKF bundle
- Any low-confidence records flagged for review
- Lessons changes, if any
