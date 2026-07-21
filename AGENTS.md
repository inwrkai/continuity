# Continuity Agent Skill — Navigation Guide

Guide for AI agents extracting records from chats (with optional grounding sources), discovering a workspace schema, and answering questions from existing bundles.
Read [SKILL.md](SKILL.md) first for Step 0, schema setup, the 6-step workflow, and query mode.

## Structure

```
continuity/
  SKILL.md          # Main skill manifest — read this first
  AGENTS.md         # This navigation guide
  references/       # Detailed reference files — load on demand
  sample/           # Demo inputs only (no bundled inwrk/) — see sample/README.md
    sources/
      crm/          # CSV exports for schema setup demos
      slack/        # Chat transcript for extract
      calendar/     # ICS grounding
      email/        # Email thread grounding
```

## Quick Start

```
New to continuity? Read in this order:
  0. sample/README.md                 ← optional hands-on demo (no connectors)
  1. SKILL.md                         ← Step 0, setup, 6-step workflow, query mode
  2. references/schema-setup.md       ← discover schema from CSVs/Sheets or interview
  3. references/schema-vocabulary.md  ← naming hints (Objects, Properties, Rules)
  4. references/stages.md             ← extract → review → write/lessons
  5. references/objects.md            ← fallback default object types
  6. references/okf-output.md         ← inwrk bundle layout (incl. schema.md)
  7. references/query.md              ← answering questions from an existing bundle
  8. references/canvas.md             ← visualize the knowledge base in a live canvas
  9. references/canvas-update.md      ← apply batched CRUD patches from the canvas
```

---

## Reference Index

| File | What it covers |
|------|----------------|
| [`references/stages.md`](references/stages.md) | Three pipeline stages: extract draft records, self-review, write bundle + conditional lessons |
| [`references/objects.md`](references/objects.md) | Fallback default object catalog; custom types; points to workspace schema |
| [`references/schema-setup.md`](references/schema-setup.md) | Setup: discover schema from tabular sources or interview; confirm; evolve |
| [`references/schema-vocabulary.md`](references/schema-vocabulary.md) | Suggestion vocabulary for Objects, Properties, Rules (not Actions/Interfaces/AI) |
| [`references/okf-output.md`](references/okf-output.md) | Bundle layout, `schema.md` / snapshots, frontmatter, summary-first loading, runs |
| [`references/query.md`](references/query.md) | How to answer questions from an existing `inwrk/` bundle without re-extracting |
| [`references/canvas.md`](references/canvas.md) | Suggest and build an interactive knowledge-base canvas (explorer + graph + CRUD queue) |
| [`references/canvas-update.md`](references/canvas-update.md) | Apply batched create/update/delete patches from the canvas to `inwrk/` |

---

## Pipeline Overview

```
optional setup: CSVs/Sheets or interview → confirmed inwrk/schema.md
        ↓
chat (+ connectors / grounding) + lessons + schema + prior summaries + anchor_date
        ↓
   Stage 1: Extract     → draft records (+ schema misfits)
        ↓
   Stage 2: Review      → revise drafts in place (+ identity/dedup)
        ↓
   Stage 3: Write       → inwrk bundle; lessons; misfits/proposals in run file
        ↓
   Chat summary (propose schema bump only with user approval)
```

Query path (no new transcript): `index.md` + `records/index.md` → load matches → answer with citations. See [query.md](references/query.md).

Visualize path (on user request): load bundle → build live canvas per [canvas.md](references/canvas.md). Suggest canvas after query/extract summaries when records exist.

Canvas apply path: validate patch → write CRUD to bundle per [canvas-update.md](references/canvas-update.md) → refresh canvas.

---

## Most Common Agent Mistakes

| Mistake | Correct approach |
|---------|------------------|
| Leaving a legacy `okf/` directory in place | If `inwrk/` is missing and `okf/` exists, rename `okf/` → `inwrk/` before the run |
| Skipping Step 0 on a GitHub-cloned skills install | Fetch/pull when `origin` is `inwrkai/continuity` (or legacy `ops-brain`) and workspace ≠ `SKILL_DIR` |
| Running `git pull` while developing this repo | Skip update check when workspace root is `SKILL_DIR` |
| Imposing only the nine defaults when sheets are available | Offer [schema setup](references/schema-setup.md); discover the workspace schema |
| Silently confirming a custom schema without review | Present a proposal; write `status: confirmed` only after the user confirms |
| Ignoring confirmed `schema.md` in favor of defaults | Confirmed workspace schema is authoritative |
| Hallucinating fields not in transcript | Only extract fields clearly present or directly inferable |
| Forcing misfits into wrong fields | Flag schema misfits; propose changes when patterns recur |
| Auto-bumping schema without approval | Propose in run/chat; bump only on user approval |
| Merging all items of the same object type into one record | One draft = one real-world item |
| Inventing `record_id` for updates | Copy `record_id` only from loaded prior records |
| Matching updates by title only when identity keys exist | Prefer schema dedup / identity rules |
| Storing relative dates ("Thursday") without anchoring | Resolve against `anchor_date` to ISO 8601; keep original phrasing in citations |
| Writing numeric confidence scores | Use `high` / `medium` / `low` only |
| Loading every `records/*.md` on every run | Read `records/index.md` first; load full files only for likely matches |
| Updating lessons from every self-review | Prefer user corrections; add self-review lessons only for recurring patterns |
| Re-running extraction to answer a read question | Use [query.md](references/query.md) |
| Referencing external pipeline code | Skill is standalone — agent performs all stages directly |
| Persisting Actions / Interfaces / Intelligence in schema | Only Objects, Properties, Relationships, Rules |
| Auto-opening a canvas after every run | Suggest always; open only when the user asks — see [canvas.md](references/canvas.md) |
| Writing Cursor `.canvas.tsx` on non-Cursor agents | Detect the host canvas skill; adapt file location and imports |
| Rendering empty graph/table sections in a canvas | Omit sections with no data; never show placeholders |
| Inventing records or edges for the visualization | Only embed data present in the bundle |
| Writing `inwrk/` from canvas runtime without agent apply | Queue ops in canvas; agent applies per [canvas-update.md](references/canvas-update.md) |
| Applying canvas patches without schema validation | Validate field keys and enums; reject unknown fields |

---

## Output Location

Default: `inwrk/` in the current workspace. User may override. See [okf-output.md](references/okf-output.md) for the full bundle structure (including `schema.md` and `schema/vN.md`).
