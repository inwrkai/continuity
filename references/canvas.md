# Visualizing an inwrk Bundle (Canvas)

How to suggest and build an interactive live canvas from an existing `inwrk/` knowledge bundle — an explorer dashboard plus a relationship graph.

Use this reference when the user asks to **visualize**, open a **canvas**, **show my knowledge base**, or **refresh the continuity canvas**. Also follow the **suggest** rules below after extract summaries, query answers, and schema setup when a bundle with records exists.

## Suggest vs open

| Situation | Action |
|-----------|--------|
| Extract Step 6 summary, query answer, or confirmed schema with records | End with one line offering a live canvas (e.g. "Say **visualize** to open an interactive knowledge-base canvas.") |
| User explicitly asks to visualize / canvas / show knowledge base | Read this file; build or refresh the canvas |
| No `inwrk/` or zero records | Do not open a canvas; explain what is missing |
| Agent has no live-canvas support | Still suggest when a bundle exists; if the user asks, explain canvas is unavailable here and answer with compact tables in chat |

**Do not** auto-open a canvas after every extract or query. Suggest always; open only on user request.

## Runtime detection (portable)

1. If a **canvas skill** is available in this agent (Cursor canvas, Claude browser-canvas, agent-canvas, etc.), **read that skill first** and follow its file location, imports, and constraints.
2. **Cursor** — write one file to `~/.cursor/projects/<workspace>/canvases/continuity-kb.canvas.tsx` (kebab-case). Reuse this filename on refresh. Import **only** from `cursor/canvas`. Embed all data inline — no `fetch()`, no relative imports, no npm packages.
3. **Other hosts** — same UX intent and data model; adapt components to the host canvas skill (e.g. `App.jsx` under `.claude/artifacts/` for browser-canvas). Do not use Cursor-only APIs on non-Cursor agents.
4. Link the canvas file in chat with a **full absolute path** (required on Cursor).

Resolve `<workspace>` from the current workspace path (terminals, open files, or `~/.cursor/projects/` listing). Do not guess.

## Loading bundle data

Summary-first, same as [query.md](query.md):

1. Read `inwrk/index.md` — bundle title, description, `object_types`
2. When `inwrk/schema.md` exists with `status: confirmed`, use it for object types, relationships, and `schema_version`
3. Read `inwrk/records/index.md` for the full listing
4. Load record bodies when the bundle is small or when relationship field values are needed for the graph; for large bundles use index rows plus key frontmatter fields
5. When present, read the **tail** of `inwrk/events.jsonl` (last ~30 lines) for the Activity feed
6. When present, parse `inwrk/automations.md` YAML for the Automations panel (id, name, status, short trigger summary)

Do not invent records, relationships, events, or automations not present in the bundle.

## Embedded data model

Build a compact inline constant (TypeScript on Cursor; equivalent on other hosts) embedded in the canvas source:

```ts
type KbMeta = {
  title: string;
  description?: string;
  schemaVersion?: number;
  sourcePath: string; // e.g. "inwrk/"
  generatedAt: string; // ISO 8601
};

type KbRecord = {
  id: string;
  slug: string;
  title: string;
  type: string;
  confidence?: string;
  status?: string;
  description?: string;
  fields: Record<string, string>;
  related?: Array<{ field: string; toType: string; toTitle: string; toId?: string }>;
};

type KbGraphEdge = {
  fromId: string;
  toId: string;
  field: string;
};

type KbEvent = {
  id: string;
  ts: string;
  event: string;
  title?: string;
  type?: string;
  source: string;
};

type KbData = {
  meta: KbMeta;
  records: KbRecord[];
  edges: KbGraphEdge[];
  typeCounts: Record<string, number>;
  schemaFields?: Record<string, Array<{ name: string; type: string; required?: boolean; enumValues?: string[] }>>;
  events?: KbEvent[];       // tail of events.jsonl for Activity feed
  automations?: Array<{ id: string; name: string; status: string; triggerSummary: string }>;
};
```

### Deriving relationships

1. Parse **Relationships** from confirmed `schema.md` (From, Field, To, Cardinality)
2. For each record, inspect fields that map to related object types
3. Match field values to other records by title, slug, or identity keys from schema dedup rules
4. Emit `KbGraphEdge` only when both endpoints resolve to real record ids; skip dangling edges

## Canvas layout (both views)

Single default-export top-level component. Visual hierarchy:

### 1. Header

- Bundle title from `index.md`
- Total record count
- Schema version when present

### 2. Stats row

- One `Stat` (or host equivalent) per object type with count > 0
- Omit types with zero records

### 3. Explorer

