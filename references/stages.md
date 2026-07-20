# Pipeline Stage Instructions

This file contains the three processing stages used by the Continuity skill. The agent performs each stage directly — there is no external API call. Apply the instructions below at each step.

Object types and field schemas come from a confirmed workspace schema (`inwrk/schema.md`) when present, otherwise from [objects.md](objects.md) (fallback defaults). Schema discovery is covered in [schema-setup.md](schema-setup.md). Output format rules come from [okf-output.md](okf-output.md). Querying an existing bundle is covered in [query.md](query.md).

---

## Stage 1: Extract

You are an AI system that extracts structured draft records from conversation context.

A draft record is a candidate knowledge record representing one real-world item (task, appointment, deal, etc. — whatever the workspace schema defines).

### Input

You will receive:

- **transcript** — the gathered context (pasted text, file contents, MCP results, attachments described in chat)
- **lessons** — YAML rules from the existing bundle's `lessons.md` (may be empty on first run)
- **object_types** — allowed object types for this run (from workspace schema or fallback defaults / user request)
- **object_schemas** — field definitions per object type (from workspace schema or [objects.md](objects.md))
- **relationships** — optional; from workspace schema (how objects link)
- **rules** — optional; validation and deduplication from workspace schema
- **prior_record_summaries** — short listings from `records/index.md` (title, description, object, path). Load full prior records only when a draft clearly matches one (see [okf-output.md](okf-output.md))
- **anchor_date** — ISO 8601 date for the transcript or today's run date, used to resolve relative dates

### Task

Produce a JSON array of draft records. Each draft must represent exactly ONE record instance.

Also collect **schema_misfits**: transcript facts that look record-worthy but do not fit any configured object/field. Do not invent new object types mid-extract; list misfits for the run summary instead.

### Output format

```json
{
  "object": "Task",
  "title": "Prepare demo",
  "description": "Prepare demo for Thursday client meeting",
  "fields": {
    "name": "Prepare demo",
    "status": "New",
    "due_date": "2026-07-16"
  },
  "citations": [
    "Let's prepare the demo for Thursday"
  ],
  "confidence": "high",
  "updates_record_id": null
}
```

Required fields per draft:

| Field | Rule |
|-------|------|
| `object` | One of the configured `object_types` |
| `title` | Short display name (3–7 words) |
| `description` | Clear human-readable summary (5–12 words preferred) |
| `fields` | Schema-aligned fields; omit empty values |
| `citations` | Exact transcript excerpts that support the draft |
| `confidence` | `high`, `medium`, or `low` |
| `updates_record_id` | Exact `record_id` from a prior record when updating; otherwise `null` |

Misfit note shape (for Stage 3 / run file — not a draft record):

```json
{
  "summary": "Warranty claim 8821 mentioned",
  "citation": "opened warranty claim 8821",
  "reason": "No Claim object in workspace schema"
}
```

### How to apply lessons

- Apply lessons only when the pattern clearly matches the transcript
- Treat lessons as strong guidance, not absolute rules
- Do not force a lesson if context does not fit
- If multiple lessons apply, prefer the most specific one
- Do not contradict the core extraction rules

Lessons primarily influence:

- whether to extract a record (`extraction`)
- how to group related mentions (`grouping`)
- how to populate fields (`field`)

### Date anchoring

Relative dates must be resolved against `anchor_date` (transcript date if known, otherwise the run date):

1. Convert phrases like "Thursday", "tomorrow at 10", "next week" into ISO 8601 (`YYYY-MM-DD` or `YYYY-MM-DDTHH:MM` when time is known)
2. Keep the original phrasing in a citation (e.g., `"Let's prepare the demo for Thursday"`)
3. If no anchor date can be derived, store the raw phrase in the field and also set `as_of_date` to the run date so the record stays interpretable later

Examples (anchor_date = `2026-07-11`, a Saturday):

| Said in transcript | Stored field value |
|--------------------|--------------------|
| "Thursday" | `2026-07-16` |
| "tomorrow at 10" | `2026-07-12T10:00` |
| "next Monday" | `2026-07-13` |

### Confidence

Use categorical confidence only:

- **high** — clear actionable statement, strong evidence in transcript
- **medium** — reasonable inference with some ambiguity
- **low** — weak or partially supported; keep only if still useful, and flag for user review

Drop drafts that do not meet at least `low` confidence. Prefer precision over recall.

### Linking to prior records

- Prefer `updates_record_id` when the draft clearly refers to the same real-world item as a prior record
- When workspace **rules** define identity / deduplication keys, match on those fields first (e.g. email, external id, name+company)
- Otherwise match on title/description/fields, not just object type
- Prefer `null` when the conversation is about a different item, even of the same type
- Never invent a `record_id`; only copy values from loaded prior records
- When unsure, use `null`
- When a candidate match appears in `prior_record_summaries`, load that full record file before setting `updates_record_id`

### Field rules

- Only include fields clearly present or directly inferable
- Prefer fields defined in `object_schemas` / workspace schema
- Respect validation rules when obvious (enum membership, required fields) — omit or flag rather than invent
- Do NOT hallucinate missing values
- Do NOT include empty fields
- Do NOT force misfit information into the wrong field — add a schema misfit note instead
- Keep field values concise and normalized
- Names should match how they appear in the transcript
- Resolve relative dates per the date-anchoring rules above

### Extraction rules

- Only extract records that are reasonably actionable or meaningful
- One draft = one real-world item (do not merge different tasks into one record)
- Avoid duplicating drafts for the same item
- Use the smallest useful unit of meaning
- Do not invent information not present in the transcript
- Prefer precision over recall
- If uncertain, do not create a draft
- Stay within configured `object_types` unless the user explicitly expands scope mid-run

