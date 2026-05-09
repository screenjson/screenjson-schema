---
name: screenjson-schema
description: "Workspace instructions for screenjson-schema: The canonical JSON Schema specification for ScreenJSON screenplay documents."
---

# screenjson-schema Workspace Instructions

## Project Overview

**screenjson-schema** is the authoritative JSON Schema (Draft 2026-04) specification with Animation extensions for the ScreenJSON format. It defines:
- Complete screenplay document structure
- All element types and relationships
- Data validation rules and constraints
- Multi-language support (BCP 47 tags)
- Production metadata, revisions, and access control

This schema is embedded in `screenjson-cli` for validation and serves as the single source of truth for all ScreenJSON implementations.

## Architecture

```
screenjson-schema/
├── README.md
├── src/
│   └── json-schema/
│       ├── schema.json      # Canonical JSON Schema (Draft 2026-04)
│       └── schema.yaml      # Human-editable YAML equivalent
│
└── elasticsearch/           # Integration examples
    ├── lab.json             # Elasticsearch lab settings (queries/examples)
    ├── rag_chunks.json      # Vector chunk mapping for RAG
    ├── rag_lines.json       # Line-level vector embedding mapping
    └── search.json          # Full-text search mapping
```

## Schema Structure

### Root Document
The `Document` is the top-level object containing:

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Document identifier (RFC 4122) |
| `version` | string | ScreenJSON spec version (semver: e.g., "1.0.0") |
| `title` | Text | Script title (multi-language) |
| `lang` | string | Primary language (BCP 47, e.g., "en", "fr-CA") |
| `charset` | string | Character encoding (typically "utf-8") |
| `dir` | string | Text direction ("ltr" or "rtl") |
| `authors` | Author[] | Original screenwriter(s) |
| `contributors` | Contributor[] | Editors, directors, script doctors, etc. |
| `characters` | Character[] | All characters appearing in screenplay |
| `sources` | Source[] | Source works (novel, play, etc.) if adapted |
| `registrations` | Registration[] | WGA or other registration records |
| `revisions` | Revision[] | Global revision history |
| `genre` | string[] | Classification tags (e.g., ["Drama", "Comedy"]) |
| `themes` | string[] | Thematic tags (e.g., ["redemption", "family"]) |
| `logline` | Text | One-sentence summary (multi-language) |
| `document` | ScreenplayDocument | The actual screenplay content |
| `analysis` | Analysis | Optional derived data (embeddings, summaries, passages) |

### Document.document Object (Screenplay)
Contains the actual screenplay with:

| Field | Type | Purpose |
|-------|------|---------|
| `cover` | Cover | Title page metadata |
| `layout` | Layout | Optional rendering rules (styles, templates, format guides) |
| `bookmarks` | Bookmark[] | Named shortcuts to specific elements |
| `scenes` | Scene[] | Ordered scenes (the screenplay body) |

### Scenes
Each scene contains:

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Scene identifier |
| `heading` | Slugline | Scene heading (INT/EXT, setting, time-of-day) |
| `body` | Element[] | Ordered screenplay elements (action, dialogue, etc.) |
| `cast` | UUID[] | Character UUIDs appearing in scene |
| `props` | string[] | Properties used in scene |
| `wardrobe` | string[] | Costumes/wardrobe in scene |
| `vfx` | string[] | Visual effects |
| `sfx` | string[] | Sound effects |
| `sounds` | string[] | Sound design notes |
| `animals` | string[] | Animals/creatures in scene |
| `locations` | string[] | Physical locations (for production) |
| `authors` | UUID[] | Authors of scene |
| `notes` | Note[] | Production notes/comments |
| `revisions` | Revision[] | Scene-level revision history |

### Elements (Discriminated Union)
Scene bodies contain typed elements via the `type` discriminator:

| Type | Fields | Purpose |
|------|--------|---------|
| `action` | `text` | Scene description, stage direction |
| `character` | `character` (UUID), `display` | Character name cue before dialogue |
| `dialogue` | `text`, `origin` (V.O./O.S.), `dual` | Spoken lines |
| `parenthetical` | `text` | Actor direction within dialogue |
| `transition` | `text` | CUT TO, DISSOLVE TO, etc. |
| `shot` | `text`, `fov`, `perspective` | Camera direction + optional lens/angle info |
| `general` | `text` | Catch-all for non-standard elements |

Every element has:
```json
{
  "id": "uuid",
  "type": "action|character|dialogue|parenthetical|transition|shot|general",
  "authors": ["uuid1", "uuid2"],
  "notes": [{ "text": "...", "position": [...] }],
  "revisions": [{ "code": "blue", "range": [...] }],
  "locked": false,
  "omit": false,
  "access": { "allowRoles": [...], "denyRoles": [...] },
  "encrypt": { "algorithm": "aes-256-gcm", "nonce": "..." }
}
```

### Characters (First-Class Objects)
Characters are defined once in root `characters` array:

```json
{
  "id": "uuid",
  "name": "John Smith",
  "slug": "john-smith",           // URL-safe identifier
  "aliases": ["JOHN", "J.D."],    // Alternative names in script
  "desc": {                        // Multi-language character description
    "en": "A detective investigating..."
  },
  "traits": ["morally-ambiguous", "witty"]  // Searchable tags
}
```

