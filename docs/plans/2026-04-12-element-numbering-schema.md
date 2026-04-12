# Plan: Extend Element-Level Numbering to Action, Character, Dialogue - Schema

**Date**: April 12, 2026  
**Status**: In Progress  
**Scope**: screenjson-schema  
**Related**: 2026-04-12-element-numbering-cli.md

## Overview

This plan documents schema considerations for extending element numbering from Scene Headings to Action, Character, and Dialogue elements. The good news: **no schema changes are required** — the existing schema already supports `sceneNo` on all element types.

## Current Schema State

### Element Definition (Already Supports sceneNo)
The ScreenJSON schema element definition already includes `sceneNo` for all element types:

```json
{
  "sceneNo": {
    "description": "Optional source scene number for the element (e.g., '1', '1A', '110A/111B' for split scenes, 'I-1-A' for roman numerals).",
    "type": "string",
    "pattern": "^[A-Za-z0-9][A-Za-z0-9.\\/-]*$",
    "minLength": 1,
    "maxLength": 20
  }
}
```

### Element Types Covered
✅ All element types can have optional `sceneNo`:
- action
- character
- dialogue
- parenthetical
- transition
- shot
- general

## Schema Changes: NONE Required

**Reason**: The upstream schema (screenjson-schema) was designed with universal element numbering in mind. The `sceneNo` field was added to support:

1. **FDX Paragraph Numbers** — for all paragraph types
2. **Fountain numbering syntax** — for all element types
3. **Future numbering schemes** — extensible to any element

## Validation Coverage

The pattern `^[A-Za-z0-9][A-Za-z0-9.\\/-]*$` validates:
- Alphanumeric start (strict)
- Alphanumeric/dash/period/slash suffix (flexible)
- 1-20 character length limit
- All valid numbering formats: "42", "1A", "110A/111B", "I-1-A"

This pattern aligns with:
- FDX Paragraph.Number field format
- Fountain numbering syntax (#...#)
- Industry screenplay numbering conventions

## Implementation Notes for screenjson-cli

When implementing element numbering in screenjson-cli:

1. **No schema updates needed** — existing pattern handles all cases
2. **Validation ready** — use existing `ValidateSceneNumber()` and `NormalizeSceneNumber()` functions
3. **Pattern locked** — no breaking changes required for schema

## Elasticsearch Integration (Informational)

No changes needed to Elasticsearch mappings (elasticsearch/*.json files). The `sceneNo` field is already indexable as a keyword and can be:
- Filtered: Find all elements numbered "1A"
- Aggregated: Analyze scene structure by numbering scheme
- Included in full-text search

## Backward Compatibility

✅ Existing ScreenJSON documents without `sceneNo` validate (optional field)  
✅ No breaking changes to schema  
✅ No impact to other element fields

## Summary

**Schema Status**: ✅ Ready  
**Changes Required**: None  
**Implementation**: screenjson-cli only  
**Documentation**: Optional enhancement
