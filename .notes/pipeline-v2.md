# Extraction Pipeline v2 — Source → Truth

> **Status: DRAFT — designed stage by stage; all stages drafted, decisions in the log at the
> bottom.** Stage 5's decisions are provisional defaults (not yet explicitly confirmed).
> This document supersedes the *storage* patterns in `financial-pipeline-spec.md`; the conceptual
> model there (precedents, registry, computed proposals) still applies unless overridden here.

## Design principles

1. **Relational first.** Everything derived from a parse is stored as relational rows that reference
   each other. Exactly **one** big JSONB exists in the system: the raw parse artifact (Stage 2).
2. **Immutable inputs, append-only audit.** Raw files, parse artifacts, and decision/resolvement
   records are never mutated. Re-running a stage produces new rows; nothing is overwritten in place.
3. **Provenance end to end.** Every value in truth can be walked back: truth row → proposal →
   atom/label → cell → table → parse artifact → source file, with bboxes at every geometric step.
4. **One resolvement machine.** All labels — metric headers, entity mentions, KV keys — go through a
   single universal precedent + candidate + reasoning system. One store, one audit shape.
5. **Few object classes.** Reuse the same relational objects across table and non-table extraction
   (a freestanding value — KV pair or narrative — reuses the same `label` + atom payloads as a
   table cell). No parallel near-duplicate concepts.
6. **Modular stages, event bus in between.** Each stage reads only the persisted output of the
   previous stage and emits exactly one completion event. Any stage's *implementation* (e.g. how
   anchors are assigned) can be swapped without touching neighbors.

## Stage map

| # | Stage | Input | Persisted output | Emits |
|---|---|---|---|---|
| 1 | Source upload | raw PDF | R2 object + `sources` row | `source.uploaded` |
| 2 | Parse | source file | `parses` row + artifact JSONB + `pages` | `source.parsed` |
| 3 | Structural extraction | parse artifact | run + tables, axes, cells, atoms, labels | `run.structured` |
| 4 | Non-table extraction | parse artifact + Stage 3 rows | freestanding values (same payload objects) | `run.values_extracted` |
| 5 | Anchoring | Stage 3+4 rows | anchor rows | `run.anchored` |
| 6 | Label resolvement | labels + precedents | resolvements + precedent writes | `run.resolved` |
| 7 | Proposals | resolvements + registry + anchors | proposal rows + computation audit | `run.proposed` |
| 8 | Commit | approved proposals | truth rows + decision log | `proposal.committed` |

Out of scope for v2 (dropped for now): **bundle splitting**. A source is one logical document; a
PDF containing several statements is not split by the pipeline.

### The event bus

Each stage is an isolated job triggered by exactly one event and emitting exactly one event.
Re-running any stage (new run, new selector, precedent change) re-emits its event and the chain
downstream replays — no stage ever reaches around its neighbor.

```mermaid
flowchart LR
  A[source.uploaded] --> P[parse]
  P --> B[source.parsed]
  B --> S[structure]
  S --> C[run.structured]
  C --> V[extract values]
  V --> D[run.values_extracted]
  D --> N[anchor]
  N --> E[run.anchored]
  E --> R[resolve labels]
  R --> F[run.resolved]
  F --> O[compute proposals]
  O --> G[run.proposed]
  G -.review.-> H[decisions]
  H --> I[proposal.committed]
  X[precedent minted] -.re-resolve held labels.-> F
```

### The run spine

Stages 3–7 do not hang off `parse_id` directly. A thin `extraction_runs` row is the derivation
spine: it records *which code* derived *which relational rows* from *which parse*. This is what
makes the stages modular — fixing a bug in the deterministic table miner means starting a new run
over the same parse, never re-parsing and never mutating the old run's rows.

```
extraction_runs
  id                uuid PK
  parse_id          FK parses
  organization_id   FK
  pipeline_version  text        -- code version of the deterministic stages
  status            enum: running | structured | values_extracted | anchored | resolved | proposed | failed
  created_at        timestamptz
```

- One *active* run per parse (same pointer mechanism as active parse; see Stage 2).
- Every Stage 3+ row FKs `run_id`. Deleting a superseded run cascades its derivation cleanly.

---

## Stage 1 — Source upload

Someone (user or connector) uploads a raw file. The file goes to R2 unmodified; a `sources` row is
the provenance root for everything downstream.