- Filter by object type (Select, pills, or toggle group)
- Searchable table: title, type, status, confidence
- Selecting a row shows a **detail** panel (see [Editing (CRUD)](#editing-crud) below)
- On Cursor, persist filter and selection with `useCanvasState` (e.g. keys `"filterType"`, `"selectedId"`)

### 4. Relationship graph

- Nodes = records (label = title; color or swatch by type)
- Edges = derived `KbGraphEdge` list
- On Cursor: `computeDAGLayout` from `cursor/canvas` plus SVG rendering; style back-edges differently when flagged
- On other hosts: equivalent graph or structured adjacency list fallback
- **Omit** the graph section when fewer than 2 nodes or zero edges — never render empty placeholders

### 5. Caption

- Source path (`inwrk/` or custom) and generation timestamp

### 6. Activity feed (read-only)

When `events.jsonl` has at least one event, add a compact **Activity** section (the Feed interface from the vocabulary):

- Show the last ~20–30 events, newest first
- Each row: timestamp, event type, title (or type), source
- Highlight `automation.fired` distinctly from record mutations
- **Omit** this section when `events.jsonl` is missing or empty — never show an empty placeholder

Embed a small `KbEvent[]` array in the canvas data (tail of the log only — do not embed the entire file). See `KbEvent` in the embedded data model above.

### 7. Automations panel (read-only)

When `automations.md` lists one or more automations, add a compact panel:

- Name, status (`draft` / `confirmed`), trigger summary (one line)
- Optional last-fired hint from recent `automation.fired` events when available
- **Omit** when there are no automations
- **Do not** edit or run automations from the canvas in this version — authoring stays in chat per [automations.md](automations.md)

## Editing (CRUD)

The canvas supports batched create, update, and delete. The canvas queues changes; the agent writes `inwrk/` when the user clicks **Apply**. See [canvas-update.md](canvas-update.md) for the apply protocol.

The canvas never writes markdown files directly.

### Patch types (embed in canvas source)

```ts
type CanvasOp =
  | { op: "update"; record_id: string; title?: string; description?: string; fields?: Record<string, string> }
  | { op: "create"; temp_id: string; type: string; title: string; description?: string; fields?: Record<string, string> }
  | { op: "delete"; record_id: string };

type CanvasPatch = {
  source: "continuity-canvas";
  bundlePath: string;
  generatedAt: string;
  ops: CanvasOp[];
};
```

### Embedded schema for forms

When building the canvas, embed compact field defs from `schema.md` or defaults into `KbData.schemaFields` so the detail form can render:

- `Select` for enum / status fields
- `TextInput` / `TextArea` for text fields
- Required markers on schema-required fields

### Pending queue

On Cursor:

- `useCanvasState("pendingOps", [] as CanvasOp[])`
- Dirty indicator in header when `pendingOps.length > 0`
- **Apply N changes** primary button (disabled when queue empty)
- **Discard** clears the queue; use a two-step pattern (Discard → Confirm discard) to avoid accidental loss
- `useCanvasAction()` for Apply on Cursor

**Apply** builds a `CanvasPatch` and dispatches:

```ts
dispatch({
  type: "newComposerChat",
  userPrompt:
    "Apply this Continuity canvas patch per references/canvas-update.md:\n" +
    JSON.stringify(patch),
});
```

The canvas file is auto-@mentioned by `newComposerChat`. Do not pass the canvas path from UI code.

On other hosts with canvas state sync: queue ops the same way; Apply instructs the user to ask the agent to apply canvas changes and include the JSON patch (or read pending state from the host artifact).

### Explorer / detail edits

- Detail panel: editable title, description, status/enums, and other schema fields
- On change: upsert one `update` op per `record_id` (merge changed keys into a single op)
- Table shows a dirty pill for records with pending updates or deletes
- Pending creates appear in the table with a distinct tone until Apply

### Create

- **New record** control: Select object type → form with required fields → **Add to queue** pushes a `create` op with `temp_id` (e.g. `tmp-<uuid>`)

### Delete

- **Delete** on detail (and optionally row action) queues `delete`; strike through or hide in UI until Apply
- Deleting a pending `create` removes that create op from the queue instead of queuing a delete

### Open in editor (convenience)

Optional secondary **Open in editor** via `useCanvasAction({ type: "openFile", path: "inwrk/records/<slug>.md" })`. Does not replace Apply.

## Design rules (Cursor)

When building on Cursor, follow the canvas skill:

- Colors from `useHostTheme()` tokens only — no hardcoded hex
- No gradients, box-shadows, or emojis as decoration
- No empty states — omit sections with no data
- Mix open sections with cards; avoid a wall of identical cards
- Label charts and tables when used

On other hosts, follow that host's canvas skill aesthetics while preserving the same information architecture.

## Refresh

When the user asks to **refresh** or **re-visualize**, rebuild from the current `inwrk/` bundle and overwrite the same canvas filename (`continuity-kb.canvas.tsx` on Cursor) so the user keeps one artifact.

## Introducing the canvas

When you create or update a canvas:

1. Include a markdown link to the canvas file using its full absolute path
2. One sentence: the user can open it beside the chat
3. On first canvas in the workspace, optionally explain what a canvas is (one sentence)

Example chat line after building:

```
Open the knowledge-base canvas beside the chat: [continuity-kb](/Users/<user>/.cursor/projects/<workspace>/canvases/continuity-kb.canvas.tsx)
```

## Do not

- Auto-open a canvas without the user asking
- Write Cursor `.canvas.tsx` paths or `cursor/canvas` imports on agents without Cursor canvas support
- Render empty graph or table sections
- Invent records, fields, edges, events, or automations for the visualization
- Use `fetch()` or network calls inside the canvas
- Store the canvas inside `inwrk/` — canvas files live in the host-managed canvas directory
- Write or delete `inwrk/` record files from canvas runtime — queue ops and Apply via agent per [canvas-update.md](canvas-update.md)
- Edit or confirm automations from the canvas (read-only panel only)
- Auto-Apply on every keystroke
