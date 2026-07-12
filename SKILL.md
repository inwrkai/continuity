---
name: okf-brain
description: >-
  Converts conversation context into organized OKF v0.1 knowledge records using a
  signals-bundles-feedback-records-lessons pipeline. Use when the user mentions
  okf-brain, OKF, convert to records, organize context into structured records, or
  wants transcripts or documents turned into actionable knowledge records.
license: MIT
compatibility: Works with any Agent Skills-compatible agent (Cursor, Claude Code, etc.). Writes output to the workspace; no external services required.
metadata:
  author: okf-brain
  version: "0.1.0"
  homepage: https://github.com/anubhavmisra/okf-brain
---

# okf-brain — Context to OKF Records

Converts any provided context (attachments, MCP data, pasted transcripts) into organized knowledge records written as an OKF v0.1 bundle on disk.

This skill is self-contained. All stage instructions, object schemas, and output format rules live in this skill's `references/` directory.

## When to use

Trigger when the user:

- Mentions **okf-brain**, **OKF**, or **convert to records**
- Asks to **organize context** into structured records
- Provides a transcript or documents and wants actionable records extracted
- Wants to update an existing OKF bundle with new context

## Reference files

| File | Purpose |
|------|---------|
| [references/stages.md](references/stages.md) | Detailed instructions for signals, bundles, feedback, records, and lessons stages |
| [references/objects.md](references/objects.md) | Default object types (Task, Thread, Attendance, Appointment, Lead, Expense) and field schemas |
| [references/okf-output.md](references/okf-output.md) | OKF v0.1 bundle layout, frontmatter rules, and write/update behavior |

Read the relevant reference before executing each stage. For a navigation index, see [AGENTS.md](AGENTS.md).

---

## 8-step workflow

Execute all eight steps in order unless the user explicitly requests a partial run.

### Step 1: Gather context

Collect everything the user provided as the **transcript**:

- Pasted text in chat
- Attached files (read their contents)
- MCP tool results the user asked you to fetch
- Images or PDFs described in context (note visual content in the transcript)

Normalize into a single transcript string. Preserve speaker labels, timestamps, and message order when present. Note the source (file names, tools used) for the run traceability file.

If no context is provided, ask the user for input before proceeding.

### Step 2: Locate or create the OKF bundle

Default output directory: `okf/` in the current workspace. Honor a user-specified path if given.

**If the bundle exists:**

- Load `lessons.md` body (YAML lesson list)
- Load all records under `records/*.md` into `prior_confirmed_records` (see [okf-output.md](references/okf-output.md))
- Read `index.md` frontmatter for `object_types` and `object_schemas`

**If the bundle does not exist:**

- Create the directory structure per [okf-output.md](references/okf-output.md)
- Initialize empty lessons
- Set `prior_confirmed_records` to `[]`
- Use default object types from [objects.md](references/objects.md) unless the user scoped otherwise

Confirm the object type scope with the user only if ambiguous. Default to all six types.

### Step 3: Signals extraction

