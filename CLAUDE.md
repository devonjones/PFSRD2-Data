# Claude Instructions for PFSRD2 Data

This is a data repository for the Pathfinder 2nd Edition Reference Document (PFSRD2). It contains structured JSON files generated from HTML.

## Key Context

- **Source Files**: HTML files from pfsrd-web
- **Output**: Structured JSON following defined schemas
- **Parser**: Located in ../PFSRD2-Parser/ directory
- **Schema Validation**: All JSON must validate against schema files

## CRITICAL: Git Staging Rules

**NEVER use `git add <directory>` or `git add .` in this repo.** Modified files and new (untracked) files must be staged separately and deliberately. Using `git add` on a directory stages both, which mixes unreviewed modified data into the staging area with no way to separate it back out.

**"New files" means untracked files.** When asked to stage "new" files, use:
```bash
git ls-files --others --exclude-standard -- <directory>/ | xargs git add
```
NEVER use `git add <directory>/` as a shortcut. That stages modified files too.

**Modified files must be staged selectively.** Use `git_diff_filter` to match specific diff patterns, review the matches, then stage them. The whole point of this workflow is that parser runs produce thousands of changed files and we need to verify changes in batches by pattern before staging. Bulk-staging modified files destroys that process.

**Three categories of files, three different approaches:**
1. **Untracked (new)**: `git ls-files --others --exclude-standard` to list, then `xargs git add`
2. **Modified (known patterns)**: `git_diff_filter` to match, then `| xargs git add`
3. **Modified (complex/unknown)**: Review individually with `git diff -- <file>` before staging

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

## Utility Functions

When writing parser code, prefer using these battle-tested utility functions from `universal/universal.py` and `universal/utils.py` instead of writing custom logic:

### Object Creation (universal/universal.py)
- **`build_object(type, subtype, name)`** - Create a single stat_block_section object
- **`build_objects(type, subtype, names_list)`** - Create multiple objects from a list of names
- **`link_modifiers(modifiers)`** - Add links to modifier objects by matching names to trait database

### Text Parsing with Modifiers (universal/universal.py)
- **`extract_modifiers(text)`** - Extract text and modifiers from "text (modifier1, modifier2)" format. Returns `(text, modifiers_list)`. **Note**: Asserts text ends with `)` if it contains `(`
- **`string_with_modifiers(text, subtype)`** - Parse text with optional modifiers into an object with name and modifiers fields
- **`string_with_modifiers_from_string_list(strlist, subtype)`** - Apply `string_with_modifiers` to each item in a list
- **`modifiers_from_string_list(strlist)`** - Convert list of strings to modifier objects

### Number Parsing (universal/universal.py)
- **`parse_number(text)`** - Parse integer from text, handling negative signs and em-dashes
- **`number_with_modifiers(text, subtype)`** - Parse number with optional modifiers

### Splitting (universal/utils.py)
- **`split_maintain_parens(text, delimiter)`** - Split on delimiter while respecting parentheses. Use for splitting "value1 (mod), value2 (mod)" on comma
- **`split_comma_and_semicolon(text)`** - Split on both `,` and `;` while respecting parentheses. Use for modifier lists like "alchemical; underwater only"

### Link Extraction (universal/universal.py)
- **`get_links(bs)`** - Extract link objects from BeautifulSoup element
- **`link_value(value_obj)`** - Add links to a value object by matching text
- **`link_values(values_list)`** - Apply `link_value` to each item in a list

### Example Usage
```python
# Instead of manual object creation:
modifier = {'type': 'stat_block_section', 'subtype': 'modifier', 'name': 'magical'}

# Use:
from universal.universal import build_object
modifier = build_object('stat_block_section', 'modifier', 'magical')

# Instead of manual list creation:
modifiers = []
for name in ['alchemical', 'magical']:
    modifiers.append({'type': 'stat_block_section', 'subtype': 'modifier', 'name': name})

# Use:
from universal.universal import build_objects, link_modifiers
modifiers = link_modifiers(build_objects('stat_block_section', 'modifier', ['alchemical', 'magical']))

# Instead of manual parentheses extraction:
match = re.match(r'^(.+?)\s*\((.+)\)$', text)
if match:
    value = match.group(1)
    modifier_text = match.group(2)

# Use:
from universal.universal import extract_modifiers
value, modifiers = extract_modifiers(text)

# Instead of simple split that breaks inside parentheses:
parts = text.split(',')  # WRONG: breaks "value (a, b), other"

# Use:
from universal.utils import split_maintain_parens
parts = split_maintain_parens(text, ',')  # Correctly handles "(a, b)"
```

