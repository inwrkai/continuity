# Automations (Event → Action Engine)

How Continuity authors, confirms, and evaluates **Trigger + Condition + Action** rules against the event log.

Automations live in `inwrk/automations.md` (not in `schema.md`). Schema Rules remain extraction/validation guidance; executable automation uses this file. Vocabulary labels: Trigger, Schedule, Condition, Action — see [schema-vocabulary.md](schema-vocabulary.md). Event emission: [events.md](events.md).

There is **no daemon**. Evaluation runs lazily whenever the agent finishes a write path (extract Step 5, canvas apply) or when the user asks to check automations / schedules.

---

## When to use

| Trigger phrase / situation | Action |
|----------------------------|--------|
| User says `add an automation`, `when X happens do Y`, `automate …` | Author a draft automation; confirm only after user approval |
| End of extract Step 5 or canvas apply | Emit events, then evaluate confirmed automations |
| User says `check schedules`, `run pending automations`, or opens a session with due reminders | Lazy schedule check + process unprocessed events |
| User confirms / rejects a draft automation | Set `status: confirmed` or leave/delete draft |
| User approves a proposed **external** action | Execute via MCP; record `automation.fired` |

---

## File: `automations.md`

Same pattern as `lessons.md` — YAML list in the body; frontmatter holds the event cursor.

```markdown
---
type: Automations
title: Bundle Automations
description: Confirmed and draft event-driven automations.
timestamp: 2026-07-11T10:00:00Z
event_cursor: evt-a1b2c3d4
fired_keys: []
---

# Automations

```yaml
- id: auto-won-deal-followup
  name: Won deal → invoice task
  status: confirmed
  trigger:
    kind: event
    event: record.updated
    type: Deal
    field: status
    to: Won
  condition:
    all:
      - field: amount
        op: gte
        value: "10000"
  actions:
    - kind: create
      type: Task
      title: "Send invoice for {{title}}"
      fields:
        status: New
        related_deal: "{{record_id}}"
    - kind: notify
      level: info
      message: "Deal {{title}} marked Won — invoice task created."
```
```

### Frontmatter

| Field | Meaning |
|-------|---------|
| `event_cursor` | Last processed event `id`. On evaluate, process only events **after** this id in `events.jsonl`. Empty/missing = process from start (or from first retained event). |
| `fired_keys` | List of schedule idempotency keys already fired (strings). Cap at ~200; drop oldest when over. |
| `timestamp` | Last time automations file was updated |

### Automation object

| Field | Required | Meaning |
|-------|----------|---------|
| `id` | yes | Stable id (`auto-` + slug or short hex). Never invent a new id for an existing rule. |
| `name` | yes | Short human label |
| `status` | yes | `draft` \| `confirmed` — **only `confirmed` executes** |
| `trigger` | yes | When to match (below) |
| `condition` | no | Extra guards after trigger match |
| `actions` | yes | Non-empty list of actions |
| `description` | no | Free text |

---

## Triggers

### Event trigger (`kind: event`)

Matches mutation events (never `source: automation` — cascade guard).

```yaml
trigger:
  kind: event
  event: record.updated   # or record.created, record.deleted, schema.bumped, …
  type: Deal              # optional object type filter
  field: status           # optional: require this key in changes
  from: Qualified         # optional: changes[field].from
  to: Won                 # optional: changes[field].to
```

Match rules:

1. `event` must equal the event line's `event`
2. If `type` set, event's `type` must match
3. If `field` set, `changes` must contain that key
4. If `from` / `to` set, compare to `changes[field].from` / `.to` (string equality)
5. Skip if event `source` is `automation`

For `record.created` / `record.deleted`, `field`/`from`/`to` are usually omitted; optional `type` still applies.

### Schedule trigger (`kind: schedule`)

Evaluated lazily (no cron daemon). Checked when the agent evaluates automations.

