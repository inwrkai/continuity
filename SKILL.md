---
name: continuity
description: >-
  Extract records from chats and discover a workspace schema from spreadsheets
  or an interview. LLMs are great at summarizing but not good at records —
  point this at any chat with your connectors and get structured records out,
  with grounding from other relevant sources. Use when the user mentions
  continuity, discover schema, extract records from chats, convert a
  conversation into records, update an inwrk bundle, or asks about records
  already stored in inwrk/.
license: MIT
compatibility: Works with any Agent Skills-compatible agent (Cursor, Claude Code, etc.). Writes output to the workspace; no external services required.
metadata:
  author: inwrk
  version: "0.4.0"
  homepage: https://github.com/inwrkai/continuity
---

# Continuity

**Extract records from chats — against a discovered workspace schema.**

LLMs are great at summarizing and going through stuff but not good at records. Point Continuity at any chat with your connectors and get it to extract records. Provide grounding with other relevant sources.

Every business already has an implicit schema across spreadsheets, documents, and conversations. Prefer discovering that schema (setup) over only imposing Continuity's default nine types. Once confirmed, recurring runs create and update records against it, flag misfits, and propose schema changes when patterns recur.

Writes portable records on disk (`inwrk/` by default). Also answers questions from an existing bundle without re-extracting.

This skill is self-contained. All stage instructions, object schemas, output format rules, and query guidance live in this skill's `references/` directory.

## Step 0: Check for updates (installed copies only)

Run this **once** before query mode, schema setup, or the 6-step workflow. Do not block the user's request on a failed check.

1. Resolve `SKILL_DIR` = the directory that contains this `SKILL.md`.
2. **Skip** (development or non-GitHub install) when any of these are true:
   - The current workspace root is `SKILL_DIR` (you are editing/developing this repo)
   - `SKILL_DIR` is not a git repository
   - `git -C "$SKILL_DIR" remote get-url origin` does not contain `github.com/inwrkai/continuity` (also accept legacy `github.com/inwrkai/ops-brain`)
3. **Otherwise** (cloned install from GitHub), update if behind:

```bash
git -C "$SKILL_DIR" fetch origin --quiet
LOCAL=$(git -C "$SKILL_DIR" rev-parse HEAD)
REMOTE=$(git -C "$SKILL_DIR" rev-parse @{u} 2>/dev/null || git -C "$SKILL_DIR" rev-parse origin/main 2>/dev/null || true)
if [ -n "$REMOTE" ] && [ "$LOCAL" != "$REMOTE" ]; then
  git -C "$SKILL_DIR" merge-base --is-ancestor "$LOCAL" "$REMOTE" 2>/dev/null \
    && git -C "$SKILL_DIR" pull --ff-only --quiet
fi
```

4. If the pull succeeded, briefly tell the user Continuity was updated, **re-read** this `SKILL.md` (and any `references/` files you already loaded), then continue with the updated instructions.
5. If fetch/pull fails (offline, dirty tree, diverged history), continue with the current copy — do not ask the user to fix it unless they explicitly want an upgrade.

## When to use

**Schema setup** when the user:

- Says `continuity setup`, `discover schema`, `set up my workspace schema`, or equivalent
- Attaches CSVs / Sheets for schema discovery with no confirmed `inwrk/schema.md` yet

**Extract / update** when the user:

- Mentions **continuity**, **extract records from chats**, or wants records from a conversation
- Points at a chat (pasted transcript, connectors/MCP, attached exports) and wants actionable records
- Wants grounding from other sources alongside the chat
- Wants to update an existing inwrk bundle with new context

**Query** when the user:

- Asks about records already in `inwrk/` (open tasks, blockers, what changed, etc.)
- Wants a summary or filter of an existing bundle without new input

## Reference files

| File | Purpose |
|------|---------|
| [references/stages.md](references/stages.md) | Extract, review, and write/lessons stage instructions |
| [references/objects.md](references/objects.md) | Fallback default object types and field schemas |
| [references/schema-setup.md](references/schema-setup.md) | Discover and confirm a workspace schema |
| [references/schema-vocabulary.md](references/schema-vocabulary.md) | Suggestion vocabulary for objects, properties, rules |
| [references/okf-output.md](references/okf-output.md) | Bundle layout, schema files, frontmatter, write/update behavior |
| [references/query.md](references/query.md) | How to answer questions from an existing bundle |

Read the relevant reference before executing each stage. For a navigation index, see [AGENTS.md](AGENTS.md).

---

## Schema setup

When setup is requested (or required before first extract — see Step 2), follow [schema-setup.md](references/schema-setup.md):

1. Path A — profile CSVs / Google Sheets / tables; propose objects, fields, relationships, rules
2. Path B — short interview if no tabular data
3. Ask only questions the data cannot resolve; user confirms
4. Write `inwrk/schema.md` + `inwrk/schema/vN.md`; sync `index.md`

