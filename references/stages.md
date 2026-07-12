# Pipeline Stage Instructions

This file contains the five processing stages used by the okf-brain skill. The agent performs each stage directly — there is no external API call. Apply the instructions below at each step.

Object types and field schemas come from [objects.md](objects.md). Output format rules come from [okf-output.md](okf-output.md).

---

## Stage 1: Signals

You are an AI system that extracts structured signals from conversation context.

A signal is a meaningful piece of information implied or stated in the context that could later be turned into a record.

### Input

You will receive:

- **transcript** — the gathered context (pasted text, file contents, MCP results, attachments described in chat)
- **lessons** — YAML rules from the existing OKF bundle's `lessons.md` (may be empty on first run)
- **object_types** — allowed object types for this run (from bundle config or user request)
- **object_schemas** — field definitions per object type (see [objects.md](objects.md))

### Task

Identify signals present in the transcript.

Each signal has the format: `<VERB> <OBJECT> <SOURCE MESSAGE>`

**Verbs** (select one):

- `NEW`
- `UPDATED`
- `MENTIONED`
- `REMOVED`
- `REQUESTED`
- `CONFIRMED`

**Objects** — select from the configured `object_types` list.

### Goals

1. Apply lessons when relevant
2. Extract structured fields for each signal

### How to apply lessons

- Apply lessons only when the pattern clearly matches the transcript
- Treat lessons as strong guidance, not absolute rules
- Do not force a lesson if context does not fit
- If multiple lessons apply, prefer the most specific one
- Do not contradict the core extraction rules

Lessons primarily influence:

- whether to extract a signal (`extraction`)
- how to populate fields (`field`)

### Output format

Return a JSON array of signals. Each signal must contain:

```json
{
  "signal_id": "UUID",
  "verb": "verb",
  "object": "object",
  "summary": "one sentence describing the signal",
  "fields": {},
  "source_excerpt": "exact excerpt from transcript that triggered the signal",
  "confidence": 0.0
}
```

### Field rules

- Only include fields that are clearly present or directly inferable
- Do NOT hallucinate missing values
- Do NOT include empty fields
- Keep field values concise and normalized
- Names should match how they appear in the transcript

### Examples

Message: "Let's prepare the demo for Thursday"

```json
{
  "verb": "REQUESTED",
  "object": "Task",
  "fields": {
    "name": "Prepare demo",
    "due_date": "Thursday"
  }
}
```

Message: "Schedule a call with Priya tomorrow at 10"

```json
{
  "verb": "REQUESTED",
  "object": "Appointment",
  "fields": {
    "title": "Call with Priya",
    "participants": ["Priya"],
    "date_time": "tomorrow at 10"
  }
}
```

### Rules

- Only extract signals that are reasonably actionable or meaningful
- Avoid duplicating signals
- Use the smallest useful unit of meaning
- If multiple people are involved, capture context in summary and fields
- Do not invent information not present in the transcript
- Signals are immutable events — extract carefully
- Prefer precision over recall
- If uncertain, do not create a signal

---

## Stage 2: Bundles

You are an AI system that groups signals into bundles representing individual records.

A bundle represents ONE record instance, formed by grouping related signals of the SAME object.

### Input

You will receive:

- A JSON array of **signals** (current extraction)
- A JSON array **`prior_confirmed_records`**: records from the existing OKF bundle under `records/` (may be empty)

Each signal has: `verb`, `object`, `summary`, `source_excerpt`, `confidence`

Each prior confirmed record (when present) typically includes:

- `record_id` (stable identifier; use this for linkage)
- `object` or `record_type`
- `title`, `description`, `fields`, `confidence`

### Goals

1. Group signals that refer to the SAME underlying record
2. Ensure all signals within a bundle share the SAME object
3. Produce a clear, human-readable resolution of what the bundle represents
4. Decide if a bundle updates an existing confirmed record from `prior_confirmed_records` when the context clearly refers to the same real-world item

### Important constraint

- You MUST NOT change or infer new objects
- Objects must remain exactly as provided in signals
- A bundle can only contain signals of ONE object

### Output format

Return a JSON object with top-level key `bundles` (array).

Each bundle must contain:

```json
{
  "object": "one of object_types",
  "resolved_text": "short, clear description of the final resolved record",
  "signals": ["signal_id_1", "signal_id_2"],
  "confidence": 0.0,
  "updates_record_id": null
}
```

**`updates_record_id`:**

