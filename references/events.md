# Bundle Events (Audit Log)

How Continuity records structured mutations in `inwrk/events.jsonl` — the **Audit** element of the Rules vocabulary.

Events are the machine-readable counterpart to the human-readable `log.md`. Every extract write, canvas apply, schema confirm/bump, and automation firing appends one or more event lines. Confirmed automations evaluate against these events; see [automations.md](automations.md).

Use this reference when writing or reading the event log after Step 5, canvas apply, or schema confirm.

---

## File: `events.jsonl`

Append-only JSON Lines under the bundle root (default `inwrk/events.jsonl`).

- One JSON object per line; no trailing commas; UTF-8
- Newest events are at the **end** of the file (append only)
- Do **not** rewrite or delete historical lines except under the retention rule below
- Create the file on first emission if missing

### Event envelope

```json
{
  "id": "evt-a1b2c3d4",
  "ts": "2026-07-11T10:30:00Z",
  "event": "record.updated",
  "record_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "Deal",
  "title": "Acme enterprise deal",
  "source": "extract",
  "run": "2026-07-11-meeting-extract",
  "changes": {
    "status": { "from": "Qualified", "to": "Won" },
    "amount": { "from": "40000", "to": "52000" }
  },
  "automation_id": null,
  "meta": {}
}
```

| Field | Required | Meaning |
|-------|----------|---------|
| `id` | yes | Stable event id — `evt-` + 8 hex chars (or full UUID). Unique within the bundle. |
| `ts` | yes | ISO 8601 timestamp when the mutation was written |
| `event` | yes | Event type string (see below) |
| `record_id` | when applicable | Target record UUID; omit or null for schema/run-level events |
| `type` | when applicable | Object type name (e.g. `Deal`, `Task`) |
| `title` | no | Display title at emission time (helps feed/UI without loading the record) |
| `source` | yes | `extract` \| `canvas` \| `automation` \| `manual` \| `schema` |
| `run` | no | Run file slug without path (e.g. `2026-07-11-meeting-extract`) when emitted during a pipeline run |
| `changes` | for updates | Map of field name → `{ "from": "...", "to": "..." }`. Strings only. Include only fields that actually changed. |
| `automation_id` | for `automation.fired` | Id of the automation that fired |
| `meta` | no | Small opaque bag (e.g. `ops_count`, `schema_version`, `actions_run`) |

### Event types

| `event` | When emitted | Typical `source` |
|---------|--------------|------------------|
| `record.created` | New `records/<slug>.md` written | `extract`, `canvas`, `automation` |
| `record.updated` | Existing record edited in place | `extract`, `canvas`, `automation` |
| `record.deleted` | Record file removed | `canvas`, `manual` |
| `schema.confirmed` | First confirmed `schema.md` | `schema` |
| `schema.bumped` | Approved schema version bump | `schema` |
| `run.completed` | End of extract Step 5 (after writes) | `extract` |
| `automation.fired` | A confirmed automation executed or proposed external actions | `automation` |

---

## Emission points

Emit events **after** the corresponding disk write succeeds. Do not emit for validation failures that wrote nothing.

### Extract / update (SKILL.md Step 5)

After creating/updating records and appending `log.md`:

1. For each created record → `record.created` (`source: extract`)
2. For each updated record → `record.updated` with `changes` for fields that differ from the prior file
3. Append `run.completed` with `meta` counts (`created`, `updated`, drafts kept)
4. Then evaluate automations per [automations.md](automations.md)

### Canvas apply ([canvas-update.md](canvas-update.md))

After each successful op write:

1. `create` → `record.created` (`source: canvas`)
2. `update` → `record.updated` with `changes` (`source: canvas`)
3. `delete` → `record.deleted` (`source: canvas`)
4. Then evaluate automations

### Schema confirm / bump ([schema-setup.md](schema-setup.md))

1. First confirm → `schema.confirmed` with `meta.schema_version`
2. Approved bump → `schema.bumped` with `meta.schema_version`
3. Automations usually do not trigger on schema events unless an automation explicitly matches them

### Automation runner ([automations.md](automations.md))

1. Each firing → `automation.fired` (`source: automation`, `automation_id` set)
2. Side-effect record creates/updates from **internal** actions → `record.created` / `record.updated` with `source: automation`

---

## Cascade guard (depth 1)

Events with `source: automation` **must not** re-trigger automations.

When evaluating automations:

1. Consider only events where `source` is `extract`, `canvas`, `manual`, or `schema` (never `automation`)
2. Still **append** automation-sourced record events to `events.jsonl` for audit — they are visible in the Activity feed but skipped by the matcher
3. Do not implement multi-hop cascades in this skill version

This prevents loops (e.g. automation creates a Task → would create another Task forever).

---

## Diffing `changes` for updates

When emitting `record.updated`:

1. Load the prior frontmatter / fields before overwriting (extract and canvas already do this for identity)
2. Compare scalar fields and known `# Fields` table values
3. Emit only keys whose stringified values differ
4. Use empty string `""` for `from` when a field was previously absent; omit `to` only if clearing (prefer `""` for cleared)
5. If nothing changed except `timestamp` / citations, still emit `record.updated` with `changes: {}` when the write was intentional (canvas apply, extract refresh) — automations that require field-level conditions will not match empty or unrelated changes

---

## Retention

Keep the event log practical for agents:

1. Prefer retaining at least the last **500** events or **90 days**, whichever is larger in practice for the bundle
2. If the file grows beyond ~1000 lines, the agent MAY truncate the **oldest** lines while preserving the automations cursor (see [automations.md](automations.md) — if truncating past the cursor, reset cursor to the oldest remaining `id` and note it in the chat summary)
3. Never rewrite mid-file except for that ordered truncation

---

## Reading events (summary-first)

1. For Activity feed / canvas: read the **last N** lines (e.g. 30–50), newest last — reverse for display
2. For automation evaluation: read lines after the cursor id (see automations frontmatter `event_cursor`)
3. Do not load the entire file into context when only a tail is needed

---

## Relationship to `log.md`

| Concern | `log.md` | `events.jsonl` |
|---------|----------|----------------|
| Audience | Humans | Agents / automations / canvas feed |
| Format | Markdown bullets | JSON Lines |
| Field diffs | Rarely | `changes` object |
| Required | Yes | Yes once any mutation has been written under Continuity ≥ 0.7 |

Always keep both in sync for the same mutation: append the human log entry **and** the event line.

---

## Do not

- Emit events before the disk write succeeds
- Use `source: automation` events as automation triggers
- Invent `record_id` values in events that do not match written records
- Store full record bodies or large transcript excerpts in `meta` or `changes`
- Delete `events.jsonl` to "reset" without also resetting `automations.md` `event_cursor`
