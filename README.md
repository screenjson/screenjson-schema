# ScreenJSON Schema

ScreenJSON is a data serialisation format for screenplays. It captures the full structure of a script—scenes, dialogue, action, characters, metadata—in a single JSON document that can be validated, indexed, searched, and rendered by any compliant tool.

This repository contains:

- The canonical JSON Schema (Draft 2026-01)
- A YAML copy of the schema (for easier human editing/review)
- Elasticsearch index settings/mappings examples for storing and searching ScreenJSON documents

## Why ScreenJSON?

Screenplays have been trapped in proprietary formats (Final Draft's `.fdx`, Celtx, Highland, etc.) or lossy plain-text formats (Fountain) that discard structural information. ScreenJSON provides:

- **Structural fidelity**: Every element type (action, dialogue, parenthetical, transition, shot) is explicitly typed, not inferred from formatting.
- **Entity tracking**: Characters, authors, and contributors are first-class objects with UUIDs, enabling reliable cross-referencing and change tracking.
- **Internationalisation**: All text fields support multiple languages via BCP 47 keys, with RTL and charset support.
- **Production metadata**: Scene numbers, revision tracking (with industry-standard colour codes), registration info, and access control roles.
- **Separation of content and presentation**: The screenplay data is layout-neutral; optional `styles` and `templates` let renderers apply formatting without polluting the source.
- **AI/ML readiness**: First-class support for vector embeddings, chunking, and hierarchical summaries for retrieval-augmented generation (RAG) workflows.

## Document Structure

A ScreenJSON file is a single JSON object containing:

### Root Level

| Field | Purpose |
|-------|---------|
| `id` | Document UUID |
| `version` | ScreenJSON spec version (semver) |
| `title` | Script title (multi-language) |
| `lang`, `charset`, `dir` | Primary language, encoding, text direction |
| `authors` | Original writers |
| `contributors` | Editors, directors, script doctors, etc. |
| `characters` | All characters in the screenplay |
| `sources` | Source works (novel, play, etc.) if adapted |
| `registrations` | WGA or other registration records |
| `revisions` | Global revision history |
| `genre`, `themes` | Document-level classification tags |
| `logline` | One-sentence summary |
| `document` | The screenplay content itself |
| `rag` | Optional AI/ML support data |

### The `document` Object

Contains the actual screenplay:

- **`cover`**: Title page metadata (title, authors shown, sources credited)
- **`layout`**: Optional rendering rules (header/footer ribbons, revision status, styles, templates, format guides)
- **`bookmarks`**: Named shortcuts to specific lines
- **`scenes`**: The ordered list of scenes

### Scenes

Each scene contains:

- **`heading`**: The slugline (INT/EXT, setting, time-of-day, modifiers)
- **`body`**: Ordered list of lines (action, character cues, dialogue, parentheticals, transitions, shots, general)
- **`cast`**: Character UUIDs appearing in the scene
- **`beats`**, **`tone`**: Structural and emotional markers
- **Production tags**: `props`, `wardrobe`, `sfx`, `vfx`, `sounds`, `animals`, `locations`, etc.

### Lines

Lines are a discriminated union with a `type` field:

| Type | Description |
|------|-------------|
| `action` | Scene description, stage direction |
| `character` | Character cue (the name before dialogue) |
| `dialogue` | Spoken lines, with optional `origin` (V.O., O.S., etc.) and `dual` flag |
| `parenthetical` | Actor direction within dialogue |
| `transition` | CUT TO, DISSOLVE TO, etc. |
| `shot` | Camera direction (with optional `fov` and `perspective` for pre-vis) |
| `general` | Catch-all for non-standard elements |

Every line carries:

- `id`: UUID for stable referencing
- `authors`: Who wrote this line
- `notes`: Attached comments with optional text highlighting
- `revisions`: Line-level revision history
- `locked`, `omit`: Production flags
- `access`: Role-based visibility control
- `encrypt`: Optional per-line encryption parameters

## Characters

Characters are defined once in the root `characters` array and referenced by UUID throughout. Each character has:

- `name`: Canonical display name
- `slug`: URL-safe identifier
- `aliases`: Alternative names used in the script
- `desc`: Character description (multi-language)
- `traits`: Searchable attributes (protagonist, antagonist, comic-relief, etc.)

This avoids the "JOHN" vs "JOHN (CONT'D)" vs "JOHNNY" problem—all refer to one character object.

## Revisions

ScreenJSON supports the industry-standard revision colour system:

- white → blue → pink → yellow → green → goldenrod → buff → salmon → cherry

Revisions can be tracked at document level, scene level, or individual line level. Each revision records the authors, a label, timestamp, and parent revision for history traversal.

## Embeddings and RAG Support

The optional `rag` object provides first-class support for AI/ML workflows:

### Embeddings

Vector embeddings are stored in a centralised map keyed by the UUID of the embedded object:
```
rag.embeddings["<scene-uuid>"] = [
  { model, dims, vector, src, lang, tokens, created, ... }
]
```

Each embedding records:

- **`model`**: The embedding model used (e.g., `text-embedding-3-small`, `voyage-3`)
- **`dims`**: Vector dimensionality (must match `vector` length)
- **`vector`**: The float array
- **`src`**: What was embedded (`text`, `name`, `desc`, `heading`, `composite`)
- **`lang`**: Language of the source text
- **`tokens`**: Token count (for cost tracking and re-embedding decisions)
- **`created`**: Timestamp for staleness detection

This structure is compatible with Pinecone, Weaviate, and Chroma. Export scripts can flatten the UUID-keyed map into the vector DB's native format.

### Chunks

For retrieval, screenplays need to be segmented into chunks that fit model context windows. The `rag.chunks` array stores pre-computed chunks:

- **`scene`**: Which scene this chunk belongs to
- **`lines`**: Ordered line UUIDs included in the chunk
- **`text`**: Flattened text content
- **`tokens`**: Approximate token count
- **`overlap`**: Tokens shared with adjacent chunks
- **`prev`**, **`next`**: Linked-list navigation

Chunking dialogue differently from action (e.g., keeping character exchanges intact) is a common strategy—the `lines` array lets you reconstruct context.

### Synopses

Hierarchical summaries support multi-level retrieval:

- **`scope`**: `document`, `act`, `sequence`, or `scene`
- **`target`**: UUID of the target scene (null for document-level)
- **`range`**: For act/sequence scope, the `[start, end]` scene indices
- **`text`**: The summary text
- **`generated`**: Whether this was LLM-generated
- **`model`**: Which model generated it

This enables a retrieval strategy that first searches synopses for relevant acts/sequences, then drills down into scene chunks.

### Configuration

The `rag.config` object records chunking and embedding parameters:

- `chunk_size`, `chunk_overlap`: Token counts
- `embed_model`, `embed_dims`: Model identifier and output dimensions
- `tokenizer`: Which tokenizer was used (important for reproducibility)

This metadata lets you detect when re-embedding is needed (model change, config change, content change).

### Design Philosophy

The `rag` object is:

- **Optional**: Renderers and traditional tools can ignore it entirely.
- **Strippable**: Export a "clean" screenplay by omitting the `rag` key.
- **Centralised**: Embeddings live in one place, not scattered across scenes and lines, making bulk operations straightforward.
- **Reproducible**: Configuration is stored alongside the vectors, so you know how they were generated.

## Schema Versions

The schema uses JSON Schema Draft 2026-01:

- `$schema`: `https://json-schema.org/draft/2026-01/schema`

Versioning follows semver principles:

| Change Type | Version Bump | Examples |
|-------------|--------------|----------|
| Patch (`x.y.z`) | Bug fixes, constraint tightening | Fixing a regex, adding `maxLength` |
| Minor (`x.y`) | Additive changes | New optional fields, new enum values |
| Major (`x`) | Breaking changes | Renames, removals, semantic changes |

Include a `version` field in your ScreenJSON documents to track which schema version they target.

## Validation

Use any JSON Schema validator supporting Draft 2026-01.

**Important limitations**: JSON Schema validates structure and types, but not relational integrity. You should add a semantic validation pass for:

- Foreign key checks (does `character` UUID exist in `characters[]`?)
- Bookmark targets (do `scene` and `line` UUIDs exist?)
- Duplicate detection (same UUID in multiple places)
- Embedding consistency (`dims` matches `vector` length)

## Elasticsearch

The `elasticsearch/` directory contains index templates for full-text search.

**Recommended approach**:

1. Store the complete ScreenJSON document as `_source` (or in your primary database)
2. Generate denormalised fields for indexing:
   - `text_search`: Concatenated script text
   - `scene.text`: Per-scene searchable text
   - Character names, tags, themes, etc.

This keeps the source document clean while enabling fast search and filtering.

**Included files**:

- `elasticsearch/search.json`: Production-ready with edge n-grams for autocomplete
- `elasticsearch/lab.json`: Experimental analyzer configurations

## Editing Workflow

If editing `schema.yaml`, keep `schema.json` in sync:

1. Edit `src/json-schema/schema.yaml`
2. Convert to JSON (or update JSON directly)
3. Validate against sample documents
4. Commit both files together

## Contributing

Issues and PRs welcome.

When proposing changes, include:

- The motivating use case
- A minimal example JSON instance
- Whether the change is breaking or additive
- Impact on indexing (Elasticsearch) or rendering (CLI/UI)
- Impact on RAG workflows (if touching `rag`, `chunk`, `embedding`, etc.)