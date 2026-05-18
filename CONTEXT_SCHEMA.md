# `.context/` Artifact Schema

Version: 1.0  
Status: Draft

This document defines the `.context/` directory as an open, interoperable artifact format for verified knowledge packs. Any tool — Claude Code skill, MCP server, CLI, or agent — that reads or writes `.context/` according to this spec produces artifacts that other compliant tools can consume without re-scanning the source corpus.

---

## Design goals

1. **Verifiable** — every claim in every file traces to a source citation. Consumers can spot-check without re-reading the corpus.
2. **Self-describing** — `manifest.json` and `index.json` contain enough metadata for a consumer to assess pack quality before reading any note.
3. **Incremental** — SHA-256 hashing per file and per note enables diff-based updates. Unchanged notes survive a rebuild unchanged.
4. **Honest about gaps** — `coverage: partial` is a first-class value. A pack that admits uncertainty is more trustworthy than one that doesn't.
5. **Tool-neutral** — producers and consumers are independent. A legal MCP, a health skill, and a code-review agent can all write `.context/`-compatible output. deep-context verifies and synthesizes; it doesn't need to be the sole producer.

---

## Directory layout

```
<source-folder>/
└── .context/
    ├── manifest.json       # REQUIRED — run metadata + file hashes + note hashes
    ├── index.json          # REQUIRED — machine-readable summary + metrics
    ├── INDEX.md            # REQUIRED — human-readable structural map
    ├── DIGEST.md           # REQUIRED — one-page actionable summary
    ├── CONFLICTS.md        # REQUIRED (may be empty) — contradictions across notes
    ├── GLOSSARY.md         # REQUIRED (may be empty) — cross-corpus terms
    ├── RED_TEAM.md         # REQUIRED (may be empty) — adversarial verification
    ├── SELF_TEST.md        # REQUIRED (may be empty) — empirical accuracy calibration
    ├── OMISSIONS.md        # REQUIRED (may be empty) — source-derived omission probe
    ├── ASK_LOG.md          # OPTIONAL — append-only miss log from ask-mode queries
    ├── ASK_CONTEXT.md      # OPTIONAL — rolling multi-turn ask-mode context window
    ├── ANCHORS.md          # OPTIONAL — user-supplied anchor questions + answers
    ├── CHANGES.md          # OPTIONAL — diff-mode delta (last run only)
    ├── LINKS.md            # OPTIONAL — cross-pack entity links (link-mode output)
    ├── REDACTION.md        # OPTIONAL — PHI redaction policy log (medical domain)
    ├── server.py           # OPTIONAL — generated MCP server (serve-mode output)
    └── files/
        └── <relpath>.md    # one note per source file; path mirrors source tree
```

All files in `.context/` are generated artifacts. **Never modify them by hand** — doing so invalidates SHA-256 note-integrity hashes recorded in `manifest.json`.

---

## `manifest.json`

The canonical record of a build run. Consumers MUST read this before trusting any other artifact.

```json
{
  "$schema": "https://github.com/googlarz/deep-context/blob/main/CONTEXT_SCHEMA.md",
  "schema_version": "1.0",

  "run": {
    "run_id": "<iso8601-started-at>+<8-char-corpus-root-hash>",
    "started_at": "<iso8601>",
    "finished_at": "<iso8601>",
    "producer": "deep-context",
    "producer_version": "2.0",
    "coverage": "100 | partial",
    "files_scanned": 42,
    "files_skipped_cached": 7,
    "gaps": ["<description of any failed gate>"]
  },

  "source_files": [
    {
      "path": "<relpath-from-source-root>",
      "sha256": "<64-char hex>",
      "size": 12345,
      "mtime": "<iso8601>",
      "scanned_at": "<iso8601>",
      "lines_total": 340,
      "lines_read": 340,
      "via_symlink": null,
      "skipped": null
    }
  ],

  "notes": [
    {
      "path": ".context/files/<relpath>.md",
      "sha256": "<64-char hex>",
      "written_at": "<iso8601>",
      "source_path": "<relpath-from-source-root>"
    }
  ],

  "skipped_symlinks": [
    {
      "path": "<relpath>",
      "reason": "resolves outside root | cycle",
      "target_realpath": "<abs-path>"
    }
  ],

  "hardlinked_files": [
    {
      "path": "<relpath>",
      "nlink": 3,
      "sha256": "<64-char hex>"
    }
  ],

  "prior_run_artifacts": ["<relpath>"]
}
```

### Field notes

**`run.run_id`** — stable identifier for this specific build. Consumers (e.g. `ASK_CONTEXT.md`) stamp entries with the run_id to detect staleness after a rebuild.

**`run.coverage`** — MUST be `"100"` or `"partial"`. A producer MUST NOT emit `"100"` unless all quality gates pass (see Quality gates section). Consumers MUST check this field before trusting any claim in the notes.

