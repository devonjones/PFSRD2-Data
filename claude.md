# Claude Instructions for PFSRD2 Data

This is a data repository for the Pathfinder 2nd Edition Reference Document (PFSRD2). It contains structured JSON files generated from HTML.

## Key Context

- **Source Files**: HTML files from pfsrd-web
- **Output**: Structured JSON following defined schemas
- **Parser**: Located in ../PFSRD2-Parser/ directory
- **Schema Validation**: All JSON must validate against schema files

## Important Patterns

1. **Schema Compliance**: All generated JSON must validate against the corresponding .schema.json file

2. **Link Preservation**: When parsing HTML, preserve link objects (game-obj, aonid) - don't strip them to plain text
   - Example: `<a href="Rules.aspx?ID=2161">object immunities</a>` should become a link object with aonid and game-obj
   - The parser uses `link_objects()` from universal.universal to extract links from name fields

3. **Consistent Structure**: Follow existing patterns in the codebase for similar data types

4. **HTML Link Consistency**: If some HTML files have links for a term (e.g., "object immunities") and others don't, the HTML files should be updated to add the missing links for consistency
   - Don't just parse whatever is there - fix the source HTML when links are inconsistently present
   - Example: If "object immunities" has a link in one siege weapon but not another, add the link to all files
   - This ensures consistent, complete data across all entries