Follow [stages.md — Stage 1: Signals](references/stages.md#stage-1-signals).

**Inputs:** transcript, lessons, object_types, object_schemas

**Output:** JSON array of signals with `signal_id`, `verb`, `object`, `summary`, `fields`, `source_excerpt`, `confidence`

**Rules:**

- Verbs: `NEW`, `UPDATED`, `MENTIONED`, `REMOVED`, `REQUESTED`, `CONFIRMED`
- Apply lessons when patterns match
- Do not hallucinate fields
- Prefer precision over recall

### Step 4: Bundles grouping

Follow [stages.md — Stage 2: Bundles](references/stages.md#stage-2-bundles).

**Inputs:** signals, prior_confirmed_records

**Output:** `{ "bundles": [ ... ] }` with `object`, `resolved_text`, `signals`, `confidence`, `updates_record_id`

**Rules:**

- One bundle = one record instance
- All signals in a bundle share the same object
- Link to prior records via `updates_record_id` when clearly the same item
- Do not invent record IDs

### Step 5: Feedback self-review

Follow [stages.md — Stage 3: Feedback](references/stages.md#stage-3-feedback).

**Inputs:** transcript, signals, bundles

**Output:** JSON array of feedback actions (`merge_bundles`, `split_bundle`, `move_signal`, `ignore_signal`, `edit_bundle`)

**Rules:**

- Use transcript as source of truth
- Be conservative — prefer fewer, high-confidence suggestions
- Do not invent new signals or bundles

**User corrections:** If the user provides corrections during the session, treat them as additional feedback actions and apply them in Step 6.

### Step 6: Records finalization

Follow [stages.md — Stage 4: Records](references/stages.md#stage-4-records).

**Inputs:** signals, bundles, feedback (including any user corrections)

**Output:** JSON array of finalized records with `record_id`, `object`, `title`, `description`, `fields`, `confidence`

**Rules:**

- Apply all feedback before generating records
- Reuse `record_id` when `updates_record_id` is set
- `description` = bundle `resolved_text` (unchanged)
- `title` = short derived title (3–7 words)
- One bundle → one record

### Step 7: Lessons update

Follow [stages.md — Stage 5: Lessons](references/stages.md#stage-5-lessons).

**Inputs:** existing lessons, signals, bundles, feedback

**Output:** Updated YAML lesson list (max 30 lessons)

**Rules:**

- Only derive lessons from feedback evidence
- Preserve high-quality existing lessons
- Return the full updated set, not a diff

Skip lesson changes if feedback array is empty and the user provided no corrections.

### Step 8: Write OKF bundle and summary

Follow [okf-output.md](references/okf-output.md) to write all files.

**Write or update:**

1. `okf/index.md` — root index with `okf_version: "0.1"` and object config
2. `okf/lessons.md` — updated lesson YAML
3. `okf/records/<slug>.md` — one file per record (create or update in place)
4. `okf/records/index.md` — regenerate directory listing
5. `okf/log.md` — append Creation/Update entries for this run
6. `okf/runs/<date>-<slug>.md` — run traceability with signals, bundles, feedback

**Report a chat summary:**

- Signals extracted (count)
- Bundles formed (count)
- Feedback actions applied (count)
- Records created vs updated (list titles with object types)
- Output path (`okf/` or custom)
- Low-confidence records (< 0.5) flagged for user review

---

## Object type scoping

Default types: `Task`, `Thread`, `Attendance`, `Appointment`, `Lead`, `Expense`.

The user may request a subset:

```
okf-brain this transcript — only Task and Appointment
```

Custom types go in the bundle root `index.md` frontmatter. See [objects.md](references/objects.md).

---

## Handling updates

When `okf/` already exists:

1. Existing records become `prior_confirmed_records`
2. Bundles with `updates_record_id` update the matching record file in place
3. New bundles create new record files with fresh UUIDs
4. `log.md` gets Update entries for modified records, Creation entries for new ones
5. Lessons accumulate across runs (max 30)

---

## Quality principles

1. **Precision over recall** — skip uncertain signals and low-confidence bundles
2. **No hallucination** — only extract fields supported by the transcript
3. **Immutable signals** — correct via feedback actions, not by editing signal text
4. **Traceability** — preserve source excerpts in record citations and run files
5. **Standalone** — do not import, shell out to, or reference external pipeline code

---

## Example session

**User:** "okf-brain — here's our standup transcript: [paste]"

**Agent:**

1. Gathers pasted text as transcript
2. Creates `okf/` (or loads existing bundle)
3. Extracts signals (e.g., 12 signals across Task and Appointment)
4. Groups into bundles (e.g., 5 bundles, 1 with `updates_record_id`)
5. Self-reviews with 2 feedback actions (1 `merge_bundles`, 1 `ignore_signal`)
6. Produces 4 finalized records
7. Updates lessons with 1 new grouping rule
8. Writes OKF bundle and reports:

```
okf-brain run complete
- 12 signals → 4 records (3 created, 1 updated)
- 2 feedback actions applied
- Output: okf/
- Review: "Fix login bug" (confidence 0.45) — low confidence, please verify
```

---

## Partial runs

If the user requests only a specific stage (e.g., "just extract signals"), run that stage and return JSON output without writing the full bundle. Default to the full 8-step workflow unless explicitly told otherwise.

## Errors and edge cases

- **Empty transcript:** Ask for context; do not proceed
- **No signals found:** Report zero extraction; still update the run file if requested
- **All bundles below confidence 0.5:** Report findings but warn that no records meet the quality threshold; ask user whether to proceed
- **Conflicting prior records:** Prefer null `updates_record_id` over incorrect linkage
