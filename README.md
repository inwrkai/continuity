# inwrk Ops Brain

**Turn work context into portable operational records for AI agents.**

The open operational-context layer for [inwrk](https://github.com/inwrkai): extract, maintain, and query tasks, decisions, blockers, and other operational truth from conversations, documents, and connected sources.

```
ops-brain this standup transcript

  EXTRACT   6 drafts (Task · Decision · Blocker · Appointment)
  REVIEW    4 kept (1 merge, 1 drop)
  RECORDS   Task×2 · Decision×1 · Blocker×1  → okf/records/
  LESSONS   1 rule updated (from your correction)
  ✅ Operational bundle written
```

No keys, no signup, no install beyond the skill itself — works with any [Agent Skills](https://agentskills.io/specification)-compatible agent. Output is a portable OKF knowledge bundle on disk (`okf/` by default).

## Install (60 seconds)

**Easiest — let your agent do it.** Paste this into Claude Code, Codex, Cursor, or any agent with shell access:

> Install the ops-brain skill: clone https://github.com/inwrkai/ops-brain
> into a skills folder, read its SKILL.md, and confirm the skill is ready by
> summarizing its 6-step workflow in one sentence.

**Claude Code** — one command:

```bash
git clone https://github.com/inwrkai/ops-brain ~/.claude/skills/ops-brain
```

Then in any chat: `ops-brain this transcript` — or just ask *"extract tasks and blockers from these meeting notes."*

**Codex** — clone it, then register it in your `AGENTS.md` so Codex knows when to use it:

```bash
git clone https://github.com/inwrkai/ops-brain ~/.codex/skills/ops-brain
echo '- For turning work context into operational records ("ops-brain", "operational context"), or answering questions from an existing okf/ bundle, read and follow ~/.codex/skills/ops-brain/SKILL.md' >> ~/.codex/AGENTS.md
```

Then ask Codex: *"turn this transcript into operational records."*

**Cursor / Windsurf / any other agent** — clone into the agent's skills folder (or anywhere and point the agent at `SKILL.md`):

```bash
# Cursor
git clone https://github.com/inwrkai/ops-brain ~/.cursor/skills/ops-brain

# Generic / Windsurf / others — clone anywhere, then reference SKILL.md
# from your rules or AGENTS.md, or paste its path into the chat.
git clone https://github.com/inwrkai/ops-brain ~/skills/ops-brain
```

The `SKILL.md` contains complete instructions: the agent runs the pipeline and writes the operational bundle.

**Update later:** `git -C <install-path> pull`

<details>
<summary><b>For AI agents installing this</b> (deterministic steps)</summary>

1. `git clone https://github.com/inwrkai/ops-brain <skills-dir>/ops-brain`
   — for Claude Code, `<skills-dir>` is `~/.claude/skills`; for Cursor, `~/.cursor/skills`; for Codex, `~/.codex/skills`.
2. Verify: confirm `<skills-dir>/ops-brain/SKILL.md` exists and frontmatter `name` is `ops-brain`.
3. To run: read `<skills-dir>/ops-brain/SKILL.md` and follow it exactly. Load files under `references/` on demand for each pipeline stage.
4. Default output directory is `okf/` in the current workspace (user may override).
</details>

## What it does

| Stage | Output |
|---|---|
| Extract | Draft records with fields, citations, confidence (`high` / `medium` / `low`), optional prior-record link |
| Review | Drafts revised in place (merge / split / drop / edit); user corrections applied |
| Write | Finalized records → OKF markdown; conditional lessons (max 30); short run summary |

Default object types: **Task**, **Thread**, **Attendance**, **Appointment**, **Lead**, **Expense**, **Decision**, **Blocker**, **Question**. Scope to a subset or define custom types per bundle.

Relative dates ("Thursday", "tomorrow at 10") are resolved to ISO 8601 against the transcript or run date.

## Example prompts

```
ops-brain this standup transcript

Convert the attached export into operational records. Output to okf/.

Organize this context into records — only Task, Decision, and Blocker.

What open tasks and blockers are in okf/?
```

## Re-running on an existing bundle

If `okf/` already exists, the skill loads `lessons.md` and `records/index.md` summaries first, then full records only for likely updates. New runs update matching records (by `record_id`) or add new ones.

## Querying a bundle

Ask questions without re-extracting — e.g. *"what appointments are upcoming?"* or *"what changed since Monday?"*. The agent reads the index, loads matching records, and cites them. See `references/query.md`.

## Repository layout

```
ops-brain/
├── SKILL.md              # Skill manifest and 6-step workflow (read first)
├── AGENTS.md             # Navigation index for agents
├── README.md             # This file
├── LICENSE               # MIT
└── references/
    ├── stages.md         # Extract / review / write stage instructions
    ├── objects.md        # Default object types and schemas
    ├── okf-output.md     # Bundle layout and write rules
    └── query.md          # Answer questions from an existing bundle
```

The on-disk format is self-contained markdown with YAML frontmatter (compatible with [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) where useful).

MIT licensed. See [LICENSE](LICENSE).