### Storage

```
sources
  id                uuid PK
  organization_id   FK
  uploaded_by       FK member (nullable — connectors)
  file_name         text
  mime_type         text
  byte_size         int
  content_hash      text        -- sha256 of the raw bytes; duplicate detection
  r2_key            text        -- immutable; never rewritten
  status            enum: uploaded | parsing | parsed | failed
  created_at        timestamptz
```

Rules:

- The R2 object is written once and never mutated. `r2_key` includes the content hash so the same
  bytes always land on the same key.
- **Duplicate upload** (same `content_hash` within the org): a *new* `sources` row is created —
  who uploaded what when is itself provenance — but it points at the existing R2 object and the
  pipeline is **not** re-run; the new source links to the existing parse.

**Emits `source.uploaded`** → triggers Stage 2.

---

## Stage 2 — Parse (LlamaParse)

The parser turns the raw file into a machine-readable artifact. This artifact is **the only big
JSONB in the system**: it is the frozen, versioned snapshot of what the parser saw, and every
relational row downstream is deterministically derivable from it.

### Storage

```
parses
  id                uuid PK
  source_id         FK sources
  parser            text        -- 'llamaparse'
  parser_version    text        -- version + config hash; identifies determinism envelope
  status            enum: pending | succeeded | failed
  error             text (nullable)
  page_count        int
  artifact          jsonb       -- the raw LlamaParse output, verbatim. THE one big JSONB.
  created_at        timestamptz
  completed_at      timestamptz

sources (addition)
  active_parse_id   FK parses (nullable)   -- explicit pointer to the live parse

pages
  id                uuid PK
  parse_id          FK parses
  page_index        int         -- 0-based
  width             numeric     -- PDF points; the bbox coordinate space
  height            numeric
  UNIQUE (parse_id, page_index)
```

Rules:

- The artifact is stored **verbatim** — no cleanup, no enrichment. If we need a cleaned/derived
  shape, that is Stage 3's relational output, not a second JSONB.
- Artifact location: **JSONB in Postgres for now**; if artifact size becomes a problem, move to an
  R2 JSON object with the `parses` row keeping metadata + key. Nothing downstream depends on the
  location — Stage 3 reads it through one accessor.
- A source can have multiple parses (re-parse after a parser upgrade). Parses are append-only;
  `sources.active_parse_id` designates the live one explicitly. All downstream rows key on
  `parse_id` (via the run spine), so re-parsing creates a parallel derivation, never a mutation.
- **Pages are first-class**: parse completion writes one `pages` row per page (index + dimensions).
  Every geometric row downstream FKs `page_id`, so bbox provenance always has its coordinate space
  one join away. All stored bboxes are normalized `[x0, y0, x1, y1]` in `0.0–1.0`; the `pages`
  dimensions let you map back to PDF points.
- All bboxes downstream are copied out of this artifact at Stage 3; the artifact remains the ground
  truth to re-derive them from.

### The artifact, by example

The shape below is the real `LlamaParseArtifact` the parser seam produces today
(`lib/parsers/llamaparse.ts`). It bundles the LlamaParse result's parallel views — structured
items, grounded per-cell bboxes, markdown, plain text — into one frozen blob. Comments annotate
which Stage-3/4 output each part feeds.