- Set to the exact `record_id` from `prior_confirmed_records` when this bundle is a follow-up or change to that already approved record
- Use `null` when this bundle represents a new record with no matching prior row, or when unsure
- Never invent a `record_id`; only copy values from `prior_confirmed_records`
- If `prior_confirmed_records` is empty, every bundle must have `updates_record_id` null

### Resolved text guidelines

- Write in plain, human-readable language
- Capture the FINAL meaning after combining all signals
- Be concise (5–12 words preferred)
- Do NOT include meta language (no mention of signals or conversation)
- Do NOT invent details not present in signals

Examples:

- "Prepare demo for Thursday client meeting"
- "Fix login failure due to token expiry"
- "Schedule review meeting with design team"
- "Update onboarding documentation with new screenshots"

### Grouping logic

1. Group by RECORD INSTANCE (not just object) — signals referring to the same real-world item → same bundle
2. Do NOT group only by object — multiple Tasks ≠ one bundle
3. Use verbs to determine strength:
   - `CONFIRMED`, `REQUESTED` → strong signals (anchor bundles)
   - `NEW`, `UPDATED` → supporting signals
   - `MENTIONED` → weak (include only if reinforced)
4. Merge signals when they clearly refer to the same item, one adds detail to another, or they occur in close conversational context
5. Do NOT merge when signals refer to different records, even if similar in nature
6. A single `MENTIONED` signal without reinforcement can be ignored
7. Every bundle must represent exactly ONE record and contain at least one meaningful signal

### Update vs new

- Prefer linking (`updates_record_id`) when the context clearly refers to the same item as a prior row (match on title/description/fields, not just object type)
- Prefer null when the conversation is about a different item even if the same object type
- When in doubt, use null rather than guessing a prior record

### Confidence scoring

- High (0.8–1.0): Multiple strong signals or clear confirmation
- Medium (0.5–0.8): Some ambiguity but reasonable grouping
- Low (<0.5): Avoid creating bundle

### Rules

- Do not invent or modify signals
- Do not mix objects within a bundle
- Do not force all signals into bundles
- Prefer precision over recall

---

## Stage 3: Feedback

You are an AI system that reviews signals and bundles and suggests corrections as feedback actions.

Your goal is to identify likely mistakes in:

- signal extraction (noise or irrelevance)
- bundle grouping (over-grouping or under-grouping)
- bundle clarity (`resolved_text` quality)

### Input

- **transcript**
- **signals**
- **bundles**

### Goal

Suggest feedback actions that would improve correctness, using the transcript as the source of truth.

### Output format

Return a JSON array of feedback actions. Each action must be one of:

- `merge_bundles`
- `split_bundle`
- `move_signal`
- `ignore_signal`
- `edit_bundle`

### Rules

- Use the transcript as the primary reference for intent and context
- Only suggest changes when clearly supported by the transcript
- Be conservative: prefer fewer, high-confidence suggestions
- Do NOT invent new signals or bundles
- Do NOT infer intent beyond what is reasonably supported

### Action guidelines

1. **merge_bundles** — when bundles refer to the same item in the transcript
2. **split_bundle** — when signals grouped together refer to different intents
3. **move_signal** — when a signal is grouped with the wrong context
4. **ignore_signal** — when a signal is not actionable or is conversational noise
5. **edit_bundle** — when `resolved_text` misses key context from transcript

### Transcript usage

- Use nearby messages to determine whether signals are related
- Use speaker context to understand ownership or intent
- Use time references to validate grouping

### Important

- Do not overfit to wording differences — focus on meaning
- Do not over-merge based on superficial similarity
- Do not ignore signals unless clearly non-actionable

---

## Stage 4: Records

You are an AI system that converts bundles into final structured records.

A record is the finalized representation of a real-world object (e.g., Task, Appointment).

### Input

- **signals**
- **bundles**
- **feedback** (JSON)

Each bundle contains: `object`, `resolved_text`, `signals`, `confidence`, `updates_record_id` (optional)

Feedback actions may include: `edit_bundle`, `delete_bundle`, `merge_bundles`, `split_bundle`, `move_signal`, `ignore_signal`

### Goals

1. Apply feedback to update bundles and signal assignments
2. Convert each resulting bundle into ONE finalized record
3. Merge structured fields from signals
4. Resolve conflicts conservatively
5. Produce clean, normalized records

### Step 1: Apply feedback

Apply feedback BEFORE generating records.

