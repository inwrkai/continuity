---
name: ops-brain
description: >-
  Turn work context into portable operational records for AI agents. Extracts,
  maintains, and queries tasks, decisions, blockers, and other operational
  truth from conversations, documents, and connected sources via an
  extract-review-write pipeline. Use when the user mentions ops-brain, inwrk
  Ops Brain, operational context, convert to records, extract
  tasks/decisions/blockers from a transcript, update an OKF bundle, or asks
  about records already stored in okf/.
license: MIT
compatibility: Works with any Agent Skills-compatible agent (Cursor, Claude Code, etc.). Writes output to the workspace; no external services required.
metadata:
  author: inwrk
  version: "0.2.0"
  homepage: https://github.com/inwrkai/ops-brain
---

# inwrk Ops Brain

**Turn work context into portable operational records for AI agents.**

The open operational-context layer for inwrk: extract, maintain, and query tasks, decisions, blockers, and other operational truth from conversations, documents, and connected sources.

Converts provided context (attachments, MCP data, pasted transcripts) into organized operational records written as an OKF bundle on disk. Also answers questions from an existing bundle without re-extracting.

This skill is self-contained. All stage instructions, object schemas, output format rules, and query guidance live in this skill's `references/` directory.

## When to use

**Extract / update** when the user:

- Mentions **ops-brain**, **inwrk Ops Brain**, **operational context**, or **convert to records**
- Provides a transcript or documents and wants actionable records extracted
- Wants to update an existing OKF bundle with new context

**Query** when the user:

- Asks about records already in `okf/` (open tasks, blockers, what changed, etc.)
- Wants a summary or filter of an existing bundle without new input

## Reference files

| File | Purpose |
|------|---------|
| [references/stages.md](references/stages.md) | Extract, review, and write/lessons stage instructions |
| [references/objects.md](references/objects.md) | Default object types and field schemas |
| [references/okf-output.md](references/okf-output.md) | Bundle layout, frontmatter rules, and write/update behavior |
| [references/query.md](references/query.md) | How to answer questions from an existing bundle |

Read the relevant reference before executing each stage. For a navigation index, see [AGENTS.md](AGENTS.md).

---

## Querying a bundle

If the user is asking about an existing bundle (not providing new context to convert), follow [query.md](references/query.md):

1. Read `okf/index.md` and `okf/records/index.md`
2. Load only matching record files
3. Answer with citations; do not re-run extraction

---

## 6-step workflow (extract / update)

Execute all six steps in order unless the user explicitly requests a partial run or is in query mode.

### Step 1: Gather context

Collect everything the user provided as the **transcript**:

- Pasted text in chat
- Attached files (read their contents)
- MCP tool results the user asked you to fetch
- Images or PDFs described in context (note visual content in the transcript)

Normalize into a single transcript string. Preserve speaker labels, timestamps, and message order when present. Note the source (file names, tools used) for the run summary.

Determine **anchor_date**: prefer an explicit transcript date; otherwise use today's date (ISO 8601 `YYYY-MM-DD`). Relative dates in fields resolve against this anchor.

If no context is provided and the user is not querying an existing bundle, ask for input before proceeding.

### Step 2: Locate or create the OKF bundle

Default output directory: `okf/` in the current workspace. Honor a user-specified path if given.

**If the bundle exists:**

- Load `lessons.md` body (YAML lesson list)
- Load `records/index.md` into `prior_record_summaries` (do not load every record file yet)
- Read `index.md` frontmatter for `object_types` and `object_schemas`
- Load full prior record files only when a draft clearly matches one (see [okf-output.md](references/okf-output.md))

**If the bundle does not exist:**

- Create the directory structure per [okf-output.md](references/okf-output.md)
- Initialize empty lessons
- Set `prior_record_summaries` to empty
- Use default object types from [objects.md](references/objects.md) unless the user scoped otherwise

Confirm the object type scope with the user only if ambiguous. Default to all nine types.

### Step 3: Extract draft records

