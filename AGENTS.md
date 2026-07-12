# okf-brain Agent Skill — Navigation Guide

Guide for AI agents converting conversation context into OKF v0.1 knowledge records.
Read [SKILL.md](SKILL.md) first for the 8-step orchestration workflow.

## Structure

```
okf-brain/
  SKILL.md          # Main skill manifest — read this first
  AGENTS.md         # This navigation guide
  references/       # Detailed reference files — load on demand
```

## Quick Start

```
New to okf-brain? Read in this order:
  1. SKILL.md                         ← 8-step workflow and trigger conditions
  2. references/stages.md             ← stage prompts (signals → lessons)
  3. references/objects.md            ← default object types and field schemas
  4. references/okf-output.md         ← OKF bundle layout and write rules
```

---

## Reference Index

| File | What it covers |
|------|----------------|
| [`references/stages.md`](references/stages.md) | Five pipeline stages: signals extraction, bundle grouping, feedback self-review, record finalization, lessons update |
| [`references/objects.md`](references/objects.md) | Default object catalog (Task, Thread, Attendance, Appointment, Lead, Expense), field schemas, custom type guidance |
| [`references/okf-output.md`](references/okf-output.md) | OKF v0.1 bundle layout, frontmatter rules, record files, log/lessons/runs, prior-record loading |

---

## Pipeline Overview

```
transcript + lessons + object config
        ↓
   Stage 1: Signals      → JSON array of atomic extractions
        ↓
   Stage 2: Bundles      → group signals into record candidates
        ↓
   Stage 3: Feedback     → self-review corrections
        ↓
   Stage 4: Records      → finalized structured records
        ↓
   Stage 5: Lessons      → persistent extraction rules (max 30)
        ↓
   Write OKF bundle      → okf/ in workspace
```

---

## Most Common Agent Mistakes

| Mistake | Correct approach |
|---------|------------------|
| Hallucinating fields not in transcript | Only extract fields clearly present or directly inferable |
| Grouping all signals of same object into one bundle | One bundle = one record instance; group by real-world item |
| Inventing `record_id` for updates | Copy `record_id` only from `prior_confirmed_records` |
| Editing signal text to fix errors | Use feedback actions (`merge_bundles`, `ignore_signal`, etc.) |
| Skipping lessons on re-runs | Load existing `lessons.md` before signals; update after feedback |
| Referencing external pipeline code | Skill is standalone — agent performs all stages directly |

---

## Output Location

Default: `okf/` in the current workspace. User may override. See [okf-output.md](references/okf-output.md) for the full bundle structure.
