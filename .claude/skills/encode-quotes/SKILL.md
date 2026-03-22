---
name: encode-quotes
description: Encode biblical and patristic quotations in SCTA XML transcription files using a candidate-driven workflow
---

Encode quotations from a candidates JSON file into TEI XML transcription files using the SCTA quotation encoding workflow.

## Arguments

`$ARGUMENTS` should be provided in the form:
```
<base_dir> <candidates_file> <xml_filename_pattern>
```

Example:
```
/Users/jcwitt/Projects/scta/scta-texts/holcotwisdom bible_candidates_for_holcotwisdom.json Basel1586_{dir_id}.xml
```

- `base_dir` — absolute path to the project directory containing the lectio subdirectories
- `candidates_file` — filename of the JSON candidates file (located in `base_dir`)
- `xml_filename_pattern` — pattern for XML filenames, using `{dir_id}` as placeholder (e.g. `Basel1586_{dir_id}.xml`)

Parse these three values from `$ARGUMENTS` at the start of every session.

---

## Goal

Find biblical and patristic quotations in the XML transcription files and mark them with `<quote>` elements including a `source` attribute pointing to the SCTA identifier with word range.

### Format
```xml
<quote source="http://scta.info/resource/<sourcepid>@start-end">quote text</quote>
```

Example:
```xml
<quote source="http://scta.info/resource/gen18_2@22-24">Abraham tres vidit. et vnum adorauit.</quote>
```

---

## Candidate-Driven Workflow

The candidates file maps directory/paragraph IDs to candidate source IDs with `intersectionCount` scores and a `status` field:

```json
{
  "rhwis-l1": {
    "rhwis-l1-d1e97": [
      { "candidate_id": "io1_3", "intersectionCount": 5, "status": "encoded" },
      { "candidate_id": "gen2_7", "intersectionCount": 2 }
    ]
  }
}
```

**Status values:**
- `"encoded"` — quote found and marked in the XML
- `"not_found"` — checked, no actual quotation present
- *(absent)* — not yet checked

**Note field:** Always add a `"note"` explaining the outcome:
- For `"not_found"`: e.g. `"false positive — both cite Rom 1:20"` or `"thematic overlap only, no attribution"`
- For `"encoded"` with caveats: e.g. `"paraphrase without @range"` or `"MS cites book 9 but encoded with book 7 candidate"`

### Processing Candidates

1. Read the candidates file; find entries with no `status` (start with highest `intersectionCount`)
2. The XML file for a directory `dir_id` is at `<base_dir>/<dir_id>/<xml_filename_pattern>` (substitute `{dir_id}`)
3. Read the corresponding paragraph in the XML file
4. Check whether the passage genuinely appears — look for explicit attribution markers (e.g. "dicit", "auctoritas", an author name, a book/chapter citation). **High `intersectionCount` alone is not sufficient** — both texts may independently quote the same source.
5. If a genuine quote is found: look it up, encode it, set `"status": "encoded"` in JSON
6. If not found: set `"status": "not_found"` in JSON

### Auto-Detecting Already-Encoded Quotes

Run this script at session start to reconcile the log against the XML files:

```bash
python3 - << 'EOF'
import json, re, os, sys

base_dir = "BASE_DIR_PLACEHOLDER"
candidates_file = os.path.join(base_dir, "CANDIDATES_FILE_PLACEHOLDER")
xml_pattern = "XML_PATTERN_PLACEHOLDER"  # e.g. Basel1586_{dir_id}.xml

with open(candidates_file) as f:
    candidates = json.load(f)
for dir_id, paragraphs in candidates.items():
    xml_filename = xml_pattern.replace("{dir_id}", dir_id)
    xml_file = os.path.join(base_dir, dir_id, xml_filename)
    if not os.path.exists(xml_file): continue
    with open(xml_file) as f: content = f.read()
    for para_id, candidate_list in paragraphs.items():
        para_match = re.search(rf'<p[^>]*xml:id="{re.escape(para_id)}"[^>]*>(.*?)</p>', content, re.DOTALL)
        if not para_match: continue
        para_content = para_match.group(1)
        for candidate in candidate_list:
            if "status" not in candidate and f'scta.info/resource/{candidate["candidate_id"]}' in para_content:
                candidate["status"] = "encoded"
with open(candidates_file, "w") as f:
    json.dump(candidates, f, indent=2)
print("Done")
EOF
```