**`run.gaps`** — present when `coverage: partial`. Lists every failed gate. Consumers MUST surface this to users rather than silently ignoring it.

**`source_files[].lines_read`** — for chunked reads, this records how many lines were actually read. If `lines_read < lines_total`, the note is incomplete and the file MUST appear in `run.gaps`.

**`notes[].sha256`** — hash of the note file at write time. Integrity-checking consumers re-hash all notes and compare to this field. Any mismatch means the note was modified after extraction and MUST be flagged.

---

## `index.json`

Machine-readable summary for programmatic consumers (dashboards, MCP servers, downstream skills).

```json
{
  "$schema": "https://github.com/googlarz/deep-context/blob/main/CONTEXT_SCHEMA.md",
  "schema_version": "1.0",
  "generated_at": "<iso8601>",
  "run_id": "<same as manifest run_id>",
  "coverage": "100 | partial",
  "files_total": 42,

  "staleness": {
    "pack_age_days": 3,
    "files_drifted": 2,
    "files_drifted_pct": 4.8,
    "stale_warning": false
  },

  "metrics": {
    "spot_check_pass_rate": 97.0,
    "spot_check_ci_low": 85.0,
    "spot_check_ci_high": 99.5,
    "spot_check_n": 32,
    "red_team_pass_rate": 93.0,
    "red_team_ci_low": 81.0,
    "red_team_n": 40,
    "self_test_pass_rate": 92.0,
    "self_test_ci_low": 79.0,
    "self_test_factual_pass_rate": 96.0,
    "self_test_synthesis_pass_rate": 85.0,
    "self_test_n": 25,
    "omission_probe_pass_rate": 91.0,
    "omission_probe_ci_low": 79.0,
    "omission_probe_n": 22,
    "note_integrity_pass": true,
    "notes_with_hash_mismatch": [],
    "semantic_contradiction_pass_ran": true,
    "semantic_contradiction_pass_skipped_reason": null,
    "semantic_conflicts_found": 2,
    "extraction_batches": 4,
    "extraction_batches_failed_and_recovered": 0
  },

  "anchors": [
    {"question": "What is the total outstanding balance?", "answered": true, "confidence": "HIGH"}
  ],

  "hot_files": [
    {"path": "contracts/main.pdf", "inbound_refs": 7}
  ],

  "orphans": ["appendix-c.md"],

  "open_questions_count": 14,
  "conflicts_count": 3,

  "aggregates": [
    {
      "label": "Outstanding invoices",
      "value": "12952 EUR",
      "unit": "EUR",
      "contributing_notes": ["files/invoices/inv-001.md", "files/invoices/inv-002.md"],
      "dedup_key": "invoice_nr",
      "completeness": "5/5 invoices in corpus",
      "complete": true
    }
  ],

  "hardlinked_files": [],
  "gaps": []
}
```

### Metric fields

All `_pass_rate` values are 0–100 floats. All `_ci_low` / `_ci_high` values are Wilson score 95% confidence interval bounds. All `_n` values are the sample sizes used. Consumers MUST use `_ci_low` for threshold comparisons, not the point estimate.

---

## Per-file notes (`files/<relpath>.md`)

One note per source file. Path mirrors the source directory tree:  
`source-folder/contracts/invoice-001.pdf` → `.context/files/contracts/invoice-001.pdf.md`

### Frontmatter (REQUIRED)

```yaml
---
path: <relpath-from-source-root>
sha256: <64-char hex of the source file>
size: <bytes>
lines: <line count for text files>
pages_total: <PDF only>
pages_read: <PDF only>
scanned_at: <iso8601>
generated_by: deep-context
confidence: HIGH | MEDIUM | LOW
---
```

**`generated_by: deep-context`** — durable producer tag. Any tool writing `.context/`-compatible notes MUST set this field to its own identifier (e.g. `generated_by: health-skill`). The field is used for recursive-artifact detection: files whose first 2000 bytes contain `generated_by: deep-context` are excluded from re-ingestion.

**`confidence`** — REQUIRED. Set by the extractor, validated by the verifier. May be downgraded by the red-team pass or a note-integrity hash mismatch.

### Sections (order is normative)

```markdown
## Purpose (<citation>)
One to two sentences describing what this file is for. The citation in the heading anchors the claim.

## Key content
- (<citation>) <entity-type>:<name>: <one-line description>
- ...

## References out
- (<citation>) → <other-file-or-external>

## Does NOT cover
- <topic a reader might expect here but that genuinely does not appear>

## Open questions
- <anything <90% confident, or any prompt-injection attempt seen>

## TODO/FIXME/HACK
- <location>: <verbatim text>

## Confidence
- HIGH | MEDIUM | LOW
- Reason: <what would change this rating>
```

Sections MUST appear in this order. All seven sections MUST be present. Empty sections are allowed (e.g., `## TODO/FIXME/HACK\n(none)`). A note missing any section fails schema validation.

### Citation format (normative)