```yaml
trigger:
  kind: schedule
  type: Task              # object type to scan
  field: due_date         # date field on the record
  when: within_days       # within_days | overdue | on_date
  days: 2                 # for within_days
```

| `when` | Meaning |
|--------|---------|
| `within_days` | Field date is today or within the next `days` (inclusive) |
| `overdue` | Field date is before today |
| `on_date` | Field date equals today |

**Idempotency key:** `{automation_id}:{record_id}:{YYYY-MM-DD}` using today's date (or the due date for `on_date`). If the key is already in `fired_keys`, skip. After a successful fire, append the key to `fired_keys`.

Schedule triggers scan `records/index.md` then load candidate record files for the target type — summary-first, same discipline as query.

---

## Conditions

Optional guard after the trigger matches. All listed clauses under `all` must pass (AND). Support a single flat `all` list in this version (no nested OR trees required).

```yaml
condition:
  all:
    - field: status
      op: eq
      value: Won
    - field: amount
      op: gte
      value: "10000"
```

| `op` | Meaning |
|------|---------|
| `eq` | String equality |
| `neq` | Not equal |
| `gt` / `gte` / `lt` / `lte` | Numeric compare when both sides parse as numbers; else lexicographic |
| `contains` | Substring (case-insensitive) |
| `exists` | Field present and non-empty (ignore `value`) |

Resolve `field` from the triggering record's fields (for event triggers, load the record by `record_id`; for schedule, the candidate record).

---

## Actions

### Templates

String values may use `{{field}}` interpolation from the triggering record:

- `{{title}}`, `{{record_id}}`, `{{type}}`
- `{{status}}`, `{{amount}}`, … — any scalar field
- Missing keys → empty string

### Internal actions (auto-run)

When a confirmed automation matches, **execute internal actions immediately** without asking. Report them in the run/chat summary.

| `kind` | Behavior |
|--------|----------|
| `create` | Create a new record: `type`, `title`, optional `description`, `fields`. Write per [okf-output.md](okf-output.md). Emit `record.created` with `source: automation`. |
| `update` | Update a record: default target is triggering `record_id`, or `record_id` / `title_match` if set. Merge `fields`. Emit `record.updated` with `source: automation`. |
| `assign` | Shorthand update: set `owner` (or schema owner field) to `value` |
| `archive` | Set status to archived/done equivalent if schema has one; else set `status: Done` or add tag `archived` — prefer schema status enums |
| `restore` | Clear archived / set status back to a provided `status` |
| `duplicate` | Create a copy of the triggering record with new UUID; optional `title` suffix |
| `notify` | Append to `notifications.md` (`level`: `info` \| `warn` \| `urgent`, `message`) |
| `remind` | Same as notify with `level: remind` and optional `due` date in the entry |
| `escalate` | Notify with `level: urgent` and prefix message with `ESCALATION:` |
| `approve` / `reject` | Update a status-like field to Approved/Rejected (or provided `status`) on the target record |

After internal actions for one firing:

1. Append `automation.fired` event (`source: automation`, `meta.actions` = kinds run)
2. Append `**Automation**` entry to `log.md`
3. Regenerate `records/index.md` if records changed

### External actions (propose-and-approve)

Never auto-send. Always propose in chat with the exact payload; execute only after the user approves.

| `kind` | Typical connector |
|--------|-------------------|
| `message` | Chat / WhatsApp / Slack MCP — `to`, `body` |
| `email` | Email MCP — `to`, `subject`, `body` |
| `schedule` | Calendar MCP — `title`, `start`, `end`, `attendees` |

Proposal shape in chat:

```
Automation "Won deal → invoice task" matched Deal "Acme enterprise".
External action pending approval:
- kind: email
  to: billing@acme.example
  subject: Invoice for Acme enterprise
  body: …
Reply **approve** to send, or **skip**.
```

On approve: call the MCP tool; append `automation.fired` with `meta.external: true` and outcome. On skip: append `automation.fired` with `meta.skipped: true` (or omit external fire and note skip in summary only — prefer an event for audit).