Substitute the actual `base_dir`, `candidates_file`, and `xml_pattern` values from `$ARGUMENTS` before running.

---

## Encoding Workflow (per quote)

1. **Look up the source text** using the `textLookup` MCP service:
   ```
   POST http://localhost:3001/textLookup
   { "resourceId": "gen18_2" }
   ```

2. **Get the word range** using the `textRangeIdentify` MCP service:
   ```
   POST http://localhost:3001/textRangeIdentify
   { "resourceId": "gen18_2", "quoteFragment": "adoravit in terra" }
   ```
   Returns `start` and `end` word positions (1-indexed).

3. **Apply the encoding** — wrap the quoted text in `<quote source="...@start-end">`.

4. **Update the JSON** — set `"status": "encoded"` (or `"not_found"`) immediately after each candidate is checked.

### Paraphrases
- For paraphrases, find the closest matching fragment in the source and use its range as an approximation.
- If no close substring match exists, omit `@start-end` and cite the resource only: `source="http://scta.info/resource/gen18_2"`.

---

## MCP Services (require local server at port 3001)
- `textLookup` — retrieves the full text of a resource by ID
- `textRangeIdentify` — finds start/end word positions of a fragment within a resource
- `findTextCandidates` — finds SCTA texts matching an input string
- `identifyReference` — resolves a Latin reference string to an SCTA identifier

### Calling MCP Services via Bash
Always use `curl` via the Bash tool (NOT WebFetch). Batch parallel calls with `&` and `wait`:
```bash
curl -s -X POST http://localhost:3001/textLookup -H "Content-Type: application/json" \
  -d '{"resourceId": "gen18_2"}' &
curl -s -X POST http://localhost:3001/textRangeIdentify -H "Content-Type: application/json" \
  -d '{"resourceId": "gen18_2", "quoteFragment": "adoravit in terra"}' &
wait
```
When `textRangeIdentify` returns `found: false`, try Vulgate/standard spelling instead of manuscript spelling.

---

## Edge Cases
- **Linter**: Auto-adds `xml:id` attributes to `<quote>` elements — do not add these manually. If an Edit fails with "file has been unexpectedly modified," re-read the relevant lines before retrying.
- **`break="no"` on `<lb>`**: means a hyphenated word split across lines — keep `<lb>` elements inside `<quote>` when they fall within the quoted text.
- **Multi-verse quotes**: When a MS quote spans consecutive verses, use space-separated source URIs in a single `<quote>` element:
  ```xml
  <quote source="http://scta.info/resource/deut17_18@1-20 http://scta.info/resource/deut17_19@1-25">quote text</quote>
  ```
- **Verse boundary issues**: When a manuscript quote spans a source boundary, cite the unit containing the bulk of the text; omit `@range` if a clean range cannot be found.
- **Lectio lemmata**: Paragraphs that begin with `]` or that open with a Wisdom/source verse being expounded are typically the lectio text itself, not a quotation by the author → `not_found`.

---

## Reference: Bible Book ID Codes
- **New Testament:** mt, mc, lc, io, act, rom, Icor, IIcor, gal, eph, phil, col, Ith, IIth, Itim, IItim, tit, hebr, iac, Ipetr, IIpetr, Iio, IIio, iud, apoc
- **Old Testament:** gen, ex, lev, num, deut, ios, iudic, rut, Isam, IIsam, Ireg, IIreg, Ichr, IIchr, esd, neh, tob, iudt, est, iob, ps, prov, qoh, cant, sap, sir, is, ier, lam, bar, ez, dan, os, ioel, am, abd, ion, mic, nah, hab, ag, soph, zac, mal, Imac, IImac

Bible resource IDs follow the pattern `<bookcode><chapter>_<verse>` (e.g. `gen18_2`, `rom1_1`).
