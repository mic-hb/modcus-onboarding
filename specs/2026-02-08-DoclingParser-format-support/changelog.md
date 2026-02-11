# DoclingParser Format Support Update

**Date**: 2026-02-08  
**Version**: v0.2.1  
**Status**: Completed

## Summary

Updated the DoclingParser to support the modern DoclingDocument JSON format (v1.8.0+) while maintaining backward compatibility with the legacy format.

## Changes

### 1. DoclingParser Enhancement

**File**: `modcus_api_ingest/services/parsing/docling_parser.py`

The parser now supports two Docling JSON formats:

#### Modern Format (Docling v1.8.0+)

```json
{
  "schema_name": "DoclingDocument",
  "version": "1.8.0",
  "name": "document_name",
  "origin": {
    "mimetype": "application/pdf",
    "filename": "document.pdf"
  },
  "body": {
    "children": [
      {"$ref": "#/texts/0"},
      {"$ref": "#/texts/1"},
      {"$ref": "#/tables/0"}
    ]
  },
  "texts": [
    {"text": "Content section 1", "label": "section_header", ...},
    {"text": "Content section 2", "label": "paragraph", ...}
  ],
  "tables": [...],
  "pictures": [...]
}
```

**Key characteristics:**
- Text content is stored in a `texts` array at the top level
- Content is organized via references in `body.children` using `$ref` pointers
- Tables are in a top-level `tables` array
- Figures/images are in a top-level `pictures` array
- Rich metadata including document origin and version

#### Legacy Format (Older Docling versions)

```json
{
  "schema_name": "DoclingDocument",
  "pages": [
    {
      "text": "Page 1 content",
      "tables": [...],
      "figures": [...]
    },
    {
      "text": "Page 2 content",
      "tables": [...],
      "figures": [...]
    }
  ]
}
```

**Key characteristics:**
- Text content is stored per-page in the `pages` array
- Tables and figures are nested within each page object
- Simpler structure without reference-based organization

### 2. Implementation Details

**Text Extraction Logic:**

1. **Modern Format Detection**: Checks for `body` with `children` array
2. **Reference Resolution**: Builds a map of `texts` array indexed by reference (`#/texts/{index}`)
3. **Content Extraction**: Iterates through `body.children` and resolves each `$ref` to extract text
4. **Fallback to Legacy**: If modern format not detected, falls back to `pages` array parsing

**Additional Improvements:**

- Enhanced logging throughout parsing process
- Better error handling with context
- Support for extracting metadata (version, name, origin)
- Support for extracting tables from both formats
- Support for extracting figures/pictures from both formats

### 3. Format Detection

**File**: `modcus_api_ingest/services/parsing/base.py`

Format detection checks for:
1. File extension `.pdf` → PDF format
2. JSON with `schema_name == "DoclingDocument"` → Docling format
3. JSON with `job_id` and `pages` → LlamaParse format

**Updated detection logic** to use `schema_name` field (correct field name for DoclingDocument).

## Testing

### Test Results

**Document**: `GOTO_2024_5-pages_1.json`
- **Format**: Modern DoclingDocument (v1.8.0)
- **Text items found**: 20
- **Text parts extracted**: 10
- **Total text length**: 623 characters
- **Tables**: 1
- **Figures**: 0

**Extracted content preview:**
```
01 02 03 04 05 06 07
Ikhtisar Keuangan Utama
Key Financial Highlights
Ikhtisar Kinerja Aktual Grup G...
```

## Documentation Updates

Updated the following documentation files:

1. **docs/docs/specs/2026-02-03_v0.2-project-rebuild/03-technical-plan.md**
   - Added detailed DoclingDocument format specification
   - Documented both modern and legacy formats
   - Added format examples

2. **docs/docs/specs/2026-02-03_v0.2-project-rebuild/01-epic-brief.md**
   - Enhanced "Supported Formats" section with format details
   - Added detection criteria for each format
   - Clarified Docling JSON format variations

3. **docs/docs/MODCUS_MASTER_DOCUMENT.md**
   - Updated Document Parsing description
   - Added note about DoclingDocument format support

## Backward Compatibility

✅ **Fully backward compatible** - The parser automatically detects and handles both modern and legacy formats without requiring any configuration changes.

## Migration Notes

No migration required. Existing documents in either format will be parsed correctly.

## Related Changes

- **Error Handling**: Enhanced logging and error handling in ingestion pipeline
- **Empty Content Validation**: Added validation to prevent empty text from reaching embedding stage
- **Database Schema**: Added `stage_error_details` column to Job model for better error tracking

## References

- Docling GitHub: https://github.com/DS4SD/docling
- Docling Documentation: https://docling.io/
- Related PR: Ingestion logging and error handling enhancement
