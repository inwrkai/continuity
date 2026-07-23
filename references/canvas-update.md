# Applying Canvas Patches to inwrk

How to apply batched create, update, and delete operations from the Continuity knowledge-base canvas to the `inwrk/` bundle on disk.

Use this reference when:

- The user clicks **Apply** in the canvas (Cursor: `newComposerChat` with a JSON patch)
- The user says **apply canvas changes** and pastes or references a patch
- A chat opened from the canvas includes a `CanvasPatch` payload

The canvas never writes markdown directly — the agent validates and applies the patch per this file and [okf-output.md](okf-output.md).

## Patch contract

```ts
type CanvasOp =
  | {
      op: "update";
      record_id: string;
      title?: string;
      description?: string;
      fields?: Record<string, string>; // only changed keys
    }
  | {
      op: "create";
      temp_id: string; // client-side id until agent assigns UUID
      type: string;
      title: string;
      description?: string;
      fields?: Record<string, string>;
    }
  | {
      op: "delete";
      record_id: string;
    };

type CanvasPatch = {
  source: "continuity-canvas";
  bundlePath: string; // e.g. "inwrk/"
  generatedAt: string; // ISO 8601
  ops: CanvasOp[];
};
```

The canvas embeds these types and builds `CanvasPatch` when the user clicks Apply. See [canvas.md](canvas.md) for the UI workflow.

## When to apply

| Trigger | Action |
|---------|--------|
| Apply chat from canvas with valid patch | Follow this protocol |
| User says "apply canvas changes" with patch JSON | Same |
| Empty `ops` array | Report nothing to apply |
| Invalid or partial patch | Report validation errors; do not write partial corrupt state without user confirmation |

Do **not** re-run the extract pipeline. Do **not** bump `schema.md`.

## Apply protocol

Run Step 0 from [SKILL.md](../SKILL.md) if needed, then:

### 1. Load bundle context

1. Resolve `bundlePath` (default `inwrk/` in the current workspace)
2. Load confirmed `schema.md` when present; otherwise use `index.md` `object_types` / `object_schemas` and defaults from [objects.md](objects.md)
3. Read `records/index.md` and load record files referenced by update/delete ops (match by `record_id` in frontmatter)

### 2. Validate each op

**update**

- `record_id` must match an existing record file
- `title`, `description`, and `fields` keys must be allowed for that object type in the schema
- Enum values must match schema status lists or field enums
- Reject unknown field keys — flag as schema misfit; do not invent columns

**create**

- `type` must be in `object_types` / schema objects
- `title` is required and non-empty
- Required schema fields must be present in `fields` or derivable from `title`/`description` (e.g. `name` = title when required)
- Set `confidence: high` (user-authored via canvas)
- `temp_id` is for canvas UI only — assign a new UUID `record_id` on write

**delete**

- `record_id` must match an existing record file
- Do not delete if the same `record_id` also has a pending update in the same patch — treat delete as authoritative

**Cross-op rules**

- Phase 1: pending creates do not link to other pending creates via relationship fields
- If the same `record_id` appears in multiple update ops, merge field changes into one update
- If create and delete share a `temp_id` (user undid a pending create), drop both

### 3. Apply order

Process ops in this order regardless of array order:

1. **create** — assign UUIDs; write new `records/<slug>.md` files
2. **update** — edit matching files in place
3. **delete** — remove `records/<slug>.md` files

Track created / updated / deleted titles for the chat summary.

### 4. Write per [okf-output.md](okf-output.md)

**create**

1. Generate a new UUID for `record_id`
2. Derive slug from `title`; apply slug-collision rule if needed
3. Write frontmatter, `# Fields` table, and provenance note (below)
4. Append **Creation** to `log.md`
5. Append `record.created` to `events.jsonl` (`source: canvas`) per [events.md](events.md)

**update**

1. Find file by `record_id` in frontmatter
2. Diff prior fields for the `changes` object before overwriting
3. Update frontmatter, `# Fields` body, and `timestamp` (ISO 8601 now)
4. If `title` changed and slug should change, rename file only when no collision; otherwise keep slug and update title in frontmatter
5. Append **Update** to `log.md`
6. Append `record.updated` to `events.jsonl` with `changes` (`source: canvas`)

**delete**

1. Find file by `record_id`; note title and type for the log
2. Remove the record file
3. Append **Deletion** to `log.md`
4. Append `record.deleted` to `events.jsonl` (`source: canvas`)

After all ops: regenerate `records/index.md`.

### 4.5. Evaluate automations

After events are appended:

1. Follow [automations.md](automations.md) evaluation protocol — process events after `event_cursor`, lazy schedule check
2. Auto-run matching **internal** actions; regenerate `records/index.md` if automations created/updated records
3. Queue **external** actions for the chat summary (propose-and-approve; do not send yet)
4. Include automations fired and open notifications in the apply summary

### 5. Provenance

For canvas-sourced creates and updates, append or set a short note in the record body:

```markdown
# Citations

1. Updated via Continuity canvas on 2026-07-21.
```

Do not invent transcript citations. Preserve existing citations on update unless the user explicitly replaced description content.

### 6. Refresh canvas

After a successful apply:

1. Report created / updated / deleted titles with object types
2. Rebuild the canvas from the live bundle per [canvas.md](canvas.md) Refresh — overwrite `continuity-kb.canvas.tsx` (or host equivalent)
3. Link the refreshed canvas in chat

## Chat summary shape

```
Canvas patch applied
- Created: 1 (Follow up with Acme — Task)
- Updated: 2 (Prepare demo — Task; Client call — Appointment)
- Deleted: 1 (Stale blocker — Blocker)
- Automations: 1 fired (Won deal → invoice task)
- External pending: none
- Output: inwrk/
```

If validation failed:

```
Canvas patch rejected
- 1 error: unknown field `priority` on Task (not in schema)
- No files written
```

## Do not

- Re-run extract/review stages
- Auto-bump `schema.md`
- Invent `record_id` values (except new UUIDs on create)
- Write `inwrk/` files from canvas runtime without going through this protocol
- Apply ops with unknown field keys silently
- Skip `log.md` or `records/index.md` regeneration
- Skip event emission or automation evaluation after a successful write
- Execute external automation actions without user approval
- Let automation-sourced events re-trigger automations