Bundled demo sources (no connectors): [sample/README.md](sample/README.md) and `sample/sources/`.

Trigger phrases: `continuity setup`, `discover schema`, `set up my workspace schema`.

If the first extract includes CSV/Sheet attachments and `schema.md` is missing, run setup before extract unless the user declines and asks to use defaults.

---

## Querying a bundle

If the user is asking about an existing bundle (not providing new context to convert), run Step 0 first, then follow [query.md](references/query.md):

1. Read `inwrk/index.md` and `inwrk/records/index.md` (and `schema.md` when present for type names)
2. Load only matching record files
3. Answer with citations; do not re-run extraction

---

## 6-step workflow (extract / update)

Execute Step 0, then all six steps in order unless the user explicitly requests a partial run, schema-setup-only, or is in query mode.

### Step 1: Gather context

Collect the **chat** (and any grounding) as the **transcript**:

- The conversation the user pointed at (pasted text, thread export, or chat fetched via connectors / MCP)
- Attached files and other grounding sources the user provided or asked you to pull in
- MCP / connector results the user asked you to fetch
- Images or PDFs described in context (note visual content in the transcript)
- Spreadsheets used as **record sources** for this run (distinct from schema-setup profiling when schema already exists)

Normalize into a single transcript string. Preserve speaker labels, timestamps, and message order when present. Note the source (chat, connectors, file names) for the run summary.

Determine **anchor_date**: prefer an explicit transcript date; otherwise use today's date (ISO 8601 `YYYY-MM-DD`). Relative dates in fields resolve against this anchor.

If no context is provided and the user is not querying an existing bundle or running setup-only, ask for a chat (and optional grounding sources) before proceeding.

### Step 2: Locate or create the inwrk bundle

Default output directory: `inwrk/` in the current workspace. Honor a user-specified path if given. If `inwrk/` is missing but a legacy `okf/` bundle exists, rename `okf/` → `inwrk/` (e.g. `mv okf inwrk`) before loading, then continue with `inwrk/`.

**Schema gate:**

1. If `inwrk/schema.md` exists with `status: confirmed` → load it as authoritative (`object_types`, field defs, relationships, rules). Prefer it over `index.md` frontmatter when they disagree; then sync `index.md` if stale.
2. If `schema.md` is missing:
   - Offer setup per [schema-setup.md](references/schema-setup.md) (especially if tabular sources are present)
   - If the user declines setup and wants to extract now → use default types from [objects.md](references/objects.md) (or a user-requested subset)
3. Never silently invent and confirm a full custom schema without user review

**If the bundle exists:**

- Load `lessons.md` body (YAML lesson list)
- Load `records/index.md` into `prior_record_summaries` (do not load every record file yet)
- Load workspace schema as above (or `index.md` `object_types` / `object_schemas` when no schema)
- Load full prior record files only when a draft clearly matches one (see [okf-output.md](references/okf-output.md))

**If the bundle does not exist:**

- Create the directory structure per [okf-output.md](references/okf-output.md)
- Initialize empty lessons
- Set `prior_record_summaries` to empty
- Run or offer schema setup; otherwise use default object types from [objects.md](references/objects.md) unless the user scoped otherwise

Confirm the object type scope with the user only if ambiguous and no confirmed schema exists. With a confirmed schema, use that scope.

### Step 3: Extract draft records

