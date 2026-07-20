# Schema Suggestion Vocabulary

Hint vocabulary for workspace schema discovery. Prefer these names when they fit the user's data; allow domain-specific PascalCase objects and field names when they do not.

This is **not** a fixed catalog. Continuity discovers the schema from the business's spreadsheets, documents, and answers — then stores it in `inwrk/schema.md`. See [schema-setup.md](schema-setup.md).

**In scope for persisted schema:** Objects, Properties (field types), Relationships, Rules.

**Out of scope:** Actions (verbs), Interfaces (views), Intelligence (AI capabilities). Do not write those sections into `schema.md`.

---

## Objects (nouns)

Use as candidate object type names. Map spreadsheet tabs, repeated entities, and interview answers to these when the fit is clear.

| # | Name | Typical meaning |
|---|------|-----------------|
| 1 | Company | Organization or legal entity |
| 2 | Person | Individual (employee, contact, etc.) |
| 3 | Contact | Reachable person record |
| 4 | Lead | Sales prospect |
| 5 | Deal | Closed or in-pipeline sale |
| 6 | Account | Customer or billing account |
| 7 | User | System or workspace user |
| 8 | Team | Group of people |
| 9 | Role | Permission or job role |
| 10 | Opportunity | Pipeline opportunity (may overlap Deal) |
| 11 | Product | Sellable product |
| 12 | Service | Sellable or delivered service |
| 13 | Project | Multi-step work effort |
| 14 | Task | Action item |
| 15 | Subtask | Child of a Task |
| 16 | Doc | Document concept |
| 17 | File | File reference |
| 18 | Note | Freeform note |
| 19 | Event | Calendar or business event |
| 20 | Meeting | Scheduled meeting |
| 21 | Invoice | Bill to customer |
| 22 | Payment | Money movement |
| 23 | Subscription | Recurring plan enrollment |
| 24 | Plan | Pricing or service plan |
| 25 | Price | Price point |
| 26 | Campaign | Marketing campaign |
| 27 | Message | Message instance |
| 28 | Email | Email message |
| 29 | Call | Phone or video call |
| 30 | Activity | Generic activity log entry |
| 31 | Workflow | Named process |
| 32 | Template | Reusable template |
| 33 | Tag | Label |
| 34 | Location | Place |
| 35 | Vendor | Supplier |

When none fit, invent a singular PascalCase name (e.g. `Shipment`, `Incident`, `PolicyClaim`).

---

## Properties (field types)

Use these as the `type` for fields on objects. Pair with a snake_case or clear field `name`.

| # | Name | Use for |
|---|------|---------|
| 36 | Status | Lifecycle / state enum |
| 37 | Boolean | True/false |
| 38 | Date | Calendar date (ISO 8601 date) |
| 39 | DateTime | Date and time (ISO 8601) |
| 40 | Number | Numeric quantity |
| 41 | Currency | Money amount (+ optional currency field) |
| 42 | Percent | Percentage |
| 43 | Text | Short or long string |
| 44 | URL | Link |
| 45 | Email | Email address |
| 46 | Phone | Phone number |
| 47 | Address | Postal or structured address |
| 48 | JSON | Structured blob when unavoidable |
| 49 | Enum | Closed set of string values |
| 50 | ID | Identifier (external or internal) |
| 51 | Owner | Person responsible |
| 52 | Created At | Creation timestamp |
| 53 | Updated At | Last-modified timestamp |
| 54 | Start Date | Period start |
| 55 | Due Date | Deadline |
| 56 | Priority | Priority enum or rank |
| 57 | Score | Numeric score |
| 58 | Count | Integer count |
| 59 | Duration | Length of time |
| 60 | Time | Time of day |

Common field name conventions: `status`, `owner`, `due_date`, `created_at`, `email`, `amount`. Prefer vocabulary property types in schema field definitions (e.g. `type: Enum` with listed values).

---

## Rules (labels only)

When proposing validation, deduplication, or process constraints, label them with these kinds. Continuity does **not** execute them as an automation engine — they guide extract, review, and identity matching.

| # | Name | Use for |
|---|------|---------|
| 109 | Permission | Who may create/update (document only) |
| 110 | Policy | Business policy constraint |
| 111 | Trigger | When something should happen (document only) |
| 112 | Schedule | Time-based constraint |
| 113 | Condition | If/when guard for a rule |
| 114 | Action | What to do when a condition holds (guidance text) |
| 115 | Audit | What to retain for provenance |

Typical Continuity rules:

- **Validation** — required fields, enum membership, format (email, currency)
- **Deduplication** — identity keys (e.g. match `Person` on `email`; match `Company` on normalized name)
- **Policy** — soft constraints stated in the schema for review (e.g. "Deal without Company is incomplete")

---

## Naming guidance

1. Prefer a vocabulary object name when the entity clearly matches
2. Prefer vocabulary property types for field `type`
3. Keep object names PascalCase singular; field names snake_case
4. Do not invent Actions, Interfaces, or Intelligence sections in `schema.md`
5. Relationships are edges between objects (see [schema-setup.md](schema-setup.md)), not vocabulary nouns