```jsonc
{
  "parserVersion": "llamaparse@2026-05",   // determinism envelope: parser + our seam version
  "tier": "agentic_plus",                  // parse tier used (cost/quality knob)
  "version": "0.6.2",                      // pinned LlamaParse tier version
  "pageCount": 4,

  // ── VIEW 1: layout items per page (`expand=items`) ─────────────────────────
  // Feeds Stage 3 (tables, axes, cells) + Stage 4 (freestanding values).
  "pages": [
    {
      "page_number": 1,
      "page_width": 612, "page_height": 792,   // PDF points; needed to normalize bboxes to [0,1]
      "items": [
        { "type": "heading", "level": 1,
          "value": "Capital Account Statement",
          "bbox": [{ "x": 43, "y": 51, "w": 320, "h": 22 }] },

        { "type": "text",
          "value": "Fund: ABC Growth Fund III, L.P.\nInvestor: Jane Doe Family Trust",
          "bbox": [{ "x": 43, "y": 90, "w": 400, "h": 36 }] },     // → Stage 4 freestanding values

        { "type": "table",
          // cell grid: header rows + body rows as stringy cells
          "rows": [
            ["Description", "Current Quarter", "Year-to-Date"],
            ["Beginning Capital Balance", "$12,500,000", "$11,800,000"],
            ["Capital Contributions", "$500,000", "$1,200,000"]
          ],
          // html preserves colspan / multi-row header structure the flat grid loses
          "html": "<table><thead><tr><th>Description</th>…</thead>…</table>",
          "bbox": [
            { "x": 40, "y": 240, "w": 530, "h": 210, "label": "table", "confidence": 0.98 }
          ] }
      ]
    }
  ],

  // ── VIEW 2: grounded sidecar — per-CELL bboxes (one entry per page) ─────────
  // Aligned to view 1 by page + table order (nth table item ↔ nth grounded table).
  // This is where Stage-3 cell/row/column bboxes come from.
  "groundedItems": [
    {
      "page": 1,
      "tables": [
        { "rows": [
            [ { "bbox": { "x": 44, "y": 246, "w": 120, "h": 16 } },   // header cell (0,0)
              { "bbox": { "x": 210, "y": 246, "w": 110, "h": 16 } },
              { "bbox": { "x": 360, "y": 246, "w": 100, "h": 16 } } ],
            [ { "bbox": { "x": 44, "y": 268, "w": 180, "h": 15 } },
              { "bbox": { "x": 214, "y": 268, "w": 96, "h": 15 } },
              { "bbox": { "x": 362, "y": 268, "w": 96, "h": 15 } } ]
          ] }
      ]
    }
  ],

  // ── VIEW 3+4: per-page markdown + plain text ───────────────────────────────
  // Feeds anchor extraction (Stage 5 LLM reads markdown) and any full-text search.
  "markdownByPage": [ { "page_number": 1, "markdown": "# Capital Account Statement\n\nFund: ABC…" } ],
  "textByPage":     [ { "page_number": 1, "text": "Capital Account Statement Fund: ABC…" } ],

  // page-level parse metadata (confidence, cost_optimized, …)
  "metadata": [ { "page": 1, "confidence": 0.99 } ]
}
```

Why all four views stay in the one artifact rather than being split: they are different projections
of the *same parse* and are only consistent with each other within it. Stage 3 joins views 1+2 to
build relational geometry; Stage 4/5 read view 3. Nothing downstream ever needs to re-call
LlamaParse.

**Emits `source.parsed`** → triggers Stage 3.

---

## Stage 3 — Structural extraction

A **pure, deterministic** function of the parse artifact: no LLM, no DB lookups, no network. It
joins the artifact's layout view (`pages[].items`) with the grounded sidecar (`groundedItems`) and
writes the relational substrate: tables, axes (rows + headers), cells, atoms, labels. Re-running
Stage 3 with the same artifact and the same `pipeline_version` must produce identical rows.

### The object model

```
table ──< table_axes (axis: row | column; hierarchical) ──> label   (descriptor position)
   │
   └──< table_cells ──> row axis, column axis (leaf)
                   └──> exactly one of:  quantity_atom | date_atom | label (value position)
```

- **`tables`** — one per detected table.
- **`table_axes`** — one object class for *both* rows and column headers. A column header and a row
  are the same thing structurally: an indexed line of the grid that may carry a label and may sit
  in a hierarchy (spanning column headers ↔ indented row groups — the *same* parent mechanism).
  This is the "reduce similar concepts" move: instead of `table_rows` + `table_columns` +
  `table_headers`, one class covers all three.
- **`table_cells`** — the grid intersections. A cell FKs its row axis and its leaf column axis,
  carries its own bbox (from the grounded sidecar), and exactly one payload reference (below).
- **Atoms** (`quantity_atoms`, `date_atoms`) — deterministically parsed values. Pure payload: no
  geometry, no label text, and **never resolved** — an atom has no queue, no candidates, no
  precedent. Geometry and context live on whatever carries the atom (a cell here; a KV pair or
  narrative span in Stage 4).
- **`labels`** — pure text identity for any string whose *meaning* requires resolvement: surface
  text, normalized form, structural kind, cross-document `label_key`. No geometry — the carrier
  (axis, cell, KV key, narrative span) owns the bbox, so nothing is stored twice.

### The atom/label seam

The discriminator between atom and label is **not** number-vs-text — it is:

