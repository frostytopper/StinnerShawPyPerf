# PMC identifier processing handoff

Date: 2026-11-07
Status: captured for later review; no StinnerShaw performance review or benchmark has been performed.

## Scope

Check two separate identifier files against a locally downloaded PMC identifier index. Preserve namespaces: pmids.txt contains bare numeric PMIDs; pmcids.txt contains PMC-prefixed PMCIDs. Resolve PMIDs to PMCIDs, then merge and deduplicate. No article downloading is performed by these scripts. Index membership is not a guarantee of current full-text download availability.

## Inputs and preparation

Run from the directory containing pmids.txt, pmcids.txt, and PMC-ids.csv.gz. Python 3 is required; both scripts use only its standard library.

The index was downloaded with:

```bash
curl -fL --retry 3 -O https://ftp.ncbi.nlm.nih.gov/pub/pmc/PMC-ids.csv.gz
gzip -cd PMC-ids.csv.gz | head -n 5
```

The user confirmed the first five lines printed as expected. The index checksum, exact snapshot date, file size, runtime, peak memory, and machine specifications were not recorded.

## Script 1: check existing PMCIDs

This is the script supplied in the conversation, preserved without optimization.

```bash
python3 - <<'PY'
import csv
import gzip
from pathlib import Path

requested = set(Path("pmcids.txt").read_text().split())
matched = set()

with gzip.open("PMC-ids.csv.gz", "rt", encoding="utf-8-sig", newline="") as f:
    reader = csv.DictReader(f)
    if "PMCID" not in (reader.fieldnames or []):
        raise SystemExit(f"Missing PMCID column. Headers: {reader.fieldnames}")

    for row in reader:
        pmcid = row["PMCID"].strip()
        if pmcid in requested:
            matched.add(pmcid)

for filename, ids in (
    ("pmcids.valid.txt", matched),
    ("pmcids.unmatched.txt", requested - matched),
):
    Path(filename).write_text("".join(f"{i}\n" for i in sorted(ids)))
    print(f"{filename}: {len(ids):,}")
PY
```

### Reported results

```text
pmcids.valid.txt: 74,896
pmcids.unmatched.txt: 0
```

These counts were reported by the user, not independently reproduced. All 74,896 unique input PMCIDs matched the local index. Raw input line count and duplicate count are unknown because the script converts whitespace-separated inputs to a set. The output files themselves are local and have not been uploaded here.

## Script 2: resolve PMIDs and merge PMCIDs

This is the script supplied in the conversation, preserved without optimization. Run after Script 1 because it reads pmcids.valid.txt.

```bash
python3 - <<'PY'
import csv
import gzip
from pathlib import Path

requested = set(Path("pmids.txt").read_text().split())
matched_pmids = set()
mapped_pmcids = set()

with gzip.open("PMC-ids.csv.gz", "rt", encoding="utf-8-sig", newline="") as f:
    reader = csv.DictReader(f)
    required = {"PMID", "PMCID"}
    if not required.issubset(reader.fieldnames or []):
        raise SystemExit(f"Missing required columns. Headers: {reader.fieldnames}")

    for row in reader:
        pmid = row["PMID"].strip()
        pmcid = row["PMCID"].strip()
        if pmid in requested and pmcid:
            matched_pmids.add(pmid)
            mapped_pmcids.add(pmcid)

existing = set(Path("pmcids.valid.txt").read_text().split())

for filename, ids in (
    ("pmids.matched.txt", matched_pmids),
    ("pmids.unmatched.txt", requested - matched_pmids),
    ("pmids.pmcids.txt", mapped_pmcids),
    ("pmcids.combined.txt", existing | mapped_pmcids),
):
    Path(filename).write_text("".join(f"{i}\n" for i in sorted(ids)))
    print(f"{filename}: {len(ids):,}")

print(f"New PMCIDs added: {len(mapped_pmcids - existing):,}")
PY
```

### Results status

Script 2 was provided, but no execution results were reported before this handoff. Do not infer counts or completion.

| Output | Meaning | Reported count |
| --- | --- | --- |
| pmids.matched.txt | Unique original PMIDs with a nonempty PMCID match | Not reported |
| pmids.unmatched.txt | Unique original PMIDs without a PMCID match in this snapshot | Not reported |
| pmids.pmcids.txt | Unique PMCIDs resolved from PMIDs | Not reported |
| pmcids.combined.txt | Union of resolved PMCIDs and previously validated PMCIDs | Not reported |
| New PMCIDs added | Resolved PMCIDs absent from pmcids.valid.txt | Not reported |

## Review handoff

Review these scripts using this repository's methodology before changing them. The immediate goal is accurate, simple identifier filtering, not a general metadata manifest or article downloader.

Suggested review checks, not completed findings:

- Record input counts, index checksum, environment, wall-clock time, and peak memory before making performance claims.
- Test quoted CSV fields, missing headers, duplicate inputs, unmatched identifiers, empty inputs, and one PMID associated with multiple index rows.
- Confirm set-based deduplication and lexicographic output ordering are acceptable.
- Consider whether two full gzip/CSV scans merit consolidation only after measuring the baseline.
- Review output overwrite behavior and interruption safety; current scripts write directly to output paths.
- Record Script 2 results when available, then separately assess download availability for the combined PMCIDs.

No API key or secret is needed for these local filtering steps. No identifier lists or article content are included in this document.