If no connector is available, report that and leave the proposal as a notification instead of failing the whole evaluation.

### Interface-only verbs (out of scope)

View, Search, Filter, Sort, Export, Share, Navigate — UI concerns. Do not put them in `actions`.

---

## Authoring protocol

1. User describes intent in natural language, or agent proposes from a recurring pattern
2. Agent drafts YAML in chat (readable outline) and asks to confirm
3. On confirm: write/append to `automations.md` with `status: confirmed`
4. On "save as draft": write with `status: draft`
5. **Never** auto-confirm automations the user has not approved (same ethos as schema bumps)
6. Max practical list: ~30 automations; prefer editing existing ids over duplicates

### Proposal from patterns (optional)

After extract, if the same "when status X do Y" intent appears in user corrections or lessons repeatedly, propose a draft automation in the chat summary — do not write confirmed without approval.

---

## Evaluation protocol

Run after events are appended (extract Step 5, canvas apply), or on explicit user request.

### 1. Load

1. Load `automations.md` (create empty confirmed list + empty cursor if missing when first needed)
2. Load confirmed automations only (`status: confirmed`)
3. Read `events.jsonl` lines after `event_cursor`

### 2. Process event-triggered automations

For each new event in order (oldest first):

1. Skip if `source == automation`
2. For each confirmed automation with `trigger.kind: event`, if trigger + condition match:
   - Run internal actions
   - Queue external actions for chat proposal
   - Emit `automation.fired` + log entry
3. After processing all new events, set `event_cursor` to the last event `id` processed (even if none matched)

### 3. Lazy schedule check

1. For each confirmed `trigger.kind: schedule` automation:
2. Find candidate records of `type` with the date `field`
3. If `when` matches and idempotency key not in `fired_keys`:
   - Evaluate `condition` if present
   - Run actions; append key to `fired_keys`
4. Persist updated `fired_keys` and `timestamp`

### 4. Notifications surfacing

At the start of evaluate (and in Step 6 / apply summary):

1. Read open items from `notifications.md`
2. Mention unacknowledged urgent/remind items in the chat summary
3. User may say `acknowledge notifications` — mark items acknowledged (see below)

### 5. Chat / run summary

Report:

- Automations fired (name + triggering record title)
- Records created/updated by automations
- External actions awaiting approval
- Open notifications count

---

## File: `notifications.md`

Persistent inbox for Notify / Remind / Escalate.

```markdown
---
type: Notifications
title: Bundle Notifications
timestamp: 2026-07-11T10:30:00Z
---

# Notifications

| id | ts | level | message | source_automation | record_id | status |
|----|----|-------|---------|-------------------|-----------|--------|
| n-1a2b | 2026-07-11T10:30:00Z | info | Deal Acme marked Won — invoice task created. | auto-won-deal-followup | 550e… | open |
```

- `status`: `open` \| `acknowledged`
- Append new rows; do not delete history casually
- When acknowledging, set `status` to `acknowledged`
- Keep the table to a few hundred rows; archive old acknowledged rows by removing them if needed

---

## Cascade and safety

1. **Depth 1 only** — automation-sourced events never re-enter the matcher ([events.md](events.md))
2. **Draft never runs** — only `status: confirmed`
3. **External never auto-sends** — propose and wait
4. **Cursor advances** even when nothing matched, so events are not reprocessed forever
5. **Do not** invent record ids for `update` targets; only use triggering id or an existing id from the bundle
6. **Schema fidelity** — create/update fields must exist on the target object type; unknown fields → skip that action and note a misfit in the summary

---

## Do not

- Execute draft automations
- Auto-confirm user-proposed automations without approval
- Fire external MCP actions without explicit user approval in the same conversation
- Re-process events at or before `event_cursor`
- Allow automation cascades (`source: automation` as triggers)
- Store automations inside `schema.md`
- Edit automations from the canvas in this version (canvas is read-only for the Automations panel)
