# Querying an inwrk Bundle

How to answer questions from an existing `inwrk/` knowledge bundle without re-running extraction.

Use this reference when the user asks about records already stored in a bundle — for example open tasks, upcoming appointments, decisions, blockers, or what changed recently.

## When to query (not extract)

Prefer query mode when:

- The user asks a question about existing knowledge ("what tasks are open?", "any blockers?")
- The user references the inwrk bundle, `inwrk/`, or prior records without providing new transcript to convert
- The user wants a summary, filter, or count of records already on disk

Prefer the extract → review → write pipeline when the user provides new context to turn into records.

## Loading strategy (summary-first)

Do **not** load every record file into context by default.

1. Read root `inwrk/index.md` for bundle title, description, and `object_types`
2. When `inwrk/schema.md` exists with `status: confirmed`, use it for object/field names (authoritative over stale `index.md` frontmatter)
3. Read `inwrk/records/index.md` for the listing of titles, descriptions, and types
4. Filter the index to candidates that match the question
5. Load only those matching `records/<slug>.md` files
6. Optionally read `inwrk/log.md` for change history and `inwrk/lessons.md` only if relevant

If `records/index.md` is missing, scan `records/*.md` filenames and frontmatter as a fallback.

## Answering

- Cite records with bundle-relative links and titles, e.g. [Prepare demo](/records/prepare-demo.md)
- Prefer fields from frontmatter and the `# Fields` table; use `# Citations` when the user asks for evidence
- Map legacy numeric `confidence` to `high` / `medium` / `low` per [objects.md](objects.md) when comparing or filtering
- Resolve "upcoming" / "overdue" using ISO 8601 date fields against today's date; if a field is still a relative phrase with `as_of_date`, interpret relative to `as_of_date` and note uncertainty

## Common operations

| User ask | How to answer |
|----------|----------------|
| Open / incomplete tasks | Filter `type: Task` where `status` is `New` or `In Progress` |
| Upcoming appointments | Filter `type: Appointment` with `date_time` ≥ today; sort ascending |
| Open blockers | Filter `type: Blocker` where `status` is `Open` or `Mitigating` |
| Unresolved questions | Filter `type: Question` where `status` is `Open` or `Deferred` |
| Recent decisions | Filter `type: Decision`; sort by `date` or `timestamp` descending |
| Expense totals | Sum `amount` for matching `type: Expense` records; group by `currency` / `category` if asked |
| What changed since DATE | Read `log.md` entries on or after that date; follow Creation/Update links |
| Everything about X | Search `records/index.md` for X in titles/descriptions; load matches |

## Response shape

Keep answers concise:

1. Direct answer (list, total, or short paragraph)
2. Record citations with type and key fields
3. Note gaps (e.g., low-confidence records, unresolved relative dates)

Example:

```
Open tasks (2):
- [Prepare demo](/records/prepare-demo.md) — due 2026-07-16 (Task, high)
- [Fix login token expiry](/records/fix-login-token-expiry.md) — In Progress (Task, medium)

No open blockers in inwrk/.
```

## Do not

- Re-run the full extract pipeline just to answer a read question
- Invent records that are not in the bundle
- Load all run files unless the user asks about a specific extraction run
