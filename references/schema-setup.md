# Schema Setup

Discover and confirm a workspace schema before (or instead of) relying on Continuity's default nine object types.

The result is a versioned schema under `inwrk/schema.md`. Recurring extract runs then use that schema as authoritative. See [okf-output.md](okf-output.md) for file layout and [schema-vocabulary.md](schema-vocabulary.md) for naming hints.

---

## When to run setup

Run setup when any of the following are true:

- The user says `continuity setup`, `discover schema`, `set up my workspace schema`, or equivalent
- `inwrk/schema.md` is missing and the user attached or offered CSVs, Google Sheets, or similar tabular sources
- The user is starting Continuity in a workspace with no confirmed schema and wants schema discovery before extraction

**Do not** silently invent a full custom schema and write it as confirmed without user review.

If the user declines setup and wants to extract immediately, fall back to the nine default types in [objects.md](objects.md).

**Bundled demo data:** CRM CSVs for Path A live at `sample/sources/crm/` in this repo (see [sample/README.md](../sample/README.md)). Slack, calendar, and email samples under `sample/sources/` are for extract grounding, not schema discovery.

---

## Goals

Propose and confirm:

1. **Objects** the business works with
2. **Fields** and statuses on each object (property types from the vocabulary)
3. **Relationships** between objects
4. **Rules** — validation and deduplication (and optional policy/permission text)

Ask the user **only** questions the source data cannot resolve.

---

## Path A — tabular sources present

Use when the user connects, uploads, or pastes representative CSVs, Google Sheets (via available MCP/connectors), Excel exports, or similar tables.

### A1. Collect sources

1. Ask for representative files or sheet links (not necessarily every spreadsheet — a few that show how the business tracks work)
2. Note source names for schema frontmatter `sources`
3. If connectors fail or files are missing, fall back to Path B for gaps

### A2. Profile the data

For each table/sheet/tab:

1. Read headers and a sample of rows (not every row if the file is huge — sample enough to see value distributions)
2. Infer rough types: text, number, date, boolean, enum-like low-cardinality columns, IDs, emails
3. Note null rates and repeated values that look like statuses or categories
4. Look for shared keys across tables (same ID column, email, normalized company name) as relationship candidates
5. Treat each sheet/tab as a candidate object unless two tabs clearly describe the same entity

### A3. Propose a draft schema

Map findings into the schema shape in [okf-output.md — Workspace schema](okf-output.md#workspace-schema):

- Prefer vocabulary object and property names from [schema-vocabulary.md](schema-vocabulary.md) when they fit
- Map columns → fields; mark likely identity keys
- Propose status enums from observed values when clear
- Propose relationships from shared keys
- Propose validation (required columns with low null rates that look mandatory) and dedup rules (unique-looking keys)

Present the proposal in chat as a readable outline (objects, fields, relationships, rules). Set draft `status: draft` only if writing a temporary file; prefer presenting first, then writing on confirm.

### A4. Resolve open questions

Ask only what the data cannot answer. Examples:

- Which column is the stable identity for this object?
- Are sheet A and sheet B the same entity or related entities?
- What does this status value mean in the lifecycle?
- Should these two similar columns merge into one field?

Do not ask for fields already obvious from headers and samples.

### A5. Confirm and write

When the user confirms:

1. Write `inwrk/schema.md` with `status: confirmed` and `schema_version: 1` (or next version if evolving)
2. Snapshot to `inwrk/schema/vN.md` (immutable copy of this version)
3. Sync `object_types` and compact `object_schemas` into `inwrk/index.md` (see [okf-output.md](okf-output.md))
4. Append an **Initialization** or **Update** entry to `log.md` noting schema confirm/bump
5. Clear **Open questions** in the schema body (or omit the section)

Then proceed to extract if the user already provided a chat/transcript, or stop after setup if that was the only request.

---

## Path B — no tabular data (interview)

Use when the user has no CSVs/Sheets, or Path A sources are insufficient.

Ask a short interview (about 5–8 questions). Adapt wording; skip questions already answered:

1. What kinds of things do you track day to day? (people, companies, deals, jobs, tickets, …)
2. For each important kind, what statuses or stages does it move through?
3. Which people, companies, or accounts show up repeatedly?
4. Do you track money, dates, or deadlines? Which ones matter?
5. How do you know two rows or mentions are the same thing? (email, ID, name, …)
6. What does "done" or "closed" mean for your main work items?
7. What relationships matter? (e.g. Deal belongs to Company, Task belongs to Project)
8. Anything domain-specific Continuity should not flatten into a generic Task?

Then:

1. Propose a schema using vocabulary hints + their answers
2. Let the user edit the proposal
3. Confirm and write the same way as Path A5

---

## Mixed path

If the user provides some sheets and still has gaps, run Path A on the files, then use a short Path B interview only for unresolved objects/fields/relationships.

---

## Schema evolution (after confirm)

During regular Continuity runs (see [SKILL.md](../SKILL.md) and [stages.md](stages.md)):

1. Extract and update records against the confirmed schema
2. Flag **misfits** — facts that do not fit any object/field — in the run summary; do not force them into wrong fields
3. When the same misfit pattern recurs across runs (or is overwhelming within one rich source), **propose** a schema change (new field, enum value, object, or relationship)
4. On user approval only: bump `schema_version`, update `schema.md`, snapshot `schema/vN.md`, sync `index.md`, note the bump in `log.md`
5. Do **not** silently rewrite all historical records on a bump; new extracts use the new schema going forward

---

## Agent checklist

- [ ] Loaded [schema-vocabulary.md](schema-vocabulary.md) for naming hints
- [ ] Chose Path A, Path B, or mixed
- [ ] Asked only unresolved questions
- [ ] User confirmed before writing `status: confirmed`
- [ ] Wrote `schema.md` + `schema/vN.md` + synced `index.md`
- [ ] Did not persist Actions / Interfaces / Intelligence sections