**Why this matters**:
- References throughout use **UUID, not name**
- Avoids "JOHN" vs "JOHN (CONT'D)" vs "JOHNNY" ambiguity
- Character metadata changes don't break references
- Enables reliable entity resolution for analytics/search

### Revisions
Track screenplay changes with industry-standard color codes:

```json
{
  "code": "blue",  // "white", "blue", "pink", "yellow", "green", "goldenrod", "orange", "purple", "brown", "red"
  "range": {
    "startLine": 121,
    "endLine": 125
  },
  "date": "2025-01-15T10:30:00Z"
}
```

### Text (Multi-Language Support)
Any text field supports multiple languages via BCP 47 tags:

```json
{
  "title": {
    "en": "The Great Escape",
    "fr": "La Grande Évasion",
    "es": "La Gran Fuga",
    "ja": "大脱獄"
  }
}
```

Language tag validation: `^[A-Za-z]{2,3}(?:-[A-Za-z0-9]{2,8})*$`

### Access Control & Encryption
Optional per-element access control:

```json
{
  "access": {
    "allowRoles": ["writer", "director"],
    "denyRoles": ["actor"]
  },
  "encrypt": {
    "algorithm": "aes-256-gcm",
    "nonce": "...",
    "key": "..." // or reference to external key store
  }
}
```

## Core Reusable Definitions ($defs)

The schema uses `$defs` for reusable type patterns:

| Definition | Pattern | Used For |
|-----------|---------|----------|
| `uuid` | RFC 4122 format | All IDs: document, scene, element, character, author |
| `slug` | `^[a-z0-9]+(?:-[a-z0-9]+)*$`, 3–50 chars | Character/scene slugs, URL-safe identifiers |
| `text` | Object with language keys | Multi-language text (title, description, logline) |
| `meta` | Pattern-matched key/value pairs | Metadata (e.g., registration codes) |
| `languageTag` | BCP 47 regex | Language identification (language, region, script) |
| `revisionCode` | Enum | "white", "blue", "pink", "yellow", etc. |
| `elementType` | Union of type strings | Discriminator for Element objects |

## Elasticsearch Integration

### Mappings
The `elasticsearch/` folder contains integration examples:

| File | Purpose |
|------|---------|
| [rag_chunks.json](elasticsearch/rag_chunks.json) | Vector chunk embeddings for RAG search (screenplay passages) |
| [rag_lines.json](elasticsearch/rag_lines.json) | Line-level embeddings for precise matching |
| [search.json](elasticsearch/search.json) | Full-text search mapping (dialogue, action, character names) |
| [lab.json](elasticsearch/lab.json) | Example queries and search labs |

**Use case**: Index ScreenJSON documents in Elasticsearch for:
- Full-text search across screenplay content
- Vector similarity search (find similar scenes/characters)
- Metadata aggregations (films per director, characters per writer)
- Production workflows (prop/wardrobe/VFX lists)

## Development Conventions

### Schema Authoring
| Aspect | Convention |
|--------|-----------|
| **Format** | JSON Schema Draft 2026-04 (primary); YAML for human editing |
| **Reusable types** | Define in `$defs`, reference with `$ref` |
| **Required fields** | Explicit `required` array at object level |
| **Validation pattern** | Use `pattern` for string formats (regex) |
| **Enums** | Use `enum` array for fixed-value fields |
| **IDs** | Always UUID (RFC 4122), never bare strings |
| **Timestamps** | ISO 8601 (`format: date-time`) |
| **Versioning** | Document version follows semantic versioning |
| **Comments** | Use `description` fields (not JSON comments) |

### Sync JSON ↔ YAML
The schema exists in both formats:
- **schema.json** (canonical, tooling source)
- **schema.yaml** (human-readable for review/editing)

Keep both in sync during updates. Use tools to convert between formats.

## Common Pitfalls

### 1. Character References Must Use UUID
❌ Wrong:
```json
{
  "type": "dialogue",
  "character": "JOHN"
}
```

✓ Correct:
```json
{
  "type": "character",
  "character": "550e8400-e29b-41d4-a716-446655440000"
}
```

### 2. Multi-language Text Is an Object
❌ Wrong:
```json
{
  "title": "The Great Escape"
}
```

✓ Correct:
```json
{
  "title": {
    "en": "The Great Escape"
  }
}
```

### 3. Element Type Discriminator Is Required
Every element must have a `type` field that determines other valid fields.

### 4. Revisions Use Industry-Standard Color Codes
Valid codes: `white`, `blue`, `pink`, `yellow`, `green`, `goldenrod`, `orange`, `purple`, `brown`, `red`

### 5. Language Tags Must Be Valid BCP 47
❌ Wrong: `{"EN": "..."}`, `{"en_US": "..."}`  
✓ Correct: `{"en": "..."}`, `{"en-US": "..."}`

## Links & Resources

- [ScreenJSON Specification](https://screenjson.com)
- [JSON Schema Draft 2026-04](https://json-schema.org/draft/2026-04/json-schema-core.html)
- [BCP 47 Language Tags](https://tools.ietf.org/html/bcp47)
- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [screenjson-cli](../screenjson-cli/) — Reference implementation using this schema
