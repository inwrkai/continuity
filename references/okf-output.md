# OKF Output Format

Rules for writing okf-brain pipeline output as a conformant [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) knowledge bundle.

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
    └── <date>-<slug>.md    # Per-run traceability (signals, bundles, feedback)
```

## Root index.md

The bundle root `index.md` is the only `index.md` that may have frontmatter (per OKF §11).

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
description: Persistent rules for signal extraction and bundle grouping.
timestamp: 2026-07-11T10:00:00Z
---

# Lessons

```yaml
- object: extraction
  when: "message is a greeting or social chat"
  do: "do not extract a signal"
- object: grouping
  when: "two tasks share the same due date but different owners"
  do: "keep them in separate bundles"
```
```

On first run, create with an empty YAML list. On subsequent runs, load existing lessons before signals extraction and overwrite with the updated set after the lessons stage.

## Record concepts

Each finalized record becomes one markdown file under `records/`. Filename: slug derived from `title` (lowercase, hyphens, no special characters).

### Frontmatter

```yaml
---
type: Task
title: Prepare demo
description: Prepare demo for Thursday client meeting
tags: [demo, client]
timestamp: 2026-07-11T10:00:00Z
record_id: "550e8400-e29b-41d4-a716-446655440000"
confidence: 0.92
name: Prepare demo
status: New
due_date: Thursday
---
```

**Required OKF fields:**

- `type` — object type (e.g., `Task`, `Appointment`)
- `title` — short display name
- `description` — full resolved text from the bundle

**Recommended OKF fields:**

- `tags` — cross-cutting labels
- `timestamp` — ISO 8601 last-modified time

**Producer-defined fields (okf-brain):**

- `record_id` — stable UUID; reused on updates
- `confidence` — 0–1 score from the pipeline
- Object-specific fields — promoted to frontmatter when scalar; complex fields stay in the body table

Promote simple scalar fields (strings, numbers, booleans) to frontmatter. Keep arrays and long text in the body `# Fields` table.

### Body

```markdown
# Fields

| Field | Value |
|-------|-------|
| `name` | Prepare demo |
| `status` | New |
| `due_date` | Thursday |
| `participants` | John, Sarah |

# Related

- Part of discussion in [Q4 planning thread](/records/q4-planning-thread.md)

# Citations

[1] "Let's prepare the demo for Thursday" — meeting transcript, 2026-07-11
[2] "John confirmed the demo date" — meeting transcript, 2026-07-11
```

**`# Fields`** — table of structured fields not fully represented in frontmatter.

**`# Related`** (optional) — bundle-relative links to related record concepts.

**`# Citations`** — source excerpts from the transcript that support this record. Use numbered entries with the exact `source_excerpt` text from signals.

### Update behavior

When a bundle has `updates_record_id`:

1. Find the existing record file by matching `record_id` in frontmatter
2. Edit that file in place (do not create a duplicate)
3. Refresh frontmatter, body fields, citations, and `timestamp`
4. Append an **Update** entry to `log.md`

When creating a new record:

1. Generate a new UUID for `record_id`
2. Create `records/<slug>.md`
3. Append a **Creation** entry to `log.md`

### records/index.md

No frontmatter (per OKF §6). Auto-generate or update after each run:

```markdown
# Records

* [Prepare demo](prepare-demo.md) - Prepare demo for Thursday client meeting
* [Client call with Priya](client-call-priya.md) - Schedule call with Priya tomorrow at 10
```

Include the `description` from each record's frontmatter.

## Run traceability

Each pipeline run MAY write a concept under `runs/` for auditability.

Filename: `<YYYY-MM-DD>-<short-slug>.md` (e.g., `2026-07-11-meeting-extract.md`).

```markdown
---
type: Pipeline Run
title: Meeting transcript extraction
description: Signals → bundles → feedback → records run on 2026-07-11.
timestamp: 2026-07-11T10:30:00Z
---

# Input

Brief description of context source (file names, MCP tools used, paste length).

# Signals

```json
[ ... ]
```

# Bundles

```json
{ "bundles": [ ... ] }
```

# Feedback

```json
[ ... ]
```

# Records produced

- [Prepare demo](/records/prepare-demo.md) (created)
- [Client call with Priya](/records/client-call-priya.md) (updated)
```

## Prior confirmed records

When loading an existing bundle for a new run:

1. Read all files under `records/*.md` (excluding `index.md`)
2. Parse frontmatter into `prior_confirmed_records` JSON:

```json
[
  {
    "record_id": "550e8400-e29b-41d4-a716-446655440000",
    "object": "Task",
    "title": "Prepare demo",
    "description": "Prepare demo for Thursday client meeting",
    "fields": { "name": "Prepare demo", "status": "New", "due_date": "Thursday" },
    "confidence": 0.92
  }
]
```

3. Pass this array to the bundles stage for `updates_record_id` linkage

## Conformance checklist

A bundle produced by okf-brain is OKF v0.1 conformant when:

1. Every non-reserved `.md` file has parseable YAML frontmatter with a non-empty `type`
2. Root `index.md` declares `okf_version: "0.1"`
3. `index.md` and `log.md` follow OKF reserved-file conventions
4. Cross-links use bundle-relative paths (`/records/slug.md`)

## Chat summary

After writing the bundle, report to the user:

- Number of signals extracted
- Number of bundles formed
- Number of feedback actions applied
- Records created vs updated (list titles)
- Path to the OKF bundle
- Any low-confidence records flagged for review