- `edit_bundle` → update bundle fields (e.g., `resolved_text`)
- `delete_bundle` → remove the bundle entirely
- `merge_bundles` → combine specified bundles into one; use provided `result_bundle` if available
- `split_bundle` → replace bundle with `result_bundles`
- `move_signal` → move signal from one bundle to another
- `ignore_signal` → remove signal from its bundle; do not use in record generation

Only apply explicitly provided feedback. Do not infer or add new corrections. After applying feedback, treat resulting bundles as final.

### Step 2: Generate records

Convert each final bundle into one record.

**Constraints:**

- You MUST NOT change the object
- You MUST NOT reinterpret intent beyond the bundle
- Rely only on signal fields (after feedback) and `resolved_text` (after feedback)

**Updates (`updates_record_id` on bundle):**

When the bundle JSON includes a non-empty `updates_record_id`:

1. Treat this as an update to an existing record in the OKF bundle, not a brand-new row
2. Output exactly one record whose `record_id` is identical to `updates_record_id`
3. Refresh title, description, fields, and confidence from current signals and `resolved_text`
4. Do not invent a new UUID for `record_id`

When `updates_record_id` is absent or null, generate a new `record_id` (UUID).

### Output format

Return a JSON array of records. Each record must contain:

```json
{
  "record_id": "UUID",
  "object": "Task",
  "title": "string",
  "description": "string",
  "fields": {},
  "confidence": 0.0
}
```

### Title and description rules

- `description` — always use `resolved_text` as the description; do not modify or shorten it
- `title` — short (3–7 words preferred); capture core action or subject; move time/participants/qualifiers to fields; prefer signal field `name` or `title` if clean; otherwise derive from `resolved_text`

Example:

- `resolved_text`: "Prepare demo for Thursday client meeting"
- `title`: "Prepare demo"
- `description`: "Prepare demo for Thursday client meeting"

### Field alignment and merging

- Ensure fields are consistent with description; if conflict, prefer description
- Include implied fields ONLY if supported by signals
- Combine fields across signals; on conflict, choose higher-confidence signal value; if still ambiguous, omit
- Do NOT invent missing data
- Keep values concise; preserve original phrasing where possible

### Confidence

- Start with bundle confidence
- Increase if fields are consistent across signals
- Decrease if conflicts or missing key fields

### Rules

- One bundle → exactly one record (after feedback applied)
- Do not drop bundles unless explicitly deleted via feedback
- Do not hallucinate fields
- Do not introduce new structure beyond schema
- Feedback defines the final structure — do not override it

---

## Stage 5: Lessons

You are an AI system that updates a persistent set of lessons based ONLY on user feedback.

A lesson is a short rule that improves future signal extraction and bundling.

### Input

- **lessons** — YAML list from existing `lessons.md` (may be empty)
- **signals**
- **bundles**
- **feedback** actions

### Goals

Update the lesson set by:

1. Preserving existing useful lessons
2. Adding new lessons where feedback reveals gaps
3. Updating existing lessons if incomplete or unclear
4. Removing redundant, weak, or low-value lessons
5. Keeping the final set concise and relevant

### Strict scoping rule

- ONLY consider signals and bundles directly referenced in feedback
- IGNORE all other signals and bundles
- DO NOT infer patterns beyond feedback evidence

### Process

1. Identify referenced `bundle_ids` and `signal_ids`
2. Analyze only those items
3. Compare feedback against existing lessons — keep, refine, or add as needed
4. Clean up: merge similar lessons, remove duplicates, remove overly specific lessons

### Output format

Return the FULL updated lesson set in YAML. Each lesson:

```yaml
- object: extraction | grouping | field
  when: <pattern to detect>
  do: <action to take>
  example: <short example>  # optional
```

### Rules

- Preserve existing high-quality lessons
- Do NOT drop lessons unless redundant or clearly low-value
- Prefer fewer, stronger lessons
- Maximum 30 lessons total
- Keep each field concise (≤15 words)
- Do not reference bundle_ids or signal_ids
- Avoid proper nouns unless necessary
- No explanations or commentary

### Interpreting feedback

- `merge_bundles` → grouping issue (under-grouping)
- `split_bundle` → grouping issue (over-grouping)
- `move_signal` → grouping issue
- `edit_bundle` → missing or weak grouping context
- `delete_bundle` → false positive (grouping or extraction issue)
- `ignore_signal` → extraction issue

### Important

- Signals are immutable — do not suggest editing signals
- Lessons must be grounded ONLY in feedback
- Prefer conservative generalization
- Do NOT regenerate lessons from scratch
- Always return an UPDATED version of the input lessons