## Development Tools

### json_map - Test Case Finder

Located at `../PFSRD2-Parser/bin/json_map`

Recursively scans JSON files and builds a map of all keys, recording the first file where each key was found. Use this to find a test case file for any JSON path.

**Usage:**
```bash
../PFSRD2-Parser/bin/json_map equipment/ > equipment_map.json
```

**Output:** JSON structure where each key shows:
- `file`: First file containing this field (use as test case)
- `type`: Value type (dict, list, str, etc.)
- `children`: Nested keys if value is dict or list of dicts

**Example:** To find a file with an `activation` field, run json_map and look up that key in the output to get a concrete file to test against.

### json_cardinality - Field Value Analyzer

Located at `../PFSRD2-Parser/bin/json_cardinality`

Analyzes field values to determine cardinality and spot potential issues:
- **Very low cardinality** (<=5 unique): Effectively an enum
- **Low cardinality**: Controlled vocabulary
- **Medium cardinality**: Worth investigating for inconsistencies/bugs
- **High/Very high cardinality**: Free text or ID fields

**Two modes:**

1. **JSONPath mode** - Analyze values at a specific path:
   ```bash
   ../PFSRD2-Parser/bin/json_cardinality --path '$.type' equipment/
   ../PFSRD2-Parser/bin/json_cardinality --path '$.sources[*].name' monsters/
   ```

2. **Object type mode** - Find all objects of a type anywhere in the JSON tree and analyze a field:
   ```bash
   # By type field (e.g., "link", "source")
   ../PFSRD2-Parser/bin/json_cardinality --type link --field game-obj equipment/

   # By subtype field (for stat_block_section objects)
   ../PFSRD2-Parser/bin/json_cardinality --subtype attack_damage --field damage_type monsters/
   ```

**Output includes:**
- File/match counts
- Total occurrences including DNE (field not present)
- Unique value count
- Cardinality assessment
- Value distribution with counts and percentages

**Options:**
- `--top N` - Show top N values (default 50)
- `--all` - Show all values
- `--value "X"` - List files containing specific value X (use `<DNE>` for missing, `<null>` for null)

### git_diff_filter - Selective Staging by Diff Pattern

Located at `../PFSRD2-Parser/bin/git_diff_filter`

Filters unstaged modified files by their diff content. Outputs matching file paths to stdout for piping to `xargs git add`.

**Usage:**
```bash
git_diff_filter +N -M [string:count ...]
```

**Arguments:**
- `+N` - Exactly N lines added
- `-M` - Exactly M lines removed
- `string:count` - String appears exactly `count` times across all changed lines (partial match, case-sensitive)

**Examples:**
```bash
# Files with exactly 1 line added, 1 removed, "value" appearing twice
../PFSRD2-Parser/bin/git_diff_filter +1 -1 value:2

# Files with 3 added, 2 removed, specific pattern
../PFSRD2-Parser/bin/git_diff_filter +3 -2 value:2 "]:":2 activation_time:1

# Multiple string filters
../PFSRD2-Parser/bin/git_diff_filter +2 -2 activation:2 type:4

# Stage matching files
../PFSRD2-Parser/bin/git_diff_filter +1 -1 value:2 | xargs git add
```

**Hunk mode** - Match per-hunk instead of whole-diff. Every hunk must match at least one `--hunk` pattern:
```bash
# All hunks must be -1 +5 with "usage" appearing 3 times
../PFSRD2-Parser/bin/git_diff_filter --hunk -1 +5 usage:3 --hunk-count 4

# Multiple patterns: each hunk matches either pattern (any-of)
# e.g., files with one access hunk and several usage hunks
../PFSRD2-Parser/bin/git_diff_filter --hunk -2 +10 access:5 --hunk -1 +5 usage:3 --hunk-count 5

# Without --hunk-count, just requires all hunks match one of the patterns
../PFSRD2-Parser/bin/git_diff_filter --hunk -1 +5 usage:3 --hunk -2 +10 access:5
```

