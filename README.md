
# ScreenJSON Schema

ScreenJSON is a data serialisation format for screenplays.

This repository contains:

- The canonical JSON Schema (Draft 2020-12)
- A YAML copy of the schema (for easier human editing/review)
- Elasticsearch index settings/mappings examples for storing and searching ScreenJSON documents

## ScreenJSON at a glance

A ScreenJSON file is a single JSON object that includes:

* Document metadata (title, language, authors, etc.)
* Entity indexes (characters, contributors, colors, etc.)
* The screenplay itself as a list of scenes
* Optional presentation rules (templates/styles/header/footer) for renderers

The schema is designed to support:

* CLI viewers (pretty-print, search, filter, export)
* UI viewers/editors (structured rendering, character navigation, bookmarks)
* Indexing pipelines (denormalised `text_search` fields for Elasticsearch)


## Schema versions

The schema is Draft 2020-12:

* `$schema`: `https://json-schema.org/draft/2020-12/schema`

Schema stability rules (recommended):

* Patch changes (`x.y.z`) may fix validation bugs and tighten constraints without changing meaning.
* Minor changes (`x.y`) may add optional fields.
* Major changes (`x`) may introduce breaking changes (renames, removals, semantics).

If you include a `spec_version` field in your ScreenJSON documents, keep it sync'd with the released schema tag/version in this repo.

## Validation

### Validate a ScreenJSON file

Use any JSON Schema validator that supports Draft 2020-12.

Example with `ajv`:

```bash
npm i -g ajv-cli
ajv validate \
  -s src/json-schema/schema.json \
  -d your-screenplay.json \
  --strict=false
```

Notes:

* The schema enforces structure and types, but not full “relational integrity”.
  Example: it can validate that `character_id` is a UUID, but it cannot guarantee that
  the ID exists in `characters[]` without an additional semantic validation pass.
* Most implementations should add a lightweight semantic validator for:

  * foreign key checks (character_id exists)
  * bookmark targets exist (scene_id/element_id exist)
  * duplicates (same UUID appearing in two different entity lists, etc.)


## Elasticsearch

The `elasticsearch/` directory contains example index templates for storing ScreenJSON in a searchable form.

### Recommended approach

1. Store the full ScreenJSON document as `_source` (or in your primary DB)
2. Generate denormalised search fields for indexing, such as:

* `document.text_search` (whole-script text)
* `document.scenes[].body[].content_text` (element-level text)
* `title.en`, character names, tags, etc.

This keeps ScreenJSON clean while enabling fast full-text search and filtering.

### Files

* `elasticsearch/search.json`
  Production-oriented analyzer strategy:

  * edge n-grams at index time (autocomplete)
  * normal analyzer at query time (better relevance)

* `elasticsearch/lab.json`
  Testbed for experimenting with analyzers/filters before promoting changes.


## Editing workflow

If you edit `schema.yaml`, ensure `schema.json` stays in sync.

Typical workflow:

1. Make changes in `src/json-schema/schema.yaml`
2. Convert to JSON (or update JSON directly)
3. Run validation tests against sample ScreenJSON documents
4. Commit both schema files together


## Contributing

Issues and PRs welcome.

When proposing schema changes, please include:

* The motivating use case
* A minimal example JSON instance demonstrating the change
* Whether the change is breaking or additive
* Any impact on indexing (Elasticsearch) or rendering (CLI/UI)