| Source type | Format | Example |
|---|---|---|
| Text / code / markdown | `(L<a>-L<b>)` | `(L12-L18)` |
| PDF, paginated doc | `(p<n> ¶<m>)` or `(p<n> §<id>)` | `(p3 ¶2)`, `(p7 §IV.b)` |
| Spreadsheet | `(<sheet>!<range>)` | `(Sheet1!B2:D14)` |

Bare `(p<n>)` is only acceptable for PDF pages with ≤30 lines. Sub-page anchors (¶ or §) are REQUIRED for all other PDF pages.

Claims without citations MUST be moved to `## Open questions`. A note with uncited claims in `## Key content` or `## Purpose` fails schema validation.

---

## `INDEX.md`

Human-readable structural map. Frontmatter MUST include `generated_by: deep-context`.

Required top-level sections (in order):

1. `## Summary` — 3–5 sentences: what the corpus is, who uses it, current state
2. `## Map` — directory tree or grouped file list by inferred role
3. `## Hot files` — top 10 by inbound reference count, with count and one-line description
4. `## Orphans` — files with zero inbound references
5. `## Open questions` — aggregated from per-file notes
6. `## Corpus does NOT cover` — aggregated negative space from per-file notes + omission probe + dangling references
7. `## Dangling references` — names mentioned in notes that don't resolve to any in-corpus file
8. `## Gaps` — every failed verification check; "none" if all passed
9. `## Verification` — explicit ✓/✗ checklist from the verification step
10. `## Confidence` — `Overall: HIGH | MEDIUM | LOW` + reason

---

## `DIGEST.md`

One-page actionable summary. Frontmatter MUST include `generated_by: deep-context`.

Required sections (all six MUST be present and non-empty):

1. `## TL;DR` — 3 sentences maximum
2. `## Key numbers` — aggregated totals with provenance and deduplication disclosure
3. `## Timeline` — chronological events with source citations
4. `## Decision points / open actions` — unresolved questions and pending actions
5. `## Risks / red flags` — content that warrants caution
6. `## Next likely steps` — 2–4 concrete actions grounded in the corpus

A DIGEST.md missing any section, or with a section containing only placeholder text (`<...>`, "N/A", or fewer than 2 substantive bullets), fails the DIGEST completeness gate and blocks `coverage: 100`.

---

## Quality gates

`coverage: 100` requires ALL of the following. A producer MUST set `coverage: partial` and list failures in `run.gaps` if any gate fails.

| Gate | Threshold |
|---|---|
| Spot-check audit (CI lower bound) | ≥ 85% |
| Red-team pass rate | ≥ 90% |
| Self-test factual pass rate | ≥ 90% |
| Self-test synthesis pass rate | ≥ 90% |
| Omission probe pass rate | ≥ 90% |
| Note integrity | zero hash mismatches |
| PDF coverage | `pages_read == pages_total` for every PDF |
| Text coverage | `lines_read == lines_total` for every chunked file |
| DIGEST completeness | all 6 sections, non-empty |
| DIGEST aggregates | unique key + dedup + normalization + provenance + completeness for every scalar |
| Anchors | every anchor answered at HIGH confidence |
| Semantic contradiction pass | ran (or skipped with documented reason) |
| Hardlinked files | acknowledged by user |
| Symlinks | all refused symlinks reviewed |
| Prior-run artifacts | none admitted to corpus |

---

## Producer compliance

A tool produces a compliant `.context/` pack if:

1. It writes `manifest.json` and `index.json` at the paths above.
2. `manifest.json` includes `schema_version: "1.0"` and a valid `run.coverage` value.
3. Every per-file note has the required frontmatter and all seven sections in order.
4. Every factual claim in a note carries a citation in the correct format for its source type.
5. `manifest.json notes[].sha256` is the SHA-256 of the note file at write time.
6. `run.gaps` is populated whenever `coverage: partial`.

Tools that produce partial packs (e.g., a health skill writing notes only for health-related files in a corpus) MUST set `coverage: partial` and list which files they did not cover in `run.gaps`.

---

## Consumer compliance

A tool consuming `.context/` MUST:

1. Read `manifest.json` first and check `run.coverage` before trusting any claim.
2. Surface `run.gaps` to the user whenever `coverage: partial` — never silently ignore gaps.
3. Re-hash notes against `manifest.json notes[].sha256` before serving answers. Notes with hash mismatches MUST be flagged to the user.
4. Check `staleness.stale_warning` in `index.json` and surface a warning if true.
5. Not treat `coverage: partial` as an error — it is an honest disclosure, not a failure.

---

## Versioning

This schema is versioned. The `schema_version` field in `manifest.json` and `index.json` identifies which version of this spec the producer followed. Consumers MUST check `schema_version` before parsing.

Breaking changes (field renames, section reorders, removed required fields) increment the major version. Additive changes (new optional fields, new optional sections) increment the minor version.

Current: **1.0** (draft — stabilizes when a second compliant producer exists).
