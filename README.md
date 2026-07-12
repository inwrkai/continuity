# okf-brain

**Point your AI agent at conversation context → get organized OKF knowledge records.**

Transcripts, attachments, MCP data, pasted notes — anything becomes structured records via a **signals → bundles → feedback → records → lessons** pipeline. Output is a portable [OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) bundle on disk (`okf/` by default).

```
okf-brain this standup transcript

  SIGNALS   12 extracted (NEW / REQUESTED / UPDATED)
  BUNDLES   5 record candidates
  RECORDS   Task×3 · Appointment×1 · Lead×1  → okf/records/
  LESSONS   2 rules updated
  ✅ OKF v0.1 bundle written
```

No keys, no signup, no install beyond the skill itself — works with any [Agent Skills](https://agentskills.io/specification)-compatible agent.

## Install (60 seconds)

**Easiest — let your agent do it.** Paste this into Claude Code, Codex, Cursor, or any agent with shell access:

> Install the okf-brain skill: clone https://github.com/anubhavmisra/okf-brain
> into a skills folder, read its SKILL.md, and confirm the skill is ready by
> summarizing its 8-step workflow in one sentence.

**Claude Code** — one command:

```bash
git clone https://github.com/anubhavmisra/okf-brain ~/.claude/skills/okf-brain
```

Then in any chat: `okf-brain this transcript` — or just ask *"convert this meeting notes into OKF records."*

**Codex** — clone it, then register it in your `AGENTS.md` so Codex knows when to use it:

```bash
git clone https://github.com/anubhavmisra/okf-brain ~/.codex/skills/okf-brain
echo '- For converting transcripts/context into OKF records ("okf-brain", "convert to records"), read and follow ~/.codex/skills/okf-brain/SKILL.md' >> ~/.codex/AGENTS.md
```

Then ask Codex: *"turn this transcript into OKF records."*

**Cursor / Windsurf / any other agent** — clone into the agent's skills folder (or anywhere and point the agent at `SKILL.md`):

```bash
# Cursor
git clone https://github.com/anubhavmisra/okf-brain ~/.cursor/skills/okf-brain

# Generic / Windsurf / others — clone anywhere, then reference SKILL.md
# from your rules or AGENTS.md, or paste its path into the chat.
git clone https://github.com/anubhavmisra/okf-brain ~/skills/okf-brain
```

The `SKILL.md` contains complete instructions: the agent runs the pipeline and writes the OKF bundle.

**Update later:** `git -C <install-path> pull`

<details>
<summary><b>For AI agents installing this</b> (deterministic steps)</summary>

1. `git clone https://github.com/anubhavmisra/okf-brain <skills-dir>/okf-brain`
   — for Claude Code, `<skills-dir>` is `~/.claude/skills`; for Cursor, `~/.cursor/skills`; for Codex, `~/.codex/skills`.
2. Verify: confirm `<skills-dir>/okf-brain/SKILL.md` exists and frontmatter `name` is `okf-brain`.
3. To run: read `<skills-dir>/okf-brain/SKILL.md` and follow it exactly. Load files under `references/` on demand for each pipeline stage.
4. Default output directory is `okf/` in the current workspace (user may override).
</details>

## What it does

| Stage | Output |
|---|---|
| Signals | Atomic extractions (`NEW` / `UPDATED` / `REQUESTED` / …) with fields |
| Bundles | One bundle = one record instance, optional link to prior records |
| Feedback | Self-review corrections (merge / split / ignore / edit) |
| Records | Finalized `record_id`, title, description, fields → OKF markdown |
| Lessons | Persistent YAML rules (max 30) that improve the next run |

Default object types: **Task**, **Thread**, **Attendance**, **Appointment**, **Lead**, **Expense**. Scope to a subset or define custom types per bundle.

## Example prompts

```
okf-brain this standup transcript

Convert the attached export into OKF records. Output to okf/.

Organize this context into records — only Task and Appointment.
```

## Re-running on an existing bundle

If `okf/` already exists, the skill loads `lessons.md` and `records/*.md` as prior confirmed records. New runs update matching records (by `record_id`) or add new ones.

## Repository layout

```
okf-brain/
├── SKILL.md              # Skill manifest and 8-step workflow (read first)
├── AGENTS.md             # Navigation index for agents
├── README.md             # This file
├── LICENSE               # MIT
└── references/
    ├── stages.md         # Pipeline stage instructions
    ├── objects.md        # Default object types and schemas
    └── okf-output.md     # OKF bundle layout and write rules
```

MIT licensed. See [LICENSE](LICENSE).
