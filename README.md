# Continuity

**Extract records from chats — against a discovered workspace schema.**

LLMs are great at summarizing and going through stuff but not good at records. Point Continuity at any chat with your connectors and get it to extract records. Provide grounding with other relevant sources.

Every business already has an implicit schema across spreadsheets, documents, and conversations. Continuity can discover that schema (from CSVs, Sheets, or a short interview), save a versioned schema under `inwrk/`, then extract and update records against it — flagging misfits and proposing schema changes when patterns recur.

```
continuity this chat

  EXTRACT   6 drafts (Task · Decision · Blocker · Appointment)
  REVIEW    4 kept (1 merge, 1 drop)
  RECORDS   Task×2 · Decision×1 · Blocker×1  → inwrk/records/
  LESSONS   1 rule updated (from your correction)
  ✅ Records written
```

No keys, no signup, no install beyond the skill itself — works with any [Agent Skills](https://agentskills.io/specification)-compatible agent. Output is a portable records bundle on disk (`inwrk/` by default).

## Install (60 seconds)

**Easiest — let your agent do it.** Paste this into Claude Code, Codex, Cursor, or any agent with shell access:

> Install the continuity skill: clone https://github.com/inwrkai/continuity
> into a skills folder, read its SKILL.md, and confirm the skill is ready by
> summarizing its 6-step workflow in one sentence.

**Claude Code** — one command:

```bash
git clone https://github.com/inwrkai/continuity ~/.claude/skills/continuity
```

Then in any chat: `continuity this chat` — or just ask *"extract records from this conversation."*

**Codex** — clone it, then register it in your `AGENTS.md` so Codex knows when to use it:

```bash
git clone https://github.com/inwrkai/continuity ~/.codex/skills/continuity
echo '- For extracting records from chats ("continuity"), discovering a workspace schema, or answering questions from an existing inwrk/ bundle, read and follow ~/.codex/skills/continuity/SKILL.md' >> ~/.codex/AGENTS.md
```

Then ask Codex: *"extract records from this chat."*

**Cursor / Windsurf / any other agent** — clone into the agent's skills folder (or anywhere and point the agent at `SKILL.md`):

```bash
# Cursor
git clone https://github.com/inwrkai/continuity ~/.cursor/skills/continuity

# Generic / Windsurf / others — clone anywhere, then reference SKILL.md
# from your rules or AGENTS.md, or paste its path into the chat.
git clone https://github.com/inwrkai/continuity ~/skills/continuity
```

<details>
<summary><b>Alternative: npx skills</b></summary>

```bash
npx skills add inwrkai/continuity -g
```

Works across agents, but does not auto-update before each Continuity run (use `npx skills update continuity` manually). Prefer `git clone` when you want Step 0’s pre-run update check.

</details>

The `SKILL.md` contains complete instructions: the agent runs the pipeline and writes the records bundle.

**Updates:** Installed clones check GitHub for updates automatically before each run (skipped when you are developing this repo). Manual update: `git -C <install-path> pull`

<details>
<summary><b>For AI agents installing this</b> (deterministic steps)</summary>

1. `git clone https://github.com/inwrkai/continuity <skills-dir>/continuity`
   — for Claude Code, `<skills-dir>` is `~/.claude/skills`; for Cursor, `~/.cursor/skills`; for Codex, `~/.codex/skills`.
2. Verify: confirm `<skills-dir>/continuity/SKILL.md` exists and frontmatter `name` is `continuity`.
3. To run: read `<skills-dir>/continuity/SKILL.md` and follow it exactly (including Step 0 update check). Load files under `references/` on demand for each pipeline stage.
4. Default output directory is `inwrk/` in the current workspace (user may override).
</details>

## What it does

| Stage | Output |
|---|---|
| Setup (optional) | Versioned workspace schema in `inwrk/schema.md` from CSVs/Sheets or an interview |
| Extract | Draft records with fields, citations, confidence (`high` / `medium` / `low`), optional prior-record link; schema misfits noted |
| Review | Drafts revised in place (merge / split / drop / edit); identity/dedup via schema rules |
| Write | Finalized records → markdown under `inwrk/`; conditional lessons (max 30); short run summary; schema proposals when patterns recur |

**With a confirmed schema:** extract against that workspace's objects, fields, relationships, and rules.

**Without a schema (fallback):** default object types **Task**, **Thread**, **Attendance**, **Appointment**, **Lead**, **Expense**, **Decision**, **Blocker**, **Question**. Scope to a subset or run setup to discover a better fit.

Relative dates ("Thursday", "tomorrow at 10") are resolved to ISO 8601 against the transcript or run date.

## Try with sample data

No connectors needed. Use the bundled **Northwind Analytics** scenario under [`sample/`](sample/README.md):

1. **Setup** — discover schema from `sample/sources/crm/*.csv`
2. **Extract** — Slack thread + calendar + email as grounding → `./inwrk/`
3. **Query** — ask questions about the bundle you just built

See [sample/README.md](sample/README.md) for copy-paste prompts. Sample data is **inputs only** — `inwrk/` is created when you run the skill.

## Example prompts

```
continuity setup
# attach CSVs / Sheets, or answer a short interview

continuity this chat

Extract records from this Slack thread (and ground with the attached doc). Output to inwrk/.

Pull this meeting chat via connectors and extract records — only Task, Decision, and Blocker.

What open tasks and blockers are in inwrk/?

Visualize my knowledge base.

Apply canvas changes.

Discover schema from these spreadsheets, then extract records from this chat.
```

## Re-running on an existing bundle

If `inwrk/` already exists, the skill loads `schema.md` (when present), `lessons.md`, and `records/index.md` summaries first, then full records only for likely updates. New runs update matching records (by `record_id` and identity rules) or add new ones. Recurring misfits become schema proposals — bumps require your approval.

## Querying a bundle

Ask questions without re-extracting — e.g. *"what appointments are upcoming?"* or *"what changed since Monday?"*. The agent reads the index, loads matching records, and cites them. See `references/query.md`.

## Visualizing a bundle

When an `inwrk/` bundle exists, Continuity suggests opening an interactive live canvas after extract and query runs. Ask **visualize** (or **show my knowledge base**) to open an explorer dashboard with record stats, a filterable table, and a relationship graph.

Edit records in the canvas — create, update, or delete — then click **Apply** to write changes to `inwrk/` (agent-mediated). Works on Cursor and any agent with live canvas support. See `references/canvas.md` and `references/canvas-update.md`.

## Repository layout

```
continuity/
├── SKILL.md              # Skill manifest and workflow (read first)
├── AGENTS.md             # Navigation index for agents
├── README.md             # This file
├── LICENSE               # MIT
├── sample/               # Demo inputs (CRM, Slack, calendar, email) — see sample/README.md
│   └── sources/
└── references/
    ├── stages.md         # Extract / review / write stage instructions
    ├── objects.md        # Fallback default object types and schemas
    ├── schema-setup.md   # Discover and confirm workspace schema
    ├── schema-vocabulary.md  # Naming hints for objects / properties / rules
    ├── okf-output.md     # Bundle layout and write rules
    ├── query.md          # Answer questions from an existing bundle
    ├── canvas.md         # Visualize the knowledge base in a live canvas
    └── canvas-update.md  # Apply batched CRUD patches from the canvas
```

The on-disk format is self-contained markdown with YAML frontmatter (compatible with [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) where useful). Workspace schemas live at `inwrk/schema.md` with version snapshots under `inwrk/schema/`.

MIT licensed. See [LICENSE](LICENSE).
