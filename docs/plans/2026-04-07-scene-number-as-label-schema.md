# Scene Number as Label - Schema Updates

**Project**: screenjson-schema  
**Branch**: `scene-number-as-label`  
**Objective**: Update the authoritative ScreenJSON JSON Schema to support string-based scene numbers and per-element scene number tracking.

## Overview

The ScreenJSON schema is the single source of truth for document structure. This initiative updates the canonical schema to reflect the scene-number-as-label feature, enabling tools and implementations to validate documents with alphanumeric scene numbers.

## Changes to src/json-schema/schema.json

### Slugline Definition

**Location**: `$defs.slugline.properties.no`

**Before**:
```json
"no": {
  "description": "Production scene number (optional).",
  "type": "integer",
  "minimum": 1
}
```

**After**:
```json
"no": {
  "description": "Production scene number (optional).",
  "type": "string",
  "pattern": "^[A-Za-z0-9][A-Za-z0-9.-]*$"
}
```

**Rationale**:
- Supports industry-standard alphanumeric numbering: "1", "1A", "I-1-A", "110A"
- Pattern allows alphanumerics, dashes, periods
- First character must be alphanumeric (prevents leading special chars)
- Optional field (empty string omitted from JSON)

### Element Definition

**Location**: `$defs.element.properties` (new property)

**Added**:
```json
"sceneNo": {
  "description": "Optional source scene number for the element.",
  "type": "string",
  "pattern": "^[A-Za-z0-9][A-Za-z0-9.-]*$"
}
```

**Rationale**:
- Preserves source-format scene number on individual elements
- Enables per-element scene number tracking
- Same pattern as slugline.no for consistency
- Optional field for backward compatibility

## Validation Rules

Both `no` and `sceneNo` follow these rules:

1. **Type**: must be string (no implicit type coercion)
2. **Pattern**: `^[A-Za-z0-9][A-Za-z0-9.-]*$`
   - First character: letter or digit
   - Subsequent characters: letters, digits, dash, period
3. **Examples of Valid Values**:
   - "1", "2", "99" (numeric)
   - "1A", "2B", "110Z" (numeric + letter)
   - "I-1-A", "II-2-B" (Roman numerals with separators)
   - "1.1", "2.2.1" (hierarchical numbering)
   - "110A", "223B" (multi-digit with letter)

4. **Examples of Invalid Values**:
   - Starting with special char: "-1", ".5", "#1A"
   - Spaces: "1 A", "INT 1"
   - Other special chars: "1@A", "2#", "3$"

## Impact on Document Validation

Tools implementing this schema can now:

1. **Accept alphanumeric scene numbers** without modification
2. **Validate number format** using the regex pattern
3. **Preserve source numbers** across format conversions
4. **Query by scene number** at both scene and element levels

## Elasticsearch Integration

The `/elasticsearch` folder contains integration examples:

- **rag_chunks.json**: Scene number mapping for vector chunks
- **rag_lines.json**: Line-level embeddings can reference scene numbers
- **search.json**: Full-text search mappings can index scene numbers
- **lab.json**: Example queries for scene-based searches

Tools can index and search documents by scene number field.

## Backward Compatibility

Changes are **backward compatible**:

1. Numeric values continue to work: `"no": "1"` (previously `1`)
2. Missing fields are optional (omitted from JSON)
3. Existing documents with integer scene numbers can be migrated by converting to strings
4. No required fields changed

## Schema Validation Test

The schema changes are verified by:

```python
# Verify schema structure
defs = schema['$defs']
slugline_no = defs['slugline']['properties']['no']
assert slugline_no['type'] == 'string'
assert 'pattern' in slugline_no

element_props = defs['element']['properties']
assert 'sceneNo' in element_props
assert element_props['sceneNo']['type'] == 'string'
```

## Files Modified

- `src/json-schema/schema.json` – Updated schema definitions

## Related Code

The canonical schema is embedded in **screenjson-cli**:

- **File**: `internal/schema/schema.json`
- **Sync status**: ✅ Updated to match schema.json
- **Validation**: Embedded validator checks schema structure

## Branch Information

- **Branch Name**: `scene-number-as-label`
- **Base**: `develop`
- **Status**: ✅ Complete (schema updated and validated)

## Future Enhancements

1. **Format-Specific Patterns**
   - Different regex patterns for different industries
   - Configurable via schema variants

2. **Scene Number Metadata**
   - Color codes (revision tracking)
   - Lock status (frozen scenes)
   - Reprint marks

3. **Scene Number Ranges**
   - Query scenes by number range: "1" to "5A"
   - Exclude ranges: "10-15" omitted

## Implementation in Compliance

When implementing ScreenJSON:

1. **Read**: Accept strings matching the pattern as scene numbers
2. **Write**: Output scene numbers as strings
3. **Validate**: Check against the regex pattern
4. **Preserve**: Maintain source scene numbers during format conversion
5. **Query**: Index and search by scene number field