### Examples

Message: "Let's prepare the demo for Thursday" (anchor_date `2026-07-11`)

```json
{
  "object": "Task",
  "title": "Prepare demo",
  "description": "Prepare demo for Thursday client meeting",
  "fields": {
    "name": "Prepare demo",
    "status": "New",
    "due_date": "2026-07-16"
  },
  "citations": ["Let's prepare the demo for Thursday"],
  "confidence": "high",
  "updates_record_id": null
}
```

Message: "Schedule a call with Priya tomorrow at 10"

```json
{
  "object": "Appointment",
  "title": "Call with Priya",
  "description": "Schedule call with Priya tomorrow at 10",
  "fields": {
    "title": "Call with Priya",
    "participants": ["Priya"],
    "date_time": "2026-07-12T10:00",
    "status": "Scheduled"
  },
  "citations": ["Schedule a call with Priya tomorrow at 10"],
  "confidence": "high",
  "updates_record_id": null
}
```

---

## Stage 2: Review

You are an AI system that reviews draft records against the transcript and revises them in place.

Your goal is to catch extraction mistakes before writing the bundle.

### Input

- **transcript**
- **drafts** — JSON array from Stage 1
- **relationships** / **rules** — optional; from workspace schema
- **schema_misfits** — optional; from Stage 1
- **user_corrections** (optional) — corrections the user provided during the session

### Goal

Produce a revised JSON array of drafts. Edit the draft set directly — do not emit a separate feedback-action vocabulary. Preserve or refine the misfit list for Stage 3.

### What to fix

1. **Merge** — two drafts refer to the same real-world item → combine into one (merge fields and citations; keep higher confidence). Prefer schema identity keys when deciding sameness.
2. **Split** — one draft conflates different intents → split into separate drafts
3. **Drop** — conversational noise, non-actionable mentions, or unsupported hallucination → remove
4. **Edit** — title, description, or fields miss key transcript context → revise; fix enum/status values that violate schema validation when the transcript supports a valid value
5. **Relink** — `updates_record_id` is wrong or missing when clearly an update → correct or set to `null` using identity/dedup rules

### User corrections

When the user provides corrections (e.g., "that isn't a task", "merge those two", "the due date is Friday"):

1. Apply them as authoritative revisions to the draft set
2. Treat user corrections as primary evidence for lessons (Stage 3)

### Rules

- Use the transcript as the source of truth
- Be conservative — prefer fewer, high-confidence drafts
- Do NOT invent new facts beyond the transcript
- Do NOT invent `record_id` values
- Do NOT invent new object types to absorb misfits during review
- Preserve citations when merging; drop citations that no longer apply when splitting/editing
- Re-check date anchoring after edits
- After review, every remaining draft must still meet at least `low` confidence

### Output

Return the full revised drafts array (same schema as Stage 1), plus the misfit list for writing. This is the final set used for writing.

---

## Stage 3: Write and lessons

Persist the reviewed drafts as an inwrk bundle, update lessons when warranted, record schema misfits/proposals, and report a chat summary.

Follow [okf-output.md](okf-output.md) for file layout and write rules.

### Record finalization

For each reviewed draft:

1. If `updates_record_id` is set, reuse that `record_id` and update the existing file in place
2. Otherwise generate a new UUID for `record_id`
3. Write title, description, fields, confidence, citations, and timestamp
4. Derive a filename slug from `title`; on collision with a different `record_id`, append a short UUID suffix (see [okf-output.md](okf-output.md))
5. When relationships are known, optionally add `# Related` links to other records in the bundle

### Schema misfits and proposals

1. Write `# Schema misfits` in the run file when misfits exist (with citations)
2. If the same misfit pattern has appeared across prior runs (check recent `runs/` summaries) **or** is overwhelming within this source, add `# Schema proposals` with a concrete change (new object, field, enum value, or relationship)
3. Present proposals in the chat summary; **do not** bump `schema.md` until the user approves
4. On approval: follow [schema-setup.md](schema-setup.md) evolution steps and [okf-output.md](okf-output.md) snapshot/sync rules
5. Do not silently rewrite all historical records when the schema bumps

### Lessons update (conditional)

A lesson is a short rule that improves future extraction.

**When to update lessons:**

- **Primary:** user corrections during the session
- **Secondary:** self-review catches only when the same pattern recurs (do not add a lesson for every one-off self-fix)

**When to skip:**

- No user corrections and no recurring self-review pattern
- Feedback would not generalize beyond this transcript

**Input when updating:**

- Existing lessons from `lessons.md`
- User corrections and/or recurring review patterns
- The drafts involved

**Output format** — return the FULL updated lesson set in YAML (max 30):

```yaml
- object: extraction | grouping | field
  when: <pattern to detect>
  do: <action to take>
  example: <short example>  # optional
```

**Rules:**

- Preserve existing high-quality lessons
- Prefer fewer, stronger lessons
- Keep each field concise (≤15 words)
- Avoid proper nouns unless necessary
- Do NOT regenerate lessons from scratch
- Ground lessons in correction evidence, not speculation

**Interpreting corrections:**

- User said two items are the same → grouping lesson
- User said something is not actionable → extraction lesson
- User corrected a field value or date → field lesson

### Chat summary

After writing the bundle, report:

- Drafts extracted (count before review)
- Drafts kept after review (count)
- Records created vs updated (list titles with object types)
- Lessons added/updated (count), if any
- Output path (`inwrk/` or custom)
- Low-confidence records flagged for user review
- Schema misfits and proposals awaiting approval, if any
