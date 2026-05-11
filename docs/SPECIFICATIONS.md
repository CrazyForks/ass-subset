# Technical Specifications

<div align="right">

**[中文](./SPECIFICATIONS.zh-hans.md) | English**

</div><br/>

<div align="center">

| [Open Tool](https://subs.js.org/ass-subset/) | [Report an Issue](https://github.com/MontageSubs/ass-subset/issues) | [Join the Discussion](https://github.com/MontageSubs/ass-subset/discussions) |
| :---: | :---: | :---: |

</div><br/>

## Lossless Vector Drawing Restoration

**Supported Version**: v2.7+

When vector drawing commands are converted to font glyphs, the original command data must be preserved to enable later recovery. This tool creates a custom `draw` table within the **vector drawing font** to record this metadata.

### Glyph Allocation Rules

Vector drawing commands are embedded by converting them into font glyph outlines. Following the **subtitle timeline sequence**, each drawing command maps to one glyph with an assigned character reference: the first command becomes glyph `A`, the second becomes `B`, and so on. Character assignment skips ASS reserved symbols (`{`, `}`, `\`, `-`) and invisible characters (spaces, tabs, etc.). When subtitles are edited and drawing commands removed or modified, the `draw` table auto-reindexes—subsequent glyphs shift their character assignments accordingly, and all subtitle references sync automatically. This allocation scheme has been standard since the tool's inception, with the drawing font initially named `ASSDrawSubset_MontageSubs` before being standardized as **`ASSDrawSubset`**.

### Data Storage Specification

The custom `draw` table stores all converted drawing commands and their corresponding glyph mappings. Its structure is defined below:

#### Table Header

The table opens with a **32-bit big-endian unsigned integer** recording the total entry count. If 15 distinct drawing commands were converted, this value equals 15.

#### Entry Structure

Following the header, entries are arranged sequentially. Each entry contains three fields:

| Field | Type & Size | Purpose | Example |
|-------|------------|---------|---------|
| **data** | uint16 length + UTF-8 string | Complete original drawing command text | `m 100 100 l 200 200 c 250 250 300 300 350 350` |
| **char** | uint8 length + UTF-8 string | Single Unicode character for the glyph | `A` or `①` or any unused character |
| **flags** | 1-byte bitfield | Precision level and termination marker | See spec below |

#### flags Field Specification

The flags byte bit layout:

| Bit Range | Meaning | Value Range | Notes |
|-----------|---------|-------------|-------|
| Bits 0–3 | Drawing precision level | 1 or 2 | Value 1 = `\p1` precision; value 2 = `\p2` |
| Bit 4 | `\p0` terminator flag | 0 or 1 | 1 if the original command ends with `\p0`; 0 otherwise |
| Bits 5–7 | Reserved | 0 | Currently unused; must be 0; reserved for future extension |

**Example**: For command `\p2 m 100 100 l 200 200 \p0`, the flags value is `0b00010010` = 18 (decimal). Bits 0–3 = 2 (precision `\p2`), bit 4 = 1 (has `\p0`), bits 5–7 = 0.

### Recovery Workflow

When users click "Remove Embedded Fonts", the tool executes these steps:

1. **Scan Vector Drawing Fonts** — Iterate through all embedded fonts in the subtitle file, focusing on those containing a `draw` table.
2. **Verify draw Table** — Check if the font contains a `draw` table; skip if absent.
3. **Read Mappings** — Read the header entry count, then sequentially extract `data`, `char`, and `flags` from each entry.
4. **Restore Commands** — Search for all glyph references to this font within the subtitle's drawing instruction regions; use the `flags` field to restore the `\p` precision marker and `\p0` terminator.
5. **Remove Font File** — After recovery, delete the font file from the subtitle.

**Recovery Example**: If glyph `A` maps to command `m 100 100 l 200 200` with `\p2` precision and a `\p0` terminator, it recovers as `\p2 m 100 100 l 200 200 \p0`.

### Verification & Debugging

A command-line diagnostic tool is provided at [`/scripts/python/draw_font_inspector.py`](/scripts/python/draw_font_inspector.py) to inspect the `draw` table:

- Lists all mapping entries and their contents
- Displays `data`, `char`, and `flags` values per entry
- Validates flags bit composition against the specification
- Assists developers in debugging drawing command conversion

**Specification Maintenance Note**: The v2.7+ `draw` table specification is current. Future updates may add additional bit markers in the flags field or extend the table structure based on actual requirements. Documentation and diagnostic scripts will update accordingly.

---

## Font Name Randomization & Restoration Mapping

**Supported Version**: v2.6+

### Background & Fansub Community Solution

#### The Player Cache Problem

Font subsetting is a standard optimization for CJK typography, reducing file size by keeping only necessary glyphs. However, when subtitles span multiple episodes or a single MKV contains multiple seasons, this exposes a player caching issue:

**The Issue**: Players cache the first-loaded font and its supported character set. Since subset fonts only contain required characters, when playback reaches a later episode using the same font name but different characters, the player reuses its cache instead of reloading—resulting in missing glyphs.

#### The Fansub Community Solution: Font Name Randomization

To solve this, the CJK **Fansub community** innovated **font name randomization**: each embedded font gets a random name, forcing players to treat it as a new font and reload it, bypassing the cache entirely.

### Randomization Rules

#### Name Generation

Each time a font embeds, the tool generates a random 8-character name replacing the original. This name uses:

- Uppercase letters: A–Z (26 characters)
- Digits: 0–9 (10 characters)  
- **Total combinations**: 36^8, ensuring collision-free randomness

**Example**: Arial might randomize to `QTSGQJIO`, `8X2KL5PN`, etc.

#### Application Scope

The random name replaces the original in all subtitle locations:

- Style declaration in the stylesheet
- Font tags in dialogue lines (e.g., `{\fnQTSGQJIO}`)
- The embedded font file's name table itself

Each processing cycle generates a fresh random name, forcing player reload on every playback.

### Mapping Record Specification

To recover the original font name, the mapping between randomized and original names is recorded through dual backup methods:

#### Method 1: ASS Script Info Comments (Fansub Standard)

The standard approach used by CJK **Fansub** subsetting tools. Add comments to the `[Script Info]` section:

```
; Font Subset: randomized_name - original_name
```

**Example**:
```
[Script Info]
; Font Subset: QTSGQJIO - Arial
; Font Subset: 8X2KL5PN - SimHei
ScriptType: v4.00+
PlayDepth: 10
...
```

**Strengths & Weaknesses**: Human-readable and easily edited; however, careless text editors may accidentally strip these comment lines.

#### Method 2: Font ID 10 Table Backup (Tool Enhancement)

Starting with v2.6+, this tool additionally stores metadata in the **embedded font's ID 10 table** (Description field) as a safety net. If subtitle comments are lost, the mapping persists in the font file itself.

Within the font's ID 10 table, embed YAML metadata named `FontSubsetMap` with these fields:

| Field | Type | Purpose | Example |
|-------|------|---------|---------|
| **original** | string | The true original font name | `Arial` |
| **subset** | string | The randomized name | `QTSGQJIO` |
| **ass-subset** | version string | Tool version that created this entry | `2.6` or `2.7` |

**Complete ID 10 Record Example**:
```
FontSubsetMap: {original: Arial, subset: QTSGQJIO, ass-subset: 2.6}
```

### Recovery Workflow

When users click "Remove Embedded Fonts", the tool follows this sequence:

#### Step 1: Check Font Metadata Backup

The tool iterates through each embedded font and attempts reading its ID 10 table. If `FontSubsetMap` metadata is found and valid, extract the `original` and `subset` fields as the mapping pair.

#### Step 2: Fallback to Comments

If ID 10 lacks `FontSubsetMap`, search the subtitle's `[Script Info]` section for a matching `; Font Subset:` comment to retrieve the backup mapping.

#### Step 3: Replace Font Names in Subtitle

For fonts with found mappings, search the subtitle everywhere for the randomized name (`subset` value) and replace all instances with the original (`original` value). Replacement covers:

- Stylesheet font declarations
- Dialogue font tags
- Any other text occurrences

**Replacement Example**: For mapping `original: Arial, subset: QTSGQJIO`, all instances of `{\fnQTSGQJIO}` become `{\fnArial}`, and standalone `QTSGQJIO` becomes `Arial`.

#### Step 4: Delete Embedded Fonts

After replacement, remove all embedded font files from the subtitle. The player will render using system-installed originals during playback.

#### Mapping Priority

| Priority | Source | Notes |
|----------|--------|-------|
| First | Font ID 10 table | More reliable; immune to accidental deletion |
| Second | ASS comments | Fallback method |
| None | Both absent | Skips removal; alerts user that font cannot be removed |

### Verification & Debugging

A command-line diagnostic tool at [`/scripts/python/font_name_inspector.py`](/scripts/python/font_name_inspector.py) inspects font name tables.

---

<div align="center">

**MontageSubs (蒙太奇字幕组)**  
"Powered by Love ❤️ 用爱发电"

</div>
