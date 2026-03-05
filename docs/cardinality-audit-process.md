# Equipment Data Cardinality Audit Process

How to use `json_cardinality` to find data quality issues across the parsed JSON output.

## Tool Location

```
PFSRD2-Parser/bin/json_cardinality
```

## Two Modes

### 1. JSONPath mode — top-level and nested fields by path

```bash
cd pfsrd2-data
python3 ../PFSRD2-Parser/bin/json_cardinality -p '$.type' equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -p '$.stat_block.level' equipment/
```

### 2. Object type mode — find all objects matching type/subtype and check a field

```bash
# By type= attribute
python3 ../PFSRD2-Parser/bin/json_cardinality -t link -f game-obj equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -t trait -f name equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -t bonus -f bonus_type equipment/

# By subtype= attribute
python3 ../PFSRD2-Parser/bin/json_cardinality -s activation_type -f value equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -s bulk -f text equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -s price -f currency equipment/
```

### Finding specific files for a value

Add `-v` to locate files containing a specific value:

```bash
python3 ../PFSRD2-Parser/bin/json_cardinality -s activation_type -f value -v 'intract' equipment/
python3 ../PFSRD2-Parser/bin/json_cardinality -t link -f game-obj -v '<DNE>' equipment/
```

Use `--all` to show all unique values instead of the default top 50.

## Audit Checklist by Data Type

The schema at `pfsrd2-data/equipment.schema.1.0.json` defines all valid fields and enums. Use it to guide which fields to check.

### Phase 1: Structural integrity (enum fields)

These should be small controlled vocabularies. Any unexpected value is a bug.

```bash
# Top-level enums
-p '$.type'                    # Should match directory
-p '$.game-obj'                # Should match directory
-p '$.stat_block.subtype'      # Should match type
-p '$.edition'                 # legacy | remastered
-p '$.pfs'                     # Standard | Limited | Restricted

# Nested enums
-s action_type -f name         # One Action, Two Actions, etc.
-s save_dc -f save_type        # Fort | Ref | Will | Flat Check
-s activation_type -f value    # interact, envision, command, etc.
-t bonus -f subtype            # ac | skill | speed
-t bonus -f bonus_type         # armor | dexterity | shield
-s reload -f unit              # action | minute | round
-s statistics -f category      # Unarmed, Simple, Martial, etc.
```

### Phase 2: Controlled text fields

These have moderate cardinality. Scan for typos, inconsistencies, and garbage values.

```bash
-s bulk -f text                # L, -, 1, 2, etc.
-s price -f text               # "5 gp", "2 sp", etc.
-s price -f currency           # gp | sp | cp | pp
-s usage -f text               # "held in 1 hand", "worn", etc.
-s statistics -f hands         # 1, 2, "1 or 2"
-s modifier -f name            # "per Bulk", "by armor", etc.
-s immunity -f name            # "object immunities", etc.
-t trait -f name               # All trait names
-t weapon_group -f name        # Sword, Knife, etc.
-t armor_group -f name         # Plate, Leather, etc.
-t link -f game-obj            # Sources, Traits, etc. (<DNE> = unresolved)
```

### Phase 3: Leaked structured content in text fields

This is the most important check. Look for structured game data that should have been parsed into dedicated fields but ended up as raw text instead.

#### What to look for in `stat_block.text` and variant `.text`:

| Pattern | Should be in field |
|---|---|
| `**Destruction** ...` | `ability.destruction` |
| `**Perception** ... **Will**` | `intelligent_item` object |
| `Saving Throw DC N ...; Stage 1 ...` | `affliction` object |
| `**Activate** ...` | `activation` object |
| `**Craft Requirements** ...` | `stat_block.craft_requirements` |

#### Python scan for leaked content

```python
import json, glob, re

for f in sorted(glob.glob('equipment/**/*.json', recursive=True)):
    d = json.load(open(f))
    sb = d.get('stat_block', {})

    # Check main text
    for target in [sb] + sb.get('variants', []):
        text = target.get('text', '')
        for pat, label in [
            (r'\*\*Destruction\*\*', 'Destruction'),
            (r'\*\*Perception\*\*.*\*\*Will\*\*', 'Intelligent Item'),
            (r'Saving Throw.*Stage \d', 'Affliction'),
            (r'Stage \d.*Stage \d', 'Affliction stages'),
        ]:
            if re.search(pat, text, re.DOTALL):
                has_field = label.lower().replace(' ', '_') in target
                print(f'{label} in text (structured={has_field}): {f}')
                break

    # Check variants with Activate in text but no abilities
    for v in sb.get('variants', []):
        text = v.get('text', '')
        stats = v.get('statistics', {})
        abilities = stats.get('abilities', []) if isinstance(stats, dict) else []
        if 'Activate' in text and len(abilities) == 0:
            print(f'Variant missing activation: {f} / {v.get("name")}')
```

### Phase 4: Field population rates

Check which fields exist and how often, to spot missing data:

```python
import json, glob
from collections import Counter

fields = Counter()
for f in glob.glob('equipment/**/*.json', recursive=True):
    d = json.load(open(f))
    for k in d.get('stat_block', {}):
        fields[k] += 1

for field, count in fields.most_common():
    print(f'{field}: {count}')
```

Expected for equipment (4469 files): type/subtype/level at 100%, bulk ~99%, text ~98%, traits ~93%, statistics ~92%, links ~66%, price ~64%, variants ~29%.

## Applying to Other Data Types

The same process works for any data directory. Adjust the schema reference:

| Directory | Schema | Key differences |
|---|---|---|
| `armor/` | `equipment.schema.1.0.json` | Has `defense` (AC bonus, armor group, dex cap) |
| `shields/` | same | Has `defense` (shield bonus, hitpoints) |
| `weapons/` | same | Has `offense` (weapon_modes, damage, range) |
| `siege_weapons/` | same | Has `statistics` (crew, usage), `offense`, `defense` |
| `vehicles/` | same | Has `statistics` (crew, passengers, piloting_check, speeds) |
| `equipment/` | same | Has `variants`, `activation`, `affliction`, `intelligent_item` |
| `monsters/` | `creature.schema.1.3.json` | Completely different structure |

## Known Acceptable Patterns

These look unusual but are correct:

- **Links without `game-obj`**: Category links (`Equipment.aspx?Category=...`), `SpellLists.aspx`, `Shields.aspx`, `AnimalCompanions.aspx`, and paizo.com external links. These can't resolve to a single game object.
- **Price with no currency**: Free items (fist, club, unarmored) have text `"-"` or `"0"` and null value.
- **Material Hardness/HP/BT tables in text**: Material items (adamantine, cold iron, etc.) have markdown tables with `**Hardness**`, `**HP**`, `**BT**` columns. This is expected descriptive content.
- **Barding AC tables in text**: `barding.json` files contain markdown tables with armor stats. Expected.
- **"fire" damage_type on weapons**: `fire_poi.json` legitimately deals fire damage.
- **Long stat_block.text**: Many items (especially artifacts, intelligent items, complex equipment) have legitimately long descriptions.

## Filing Issues

After finding problems, create beads tickets:

```bash
bd create --title="Short summary" --description="Details with file list" --type=bug --priority=2
bd dep add <new-id> <epic-id>
```

Group related issues under an epic. The current equipment validation epic is `PFSRD2-hul`.