Follow [stages.md — Stage 1: Extract](references/stages.md#stage-1-extract).

**Inputs:** transcript, lessons, object_types, object_schemas (from workspace schema when present), relationships, rules, prior_record_summaries, anchor_date

**Output:** JSON array of draft records with `object`, `title`, `description`, `fields`, `citations`, `confidence` (`high` | `medium` | `low`), `updates_record_id`

**Rules:**

- One draft = one record instance
- Apply lessons when patterns match
- Align fields to the workspace schema (or fallback defaults)
- Do not hallucinate fields; do not force misfits into wrong fields — note them for the run summary
- Apply identity / dedup rules when considering prior matches
- Resolve relative dates against `anchor_date`
- Never invent `record_id` values
- Prefer precision over recall

### Step 4: Review drafts

Follow [stages.md — Stage 2: Review](references/stages.md#stage-2-review).

**Inputs:** transcript, drafts, workspace schema (relationships + rules when present), optional user corrections

**Output:** Revised drafts array (same schema). Edit drafts in place — merge, split, drop, edit, or relink. Do not emit a separate feedback-action vocabulary.

**Rules:**

- Use transcript as source of truth
- Be conservative — prefer fewer, high-confidence drafts
- Apply user corrections as authoritative
- Use schema dedup / identity keys when merging or setting `updates_record_id`
- Drop anything below `low` confidence

### Step 5: Write inwrk bundle and update lessons

Follow [stages.md — Stage 3: Write and lessons](references/stages.md#stage-3-write-and-lessons) and [okf-output.md](references/okf-output.md).

**Write or update:**

1. `inwrk/index.md` — root index with `okf_version: "0.1"` and object config (synced from schema when present)
2. `inwrk/lessons.md` — updated lesson YAML when corrections warrant it
3. `inwrk/records/<slug>.md` — one file per record (create or update in place; handle slug collisions)
4. `inwrk/records/index.md` — regenerate directory listing
5. `inwrk/log.md` — append Creation/Update entries for this run
6. `inwrk/runs/<date>-<slug>.md` — short run summary including schema misfits/proposals when relevant; retain last 20 runs

**Do not** auto-bump `schema.md` in this step. Propose schema changes in the run file and chat summary; apply only after user approval (then write `schema.md`, snapshot `schema/vN.md`, sync `index.md`, log **Schema**).

**Lessons:** Update primarily from user corrections. Add a lesson from a self-review catch only when the same pattern recurs. Skip if there is nothing to learn. Max 30 lessons.

### Step 6: Chat summary

Report:

- Drafts extracted → kept after review
- Records created vs updated (list titles with object types)
- Lessons added/updated (if any)
- Output path (`inwrk/` or custom)
- Low-confidence records flagged for user review
- Schema misfits (if any)
- Schema proposals awaiting approval (if any)

---

## Object type scoping

**With confirmed `inwrk/schema.md`:** use that schema's objects and fields. It is authoritative.

**Without a workspace schema (fallback):** default types `Task`, `Thread`, `Attendance`, `Appointment`, `Lead`, `Expense`, `Decision`, `Blocker`, `Question` from [objects.md](references/objects.md).

The user may request a subset of defaults:

```
continuity this chat — only Task, Decision, and Blocker
```

Prefer discovering a workspace schema via setup rather than hand-authoring only `object_schemas` in `index.md`. Legacy custom types in `index.md` still work; migrate them into `schema.md` when convenient. See [objects.md](references/objects.md) and [schema-setup.md](references/schema-setup.md).

---

## Handling updates

When `inwrk/` already exists:

1. `records/index.md` becomes prior summaries; load full files only for likely matches
2. Match identity using schema dedup rules when present (not title alone)
3. Drafts with `updates_record_id` update the matching record file in place
4. New drafts create new record files with fresh UUIDs
5. `log.md` gets Update entries for modified records, Creation entries for new ones
6. Lessons accumulate across runs (max 30), driven mainly by user corrections
7. Schema misfits are recorded in the run file; recurring patterns become proposals — bump schema only on user approval

---

## Quality principles

1. **Precision over recall** — skip uncertain drafts
2. **No hallucination** — only extract fields supported by the transcript
3. **Schema fidelity** — do not force facts into the wrong object or field; flag misfits
4. **Date anchoring** — resolve relative dates to ISO 8601 against `anchor_date`
5. **Categorical confidence** — use `high` / `medium` / `low`, not numeric scores
6. **Traceability** — preserve source excerpts in record citations and run summaries
7. **Standalone** — do not import, shell out to, or reference external pipeline code
8. **Summary-first loading** — do not load every record on every run
9. **No silent schema bumps** — evolve `schema.md` only with user approval

---

## Example session

**User:** "continuity — extract records from this chat: [paste]"

**Agent:**

1. Gathers the chat (plus any connector/grounding context) as transcript; sets `anchor_date`
2. Loads `inwrk/` (or creates it); loads confirmed schema or offers setup / uses defaults
3. Extracts drafts against the schema
4. Reviews — merges duplicates via identity rules, drops noise
5. Writes the inwrk bundle; notes any schema misfits in the run file
6. Reports:

```
continuity run complete
- 6 drafts → 4 records (3 created, 1 updated)
- Output: inwrk/
- Review: "Fix login bug" (confidence low) — please verify
- Schema misfits: 1 (warranty claim — no Claim object)
```

---

## Partial runs

If the user requests only a specific stage (e.g., "just extract drafts"), run that stage and return JSON output without writing the full bundle. Schema-setup-only ends after writing `schema.md`. Default to the full 6-step workflow for extract unless explicitly told otherwise.

## Errors and edge cases

- **Empty transcript (extract mode):** Ask for a chat (and optional grounding); do not proceed
- **No drafts found:** Report zero extraction; still write a run summary if useful
- **All drafts low confidence:** Report findings, flag for review, ask whether to write
- **Conflicting prior records:** Prefer null `updates_record_id` over incorrect linkage
- **Query with missing bundle:** Tell the user `inwrk/` was not found; offer to create one from a chat or run schema setup
- **Missing schema on first extract with sheets:** Offer setup before extract; if declined, use defaults
- **Schema proposal:** Present clearly; do not bump until the user approves