> Does deterministic lexical normalization fully capture the meaning (→ atom), or does the string
> need semantic resolvement (→ label)?

Quantities (`$12,500,000` → `12500000 USD`) and dates (`3/15/24` → `2024-03-15`) are atoms.
**Every other string in a value position is a label** — an entity name, a status, an IBAN. Its
resolvement decides what it means (entity link, enum member, or "literal — take the string
as-is"), through the same universal machinery as header labels.

**Descriptor labels vs value labels.** A label plays one of two roles, separated *structurally* by
which slot references it — never by a flag on the label itself:

| | Descriptor label | Value label |
|---|---|---|
| Referenced from | `table_axes.label_id`, `kv.key_label_id` (Stage 4) | `table_cells.value_label_id`, `kv.value_label_id`, narrative mention |
| What it is | *describes* values stored elsewhere (a header names what its cells mean) | **is** the value — the surface string is the payload |
| Needs a companion value? | yes — its cells / KV value carry the atoms | no — it is self-contained |
| Resolvement supplies | the meaning of *other* rows' payloads (canonical metric, period, entity axis, carrier) | the reference/type of *itself* (entity link, literal, enum) |
| Default expectation | canonical metric / period / entity / carrier | **entity mapping** (names are the common case); literal/enum are the exceptions |

Both roles run through one resolvement machine (Stage 6) with one audit shape. The role is derived
from the FK topology, so nothing is stored twice and no label ever needs a "role" column.

### Storage

```
tables
  id                  uuid PK
  run_id              FK extraction_runs
  page_id             FK pages         -- page the table starts on
  table_index         int              -- nth table in document order
  bbox                numeric[4]
  n_rows              int              -- body rows (excl. header rows)
  n_cols              int              -- leaf columns
  template_signature  text             -- hash(ordered header paths + provider + doc type); scopes structural precedents

table_axes
  id                  uuid PK
  table_id            FK tables
  axis                enum: row | column
  axis_index          int              -- row index / leaf-column index; ordering = structural correctness
  span_end            int (nullable)   -- spanning header: covers axis_index..span_end
  level               int              -- hierarchy depth: header level / row indent level
  parent_axis_id      FK table_axes (nullable)
  bbox                numeric[4] (nullable)   -- row band / column band / header cell box
  label_id            FK labels (nullable)    -- null: an unlabeled axis (e.g. blank stub)
  UNIQUE (table_id, axis, level, axis_index)

table_cells
  id                  uuid PK
  table_id            FK tables
  row_axis_id         FK table_axes
  column_axis_id      FK table_axes    -- always a LEAF column
  row_index           int              -- denormalized for cheap grid reconstruction
  col_index           int
  raw_text            text
  bbox                numeric[4]
  -- exactly ONE of the three payload refs is set (CHECK constraint);
  -- empty cells are not stored at all
  quantity_atom_id    FK quantity_atoms (nullable)
  date_atom_id        FK date_atoms (nullable)
  value_label_id      FK labels (nullable)    -- a string value: name / status / code
  footnote_marker     text (nullable)
  UNIQUE (table_id, row_index, col_index)

quantity_atoms                         -- a measurable value; never resolved
  id                  uuid PK
  run_id              FK extraction_runs
  raw_text            text             -- "$12,500,000" / "(93,750)" / "12,5%"
  value               numeric
  unit                text (nullable)  -- USD | EUR | ratio | shares | …
  normalization_version text           -- version of the lexical normalizer that produced it

date_atoms                             -- a calendar value; never resolved
  id                  uuid PK
  run_id              FK extraction_runs
  raw_text            text             -- "3/15/24" / "June 30, 2024"
  value               date
  normalization_version text

labels
  id                  uuid PK
  run_id              FK extraction_runs
  kind                enum: column_header | row_header | cell_text | kv_key | kv_value | narrative
                      -- structural provenance ONLY, never a role; role is derived from FK topology
  surface_text        text
  normalized_text     text             -- lowercased, whitespace/punct-normalized
  label_key           text             -- cross-document identity: (template_signature, axis, header_path) for table labels; lexical key otherwise
```

### Rules

- **Provenance chain**: `payload (atom | value label) → cell → axes → table → page → parse →
  source`. Every hop has a bbox except the payload itself (the cell owns it) and descriptor labels
  (the axis owns it).
- **Label per occurrence.** Each structural occurrence mints its own label row — the same header
  text in two tables is two labels (each with its own `label_key`). Dedup happens at resolvement
  time (identical `label_key`/normalized form hit the same precedent) and in the review UI
  (grouped display), never by sharing rows.
- **Normalization is lexical only** (`$5MM → 5_000_000 USD`, `(93,750) → -93750`,
  `3/15/24 → 2024-03-15`). Interpretation — what period a value covers, what a label means — is
  Stage 6+. `normalization_version` keeps old atoms explainable after normalizer changes.
- **Roles are never assigned here.** Whether an axis is "the metric axis" or a label is a date is
  a *derived* fact of Stage 6 resolvement. Stage 3 stores geometry and text, nothing semantic.
- Deterministic IDs within a run (e.g. UUIDv5 over `run_id + structural path`) so re-runs are
  diffable.

**Emits `run.structured`** → triggers Stage 4.

## Stage 4 — Non-table extraction

Values that live *outside* tables: title-block key-value pairs ("Investor: Jane Doe Family
Trust"), dates and amounts in prose ("the Fund called $2.0M from limited partners"), standalone
mentions ("statement for Gleneagles B.V."). This stage reuses the Stage 3 payload objects
wholesale — **no new value classes**. The only new object is the carrier.

### The one new object: the freestanding value

A KV pair and a narrative value are the same thing structurally: a value occurrence outside a
grid, with an *optional* key. A title-block key ("Investor:") and a prose governing phrase ("net
asset value stood at") are both descriptor labels for the value next to them; a bare name mention
simply has no key. So one carrier class covers all of it:

```
freestanding_values
  id                  uuid PK
  run_id              FK extraction_runs
  page_id             FK pages
  key_label_id        FK labels (nullable)   -- kind kv_key | narrative; null = keyless mention
  key_bbox            numeric[4] (nullable)
  raw_text            text                   -- the value's surface text, verbatim
  bbox                numeric[4]
  -- exactly ONE payload ref, same rule as table_cells (CHECK constraint)
  quantity_atom_id    FK quantity_atoms (nullable)
  date_atom_id        FK date_atoms (nullable)
  value_label_id      FK labels (nullable)
  extractor_version   text                   -- prompt + model version of the extraction pass
  context_snippet     text (nullable)        -- surrounding sentence, for extraction audit
```

- The parallel to `table_cells` is deliberate: cell ↔ freestanding value are the only two carrier
  classes in the system, and they hold payloads identically (one atom or one value label).
- Descriptor/value label semantics carry over unchanged: `key_label_id` is a descriptor slot,
  `value_label_id` a value slot.

### Extraction discipline

- **One LLM pass.** A single constrained LLM pass over the artifact's per-page markdown (tables
  stripped — those are Stage 3's) extracts all non-table values: KV pairs, prose values, keyless
  mentions. One mechanism, one prompt version to audit (`extractor_version`).
- **Verbatim guard.** Every extracted span must appear **verbatim** in the parse artifact;
  anything else is rejected. `raw_text` is the proof, `bbox` is recovered by grounding the span
  back to the artifact geometry.
- **LLM output is extraction, not interpretation.** The LLM may find "capital called: $2.0M" in
  prose; it may not decide what "capital called" means — the key becomes a label like any other
  and waits for Stage 6.
- **No dedup here.** A number appearing in both a cell and a sentence yields both carriers;
  deduplication/corroboration happens at proposal computation (same value+slots+period → one
  proposal, two provenance refs), keeping this stage high-recall and simple.

**Emits `run.values_extracted`** → triggers Stage 5.

## Stage 5 — Anchoring (decisions provisional)

Anchors are **document-level role assignments**: "this freestanding value is the document's
investor", "this date is the period end". They exist so proposal computation (Stage 7) can fill
slots a cell doesn't carry — the distribution on page 4 knows its investor because the document
does. Stage 4 already *extracted* everything; Stage 5 only **selects and links** among it.

### Storage

```
anchors
  id                  uuid PK
  run_id              FK extraction_runs
  role                enum: investor | fund | period_end | document_date | payment_date | inception_date
  value_id            FK freestanding_values   -- the selected occurrence; NEVER a copied payload
  method              enum: llm | deterministic | human
  selector_version    text                     -- prompt/model or rule version that selected it
  confidence          numeric
  rationale           text (nullable)          -- why this occurrence was selected (audit)
  UNIQUE (run_id, role)
```

### Rules

- **An anchor is a pointer, never a copy.** Its payload is read through the freestanding value it
  points at: a value label for entity roles (investor, fund), a date atom for date roles. There is
  no `resolved_entity_id` on the anchor — entity resolution belongs to the value label's
  resolvement (Stage 6), so anchor selection and entity resolution stay independently swappable.
- **Selection is candidate-only.** The selector (LLM or rule) may only pick among existing
  Stage 4 rows; it can never introduce a new value. This is what makes anchor assignment a
  modular, replaceable strategy: swap the selector, re-run Stage 5, nothing upstream or downstream
  changes shape.
- Competing occurrences (two dates that could be the period end) are not stored on the anchor —
  they remain ordinary freestanding values; `rationale` records why the winner won. Changing an
  anchor in review re-points `value_id`, which is exactly one auditable row.

**Emits `run.anchored`** → triggers Stage 6.

## Stage 6 — Label resolvement

Every label — descriptor or value, table or freestanding — gets exactly one resolvement per run,
through one machine. "What still needs mapping" is precisely *labels without a resolvement*.

### The universal target

A resolvement (and a precedent) always asserts one thing: a **target**. The target's kind is what
makes the system universal — one machine, one audit shape, every kind of meaning:

| target_kind | target payload | typical label |
|---|---|---|
| `canonical_metric` | registry canonical id | metric header, KV key like "Capital Called" |
| `entity` | entity id (or a mint request) | fund/investor name — value label or entity-axis header |
| `period` | period strategy (`QUARTER@period_end`, literal date, …) | date-axis header, "Period Ended" key |
| `carrier` | `value` \| `share` | non-semantic header (`Bedrag`, `%`) — resolved but roleless |
| `literal` | — (the surface string is the value) | IBAN, account number, status code |

### The resolve ladder (per label)

```
1. PRECEDENT — exact match, deterministic, auto-accepted
   a. structural: label_key (template_signature, axis, header_path)
   b. lexical: normalized_text within scope chain (template → doc_type → org → global)
2. CANDIDATE GENERATION — mechanical, non-LLM: registry similarity (metrics),
   blocking + similarity vs entities + aliases, period/carrier recognizers
3. REASONING — LLM picks among the enumerated candidates only (or abstains);
   result is PROPOSED, prefilled in review, never auto-accepted
4. HUMAN — confirms / overrides in review; confirmation writes a precedent (the flywheel)
```

### Storage

```
label_resolvements                      -- one row per label per run
  id                  uuid PK
  run_id              FK extraction_runs
  label_id            FK labels UNIQUE  -- (labels are per-run, so label_id alone is the key)
  status              enum: auto | proposed | confirmed | rejected
  target_kind         enum: canonical_metric | entity | period | carrier | literal
  target_ref          text (nullable)   -- canonical id / entity id / period strategy / carrier role
  mint_request        jsonb (nullable)  -- {display_name, entity_type} when proposing a NEW entity
  method              enum: precedent_structural | precedent_lexical | llm | human
  precedent_id        FK precedents (nullable)   -- the precedent that fired (method = precedent_*)
  confidence          numeric
  rationale           text (nullable)   -- LLM reasoning / human note — the audit "why"
  resolver_version    text
  created_at          timestamptz

resolvement_candidates                  -- the enumerated options the decision was made among
  id                  uuid PK
  resolvement_id      FK label_resolvements
  rank                int
  target_kind         enum (as above)
  target_ref          text
  score               numeric
  source              enum: precedent_knn | registry_similarity | entity_blocking | recognizer

precedents                              -- THE universal precedent store
  id                  uuid PK
  organization_id     FK
  match_kind          enum: structural | lexical
  match_key           text              -- label_key (structural) | normalized_text (lexical)
  scope               enum: template | doc_type | org | global
  scope_ref           text (nullable)   -- template signature / doc type slug
  target_kind         enum (as above)
  target_ref          text
  created_from        FK label_resolvements   -- the confirmed resolvement that minted it
  created_by          FK member
  match_count         int
  last_used_at        timestamptz
  revoked_at          timestamptz (nullable)  -- precedents are never deleted, only revoked
```

### Rules

- **Exact precedent hits auto-accept** (`status = auto`) — a lookup is not a decision. Everything
  from the LLM is `proposed` and prefilled in review; only a human moves it to `confirmed`.
- **Provenance of every mapping**: `method` says how, `precedent_id` or
  `resolvement_candidates` + `rationale` say why, `resolver_version` says with what. A confirmed
  mapping's precedent points back at the resolvement that minted it — the flywheel is walkable in
  both directions.
- **One store for metrics *and* entities.** An entity alias is just a lexical precedent with
  `target_kind = entity`. No parallel alias table.
- **Precedents are append-only**: corrections revoke (timestamp) and re-mint, so historical
  resolvements keep pointing at the precedent text that actually fired.
- Confirming a resolvement whose label recurs (same `label_key` / normalized form) re-resolves the
  siblings in the same run automatically — the grouped-review UX falls out of the data model.

**Emits `run.resolved`** → triggers Stage 7.

## Stage 7 — Proposals

A proposal is **computed, never authored**: `proposal = f(carrier payload, resolvements, registry,
anchors)`. Stage 7 runs `f()` over every carrier in the run and **persists the result plus its
input lineage** — the persisted body is a recomputable projection (change an anchor or a mapping →
recompute in place), and the input lineage is what makes every proposal auditable.

### Two proposal kinds

- **`entity`** — a value label resolved to a mint request (new entity) or a low-confidence link
  that needs confirmation. Approval creates (or links) the entity row.
- **`metric`** — a quantity/date atom whose metric label resolved to a canonical. The registry
  entry for that canonical supplies class (`entity_metric` / `position_metric` / `transaction`),
  arity, required slots, sign convention, and period kind; slots are filled **by entity type**
  from the composition chain (own axis labels → anchors); the time coordinate is composed from the
  date-axis resolvement / date anchors + period kind. Class is always derived, never stored as a
  decision.

Relationship edges (`invests_in`) are **not** proposals in v2: a position metric already names
both endpoints, so the edge is upserted as a side effect at commit. Explicit edge proposals
(GP_of, owns) are deferred.

### Storage

```
proposals                               -- recomputable projection of f(); one row per computed proposal
  id                  uuid PK
  run_id              FK extraction_runs
  identity            text              -- stable hash over payload lineage (carrier + slot lineage);
                                        -- survives recompute + reclassification, keys the decision log
  kind                enum: entity | metric
  body                jsonb             -- the computed proposal: canonical, class, slots, when, value, sign
  body_hash           text
  status              enum: ready | blocked | flagged   -- blocked: missing required slot (loud, never downgraded)
  flags               text[]            -- missing_required_slot:investor, expected_instant_got_interval, …
  compute_version     text
  computed_at         timestamptz
  UNIQUE (run_id, identity)

proposal_inputs                         -- the explicit computation audit: exactly what f() consumed
  proposal_id         FK proposals
  input_kind          enum: cell | freestanding_value | resolvement | anchor | registry_version
  input_ref           text              -- FK value (or registry version string)
  role                text              -- value | metric_label | entity_slot:investor | period | …
```

`body` is the one pragmatic JSONB beyond the parse artifact: it is a *disposable projection*
(rebuildable from `proposal_inputs` at any time), not stored truth — the relational-first rule
applies to durable facts, and the durable facts here are the inputs.

### Rules

- **Recompute, not sync.** Editing a resolvement or anchor re-runs `f()` for affected proposals;
  same `identity` → same row refreshed in place. Nothing per-cell to keep consistent.
- **Loud failure**: a position metric missing its investor slot is `blocked` with a flag — never
  silently reshaped into an entity metric.
- Deduplication/corroboration happens here: identical `(canonical, slots, when, value)` from a
  cell and a sentence merge into one proposal with two `proposal_inputs` value rows.
- Every atom/value label must end up referenced by some proposal or be explicitly marked dropped
  with a reason (the coverage invariant); the residue is a run-level report, not silence.

**Emits `run.proposed`** → review is ready.

## Stage 8 — Commit

Approval turns a proposal into truth. Decisions are **append-only** and keyed on the proposal's
stable `identity` (not its row id), so they survive recompute and even re-extraction.

### Storage

```
proposal_decisions                      -- append-only; the durable human/agent record
  id                  uuid PK
  organization_id     FK
  proposal_identity   text              -- joins to proposals.identity
  kind                enum: approve | reject | edit
  actor               FK member (or agent identifier)
  body_hash_at_decision text            -- what the human actually saw and approved
  edited_body         jsonb (nullable)  -- present on edit
  created_at          timestamptz
```

### Rules

- **Approve → commit**: writes the truth rows (entity / entity_metric / position_metric /
  transaction tables) inside one transaction with the decision row; each truth row carries
  `decision_id`, so truth → decision → proposal → inputs → cell → parse → source is one unbroken
  provenance walk.
- **Staleness is derived**: if a decided proposal's recomputed `body_hash` no longer matches
  `body_hash_at_decision`, it is flagged stale and re-queued — an approval is never silently
  re-applied to a body the human didn't see.
- Reverts append a decision and reverse the truth write; nothing is deleted.

**Emits `proposal.committed`** per commit (consumers: portfolio recompute, notifications).

## The review screen (per source)

One screen per source, **one queue**, reading the relational substrate directly. The queue mixes
item types (filterable), ordered so that upstream decisions come first — confirming a label
mapping collapses the cell-level items beneath it:

1. **Label mappings** — labels with `proposed` / missing resolvements, grouped by normalized form,
   prefilled with the LLM's pick + candidates + rationale. Confirming writes the resolvement,
   mints the precedent, and recomputes affected proposals live.
2. **Anchors** — the run's anchor rows with their pointed-at values (and page/bbox highlight);
   re-pointing an anchor recomputes downstream proposals.
3. **Freestanding values** — non-table values with their (optional) key labels and payloads;
   mostly confirmation reading, escalates into a label-mapping item when a key needs resolvement.
4. **Bulk approve tables** — the grid in document geometry, axis labels shown as chips
   (resolvement state), cells showing their computed proposals; approve-all commits every `ready`
   proposal in the table.

---

## Decision log

Resolved decisions are folded into the stage text above; this is the running record.

| # | Question | Decision |
|---|---|---|
| 1 | Bundle splitting | Dropped from v2 |
| 2 | Duplicate upload | New `sources` row sharing the R2 object; no re-parse |
| 3 | Active parse | Explicit `active_parse_id` pointer on `sources` |
| 4 | Artifact location | JSONB in Postgres now; R2 later if size demands |
| 5 | Run spine | Thin `extraction_runs` keyed to `parse_id` |
| 6 | Pages | First-class `pages` table |
| 7 | Axes model | Unified `table_axes` class for rows + columns + headers |
| 8 | Empty cells | Skipped; indices preserve grid geometry |
| 9 | Label dedupe | One label per structural occurrence; dedup via `label_key` at resolvement |
| 10 | Atom seam | Atoms = deterministically parsed values only (`quantity_atoms`, `date_atoms`, two tables). All strings in value positions are labels |
| 11 | Descriptor vs value labels | Role derived from FK topology (which slot references the label), never a column on the label |
| 12 | Label kind | Pure structural provenance; meaning (entity / literal / enum) is always a resolvement outcome |
| 13 | Stage 4 carrier | One `freestanding_values` class for KV pairs + narrative values + keyless mentions (nullable key) |
| 14 | Stage 4 extraction | Single LLM pass over per-page markdown (verbatim guard); no separate geometric KV miner |
| 15 | Stage 4/5 split | Stage 4 extracts everything; Stage 5 only selects + links anchors among it |
| 16 | Anchor shape | *(provisional default)* Thin pointer: role + FK to freestanding value; resolution stays on the label |
| 17 | Anchor roles | *(provisional default)* Small closed enum for v2 |
| 18 | Stage 4/5 LLM calls | *(provisional default)* Two separate calls — anchor selection independently swappable |
| 19 | Anchor gating | *(provisional default)* Proposals compute immediately; anchor edits repoint + recompute |
| 20 | Precedent store | One universal table; `target_kind` discriminates (entity aliases included) |
| 21 | Scope chain | template → doc_type → org → global, first hit wins |
| 22 | Entity minting | No entity rows before proposal approval; mint request lives on the resolvement |
| 23 | Proposal model | Computed body (jsonb projection) + explicit `proposal_inputs` lineage + append-only decisions keyed on stable identity |
| 24 | Edge proposals | Deferred from v2; `invests_in` upserted at commit as side effect |
| 25 | Approve = commit | One action; truth rows + decision row in one transaction |
| 26 | Review screen | One queue per source mixing item types; "freestanding values" replaces "KV pairs" terminology |