**Hunk mode arguments:**
- `--hunk` - Start a new hunk pattern (same +N -M string:count syntax)
- `--hunk-count C` - Require exactly C hunks total

**Workflow:** When parser changes produce many modified JSON files, use this to stage files in batches by their change pattern:
1. Run `git diff --stat` to see change patterns
2. Use `git_diff_filter` to isolate files with specific patterns
3. Review a sample with `git diff -- $(git_diff_filter ... | head -1)`
4. Stage the batch with `| xargs git add`

### git_stage_license_only.py - License/Reorder Staging

Located at `../PFSRD2-Parser/bin/git_stage_license_only.py`

For each modified JSON file, builds an intermediate version with old data values re-serialized using the new file's key ordering, plus the new license block. This stages key-reorder noise and license changes while leaving real data changes unstaged.

**Usage:**
```bash
python3 ../PFSRD2-Parser/bin/git_stage_license_only.py equipment/
python3 ../PFSRD2-Parser/bin/git_stage_license_only.py monsters/
```

### Hunk-Level Staging with awk + git apply

When `git_diff_filter` finds whole-file matches, use `| xargs git add`. For files with mixed changes where only specific hunks should be staged, extract matching hunks and apply them:

```bash
git diff -- "$file" | awk '
  /^diff --git/  { header = $0; next }
  /^index /      { header = header "\n" $0; next }
  /^--- /        { header = header "\n" $0; next }
  /^\+\+\+ /     { header = header "\n" $0; next }
  /^@@/ {
    if (hunk) {
      # Check if hunk matches your pattern
      if (hunk ~ /your_pattern/) matched = matched hunk
    }
    hunk = $0 "\n"
    next
  }
  { hunk = hunk $0 "\n" }
  END {
    if (hunk && hunk ~ /your_pattern/) matched = matched hunk
    if (matched) print header "\n" matched
  }
' | git apply --cached
```

**Key rules for hunk-level staging:**
- Use a single header with all matching hunks concatenated (don't repeat the header per hunk — git can't apply two patches to the same file sequentially)
- Match on changed lines (`/^[+-]/`) not context lines to avoid false positives
- Hunks that depend on other hunks' context will fail — these need to be staged together or the dependent hunks staged first
- After staging other hunks (e.g., no-newline changes), retry failed hunks — they may now apply cleanly

## Staging Order Recommendations

When staging a large parser run, work through changes in this order:
1. **Untracked files** — `git ls-files --others --exclude-standard -z | xargs -0 git add`
2. **License/reorder** — `git_stage_license_only.py <dir>`
3. **No-newline-at-EOF** — hunk-level stage (these unblock other hunk staging)
4. **Simple whole-file patterns** — `git_diff_filter` for exact-match diffs
5. **Structural hunks** — alternate_link additions, source removals, image additions, link movements, save_results movements
6. **Content hunks** — errata version bumps, craft_requirements, text changes
7. **Remaining complex files** — review and stage individually or in bulk after vetting

## Common Change Categories (Remaster Migration)

These are expected changes when re-parsing after AoN content updates:
- **Magic school trait removals** — Abjuration/Conjuration/etc. deprecated by Remaster
- **Spell renames** — acid arrow→acid grip, true strike→sure strike, cone of cold→howling blizzard, etc.
- **Terminology** — flat-footed→off-guard, negative→void damage, positive→vitality, level→rank
- **Alignment→Sanctification** — Good→Holy, Evil→Unholy, Celestial→Empyrean
- **Apostrophe normalization** — straight (') → curly (')
- **Encoding fixes** — `\u00C2` artifacts cleaned up (e.g., "10Â minutes" → "10 minutes")
- **Trait source updates** — e.g., Clockwork trait source Grand Bazaar → Monster Core 2
- **Link aonid updates** — AoN database reorganization changes reference IDs