Follow [stages.md — Stage 1: Extract](references/stages.md#stage-1-extract).

**Inputs:** transcript, lessons, object_types, object_schemas, prior_record_summaries, anchor_date

**Output:** JSON array of draft records with `object`, `title`, `description`, `fields`, `citations`, `confidence` (`high` | `medium` | `low`), `updates_record_id`

**Rules:**

- One draft = one record instance
- Apply lessons when patterns match
- Do not hallucinate fields
- Resolve relative dates against `anchor_date`
- Never invent `record_id` values
- Prefer precision over recall

### Step 4: Review drafts

Follow [stages.md — Stage 2: Review](references/stages.md#stage-2-review).

**Inputs:** transcript, drafts, optional user corrections

**Output:** Revised drafts array (same schema). Edit drafts in place — merge, split, drop, edit, or relink. Do not emit a separate feedback-action vocabulary.

**Rules:**

- Use transcript as source of truth
- Be conservative — prefer fewer, high-confidence drafts
- Apply user corrections as authoritative
- Drop anything below `low` confidence

### Step 5: Write OKF bundle and update lessons

Follow [stages.md — Stage 3: Write and lessons](references/stages.md#stage-3-write-and-lessons) and [okf-output.md](references/okf-output.md).

**Write or update:**

1. `okf/index.md` — root index with `okf_version: "0.1"` and object config
2. `okf/lessons.md` — updated lesson YAML when corrections warrant it
3. `okf/records/<slug>.md` — one file per record (create or update in place; handle slug collisions)
4. `okf/records/index.md` — regenerate directory listing
5. `okf/log.md` — append Creation/Update entries for this run
6. `okf/runs/<date>-<slug>.md` — short run summary (not full intermediate JSON); retain last 20 runs

**Lessons:** Update primarily from user corrections. Add a lesson from a self-review catch only when the same pattern recurs. Skip if there is nothing to learn. Max 30 lessons.

### Step 6: Chat summary

Report:

- Drafts extracted → kept after review
- Records created vs updated (list titles with object types)
- Lessons added/updated (if any)
- Output path (`okf/` or custom)
- Low-confidence records flagged for user review

---

## Object type scoping

Default types: `Task`, `Thread`, `Attendance`, `Appointment`, `Lead`, `Expense`, `Decision`, `Blocker`, `Question`.

The user may request a subset:

```
ops-brain this transcript — only Task, Decision, and Blocker
```

Custom types go in the bundle root `index.md` frontmatter. See [objects.md](references/objects.md).

---

## Handling updates

When `okf/` already exists:

1. `records/index.md` becomes prior summaries; load full files only for likely matches
2. Drafts with `updates_record_id` update the matching record file in place
3. New drafts create new record files with fresh UUIDs
4. `log.md` gets Update entries for modified records, Creation entries for new ones
5. Lessons accumulate across runs (max 30), driven mainly by user corrections

---

## Quality principles

1. **Precision over recall** — skip uncertain drafts
2. **No hallucination** — only extract fields supported by the transcript
3. **Date anchoring** — resolve relative dates to ISO 8601 against `anchor_date`
4. **Categorical confidence** — use `high` / `medium` / `low`, not numeric scores
5. **Traceability** — preserve source excerpts in record citations and run summaries
6. **Standalone** — do not import, shell out to, or reference external pipeline code
7. **Summary-first loading** — do not load every record on every run

---

## Example session

**User:** "ops-brain — here's our standup transcript: [paste]"

**Agent:**

1. Gathers pasted text as transcript; sets `anchor_date`
2. Creates `okf/` (or loads existing bundle summaries)
3. Extracts drafts (e.g., 6 drafts across Task, Decision, Blocker)
4. Reviews — merges 1 duplicate, drops 1 noise item → 4 drafts
5. Writes OKF bundle; adds 1 lesson if user corrected something
6. Reports:

```
ops-brain run complete
- 6 drafts → 4 records (3 created, 1 updated)
- Output: okf/
- Review: "Fix login bug" (confidence low) — please verify
```

---

## Partial runs

If the user requests only a specific stage (e.g., "just extract drafts"), run that stage and return JSON output without writing the full bundle. Default to the full 6-step workflow unless explicitly told otherwise.

## Errors and edge cases

- **Empty transcript (extract mode):** Ask for context; do not proceed
- **No drafts found:** Report zero extraction; still write a run summary if useful
- **All drafts low confidence:** Report findings, flag for review, ask whether to write
- **Conflicting prior records:** Prefer null `updates_record_id` over incorrect linkage
- **Query with missing bundle:** Tell the user `okf/` was not found; offer to create one from new context
