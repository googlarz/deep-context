---
name: deep-context
description: Build a verified, comprehensive context pack from a folder of ~200 or fewer text files. Use when the user asks to "scan a folder", "build context for", "ingest", or pass a folder path to thoroughly understand. Produces `.context/INDEX.md` + `.context/DIGEST.md` + per-file notes with explicit completeness checks, citations, contradiction detection, and self-test calibration. Caches via SHA-256; re-scans only changed files when last run hit 100% coverage.
---

# deep-context

Build comprehensive, verified context from a folder. Designed for corpora ≤200 files where full reads are tractable.

## Modes

Pick based on user input:

| Mode | Trigger | Action |
|------|---------|--------|
| **build** | folder path + "scan/ingest/build context" | Full workflow (steps 1–8 below) |
| **build with anchors** | folder + 3–5 anchor questions ("how does X work?") | Build, then verify pack can answer each anchor with HIGH confidence; unanswerable anchors → Gaps |
| **ask** | existing `.context/` + a question, e.g. `/context-pack ask <folder> "<question>"` | Skip to step 9 — answer from cached notes |
| **diff** | cache hit on rebuild, ≥1 file changed | Run build on changed files only, then write `CHANGES.md` summarizing what's new since last run; run drift detection on dependent notes |
| **link** | `/context-pack link <folder-a> <folder-b>` | Cross-link two existing packs — see step 11 |

Default to **build** if mode is ambiguous and `.context/` doesn't exist; default to **diff** if it exists and any file changed; default to **ask** if user passes a question alongside an unchanged corpus.

## Output layout

## Output layout

Inside the target folder:

```
<folder>/.context/
├── INDEX.md              # human-readable summary + verification report
├── CHANGES.md            # written by diff mode — what changed since last run
├── CONFLICTS.md          # contradictions between per-file notes (may be empty)
├── GLOSSARY.md           # cross-corpus terms and entities (may be empty)
├── manifest.json         # {path, sha256, size, mtime, scanned_at} per file + run metadata
└── files/
    └── <relpath>.md      # one note per source file, mirrors source tree
```

Never write outside `.context/`. Never modify source files.

## Workflow

### -1. Domain hint (optional)

If the user passes `--domain=<legal|code|sales|research|finance|medical>` or the corpus filename/content patterns strongly suggest a domain, load the matching extraction template (small additive section, not a replacement). Domain templates add structured fields to per-file notes. Examples:

- **legal**: case numbers, court, parties, statute references, dates (DD.MM.YYYY normalized to ISO), monetary amounts, deadlines, cited precedents
- **code**: language, exports, imports, side effects, test coverage, public API surface
- **sales**: account, deal stage, ACV, close date, decision-makers, objections
- **research**: hypothesis, method, sample size, findings, confidence level, citations
- **finance**: amounts, currency, dates, parties, account/invoice numbers, due dates, status (paid/outstanding)
- **medical**: patient identifiers (flag PII!), diagnoses, medications, dosages, dates of service, providers

If no domain hint is provided AND the corpus is mixed, skip domain enrichment and use the generic schema. **Domain templates are additive — they don't replace the standard Purpose/Key content/etc. sections.**

### 0. Ask for anchors (before scanning)

Before any scan, ask the user a single brief question: "Any 2–5 anchor questions you want this pack to answer with HIGH confidence? (Skip = generic build.)" Wait for reply. If skipped, proceed without anchors. If provided, store them in `.context/ANCHORS.md` and use them to:
- Bias deep-tier classification toward files relevant to the anchors
- Drive the verification anchor-answerability check (step 6)
- Seed the self-test question pool (step 5.6)

Skip this step in `diff` and `ask` modes.

### 1. Inventory

- Walk the folder. Skip `.context/`, `.git/`, `node_modules/`, `.venv/`, `dist/`, `build/`, `.DS_Store`, true binaries (`.zip .mp4 .so .dylib .exe .png .jpg`). PDFs are readable — include them.
- For each file: compute SHA-256, size, mtime. Build candidate manifest.
- **Hard cap: 200 files.** If over, stop and report the count + top directories by file count. Ask user to narrow scope.

### 2. Cache check

- If `.context/manifest.json` exists AND its prior run recorded `coverage: 100`:
  - Diff old vs new manifest. Work set = added + changed files.
  - If work set empty → print "no changes since <timestamp>", exit.
- Else: work set = all files. (Partial prior runs do not earn cache credit.)

### 2.5. PDF page-count safety (mandatory for any PDF in the work set)

PDFs require an extra step to prevent silent truncation:

1. **Probe page count first.** For each PDF, run `pdfinfo <file> | grep Pages` (or equivalent) to get `pages_total`. If `pdfinfo` is unavailable, install poppler-utils first.
2. **Always pass `pages` to Read.** Never call Read on a PDF without an explicit `pages` parameter. Chunk PDFs >10 pages into `pages: "1-10"`, `pages: "11-20"`, etc. — Read's hard cap is 20 pages per call.
3. **End-of-document signature check.** After reading the last chunk, verify the file ends with a recognizable terminal marker: signature line, "Ende"/"Schluss", page footer like `N/N`, signature image, or known boilerplate (e.g. Rechtsmittelbelehrung for German court orders). If absent, mark `truncated_read: true` in manifest and add a Gap entry.
4. **Record both counts in the per-file note frontmatter and manifest:**
   - `pages_total: N` (from probe)
   - `pages_read: M` (from chunks combined)
5. **Coverage gate**: `coverage: 100` is impossible unless `pages_read == pages_total` for every PDF in the corpus.

### 3. Read + extract via subagents

For corpora >30 files, **delegate extraction to `Explore` subagents** to keep raw file content out of the main context.

- Partition the work set into batches of ~25 files each.
- Spawn one `Explore` subagent per batch in parallel (single message, multiple tool calls).
- Each subagent's prompt: "Read these N files. For each, write `<.context/files/<relpath>.md>` following the schema in `~/.claude/skills/context-pack/SKILL.md` step 4. Do BOTH passes (extract + self-verify). Return only: list of files written, list of citation failures, list of files you could not read."
- Subagents must Read+Write only — do not let them modify source files.
- Main agent then runs steps 5–8 on the resulting notes (cross-index, verification, INDEX, manifest).

For ≤30 files, do it inline with parallel Read batches (~20 per batch). Use Read tool, not Bash.

For files >2000 lines, subagent reads in chunks; note truncation in the per-file note.

### 3.5. Effort weighting

Before extraction, classify each file by depth:

- **shallow** (≤30 lines, or pure config/lockfile/data): 1-line Purpose, list key keys/values, skip Key content section. ~10% effort.
- **standard** (30–500 lines, ordinary source/doc): full schema below. ~70% effort.
- **deep** (>500 lines, OR appears in ≥5 inbound refs after pass 1, OR matches an anchor question): full schema + extra Key content density (one bullet per logical block, not per file). ~20% effort.

Inbound-ref count requires a second pass — initial classification is by size; promote to deep after cross-index reveals hubs, then re-extract those notes only.

### 4. Per-file extraction (two-pass)

**Entity types.** During Pass A, classify each item in Key content as one of these types and tag it. Choose the type set that fits the corpus (declare it in INDEX.md):

- **Code corpus**: `module | symbol | test | config | route | schema`
- **Docs corpus**: `spec | decision | example | reference | requirement`
- **Mixed/text**: `entity | concept | claim | reference | example`

Tag format: `(type:L<a>-L<b>) <name>: <one line>`. Forces consistency, makes ask-mode much sharper.

**Pass A — extract.** For each scanned file, write `.context/files/<relpath>.md`:

```markdown
---
path: <relpath>
sha256: <hash>
size: <bytes>
lines: <n>
scanned_at: <iso8601>
---

## Purpose (L<a>-L<b>)
<1–2 sentences — what this file is for. Cite the line range that grounds this.>

## Key content
- (L<a>-L<b>) <symbol/section/heading>: <one line>
- ...

## References out
- (L<n>) → <other file or external resource>

## Does NOT cover
- <topics a reader might expect here but aren't actually in the file>

## Open questions
- <anything ambiguous, contradictory, missing context, or that you're <90% confident about>

## TODO/FIXME/HACK
- L<n>: <verbatim line>

## Confidence
- HIGH / MEDIUM / LOW
- Reason: <what would change this — e.g. "file references config.yaml not in corpus">
```

**Citation rule (mandatory):** every factual claim in Purpose / Key content / References must carry a citation pointing to the source location that justifies it. Format depends on file type:
- Text/code/markdown with stable line numbers: `(L<a>-L<b>)`
- PDFs / paginated docs: **`(p<n> ¶<m>)` or `(p<n> §<id>)`** — page + paragraph index OR page + section anchor (e.g. `§I.b`, `§Tatbestand`). Bare `(p<n>)` is allowed only for documents <30 lines per page.
- Spreadsheets: `(<sheet>!<range>)`

Sub-page anchors are mandatory for any PDF page with >30 lines — otherwise grep-back is too coarse to verify a specific claim. Use the document's own structure markers (Roman numerals, section headings, numbered paragraphs) when available; fall back to ¶<m> counted from top of page.

Claims without citations are forbidden — if you can't cite it, move it to Open questions.

**Pass B — self-verify.** After writing the note, re-read the cited line ranges and confirm each claim is actually supported. Any claim that fails verification → demote to Open questions or delete. Set Confidence based on verification result, not on writing fluency.

Keep notes terse. The note is a pointer, not a copy. Citations are not optional.

### 5. Cross-index + contradiction pass

After all per-file notes are written, build:

- **Inbound reference map**: which files mention each file (by name or path).
- **Orphans**: files with zero inbound references.
- **Hot files**: top 10 by inbound count.
- **Dangling references**: names mentioned in text that don't resolve to any file in the corpus or a known external.
- **Contradictions**: scan per-file notes for claims about the same entity/topic that disagree. Write `.context/CONFLICTS.md` listing each pair with citations (file + line range on both sides). Empty file is fine — but the pass must run.
- **Glossary**: terms/acronyms/entities appearing in ≥3 files. Write `.context/GLOSSARY.md` with one-line definitions, each cited.

### 5.5. Adversarial verification (red team pass)

**Mandatory for corpora ≥10 files.** For corpora <10 files, run an inline skeptical re-read instead (cheaper, less independent — must be flagged in RED_TEAM.md as a deviation). For ≥10 files, spawn one or more *separate* `Explore` subagents — different from the agents that wrote the notes. Their job is to disprove claims, not confirm them.

Per batch (~25 notes per agent), prompt: "You are a skeptical reviewer. For each per-file note in this batch, read the source file and try to find: (a) any claim NOT supported by the cited line range, (b) any claim that's technically true but misleading, (c) any important content in the source that the note OMITS. Output a list per note: `<note path>: PASS` or `<note path>: FAIL — <specific issue> (src L<a>-L<b>)`."

Aggregate findings into `.context/RED_TEAM.md`:

```markdown
# Red Team Findings

## Summary
- Notes audited: <n>
- Pass: <n>
- Fail: <n>
- Pass rate: <%>

## Failures
- [<note>](.context/files/<path>.md) — <issue> — src L<a>-L<b>
- ...

## Omissions
- [<note>](.context/files/<path>.md) — missing: <topic from source>
```

Notes that fail are demoted to LOW confidence and the issue is appended to their Open questions. The original extracting agent must NOT do this pass — confirmation bias kills its value.

### 5.6. Self-test calibration

After red team, generate 10–20 questions by sampling Key content claims across the corpus (weight toward hot files and anchor questions if provided). Then in a separate subagent, answer each question using ONLY `.context/` (no source reads). Then in another subagent, grade each answer by reading the cited source line ranges.

Output `.context/SELF_TEST.md`:

```markdown
# Self-Test Calibration

Sample size: <n>
Pass rate: <%>

## Results
- Q: <question>
  A: <answer from notes>
  Grade: PASS | FAIL — <reason>
  Source check: src L<a>-L<b>
```

Self-test pass rate is reported in INDEX.md as the empirical accuracy number — it overrides any self-reported confidence. **A pack with self-test pass rate <90% cannot claim `coverage: 100`.**

### 6. Verification (mandatory — do not skip)

Fill this checklist explicitly in INDEX.md. Each line gets ✓ or ✗ with evidence:

- [ ] Inventory count matches per-file note count
- [ ] Every file in manifest has a corresponding `.context/files/<relpath>.md`
- [ ] Every per-file note has Pass A + Pass B completed
- [ ] Every claim in Purpose / Key content / References carries a `(L<a>-L<b>)` citation
- [ ] **Spot-check audit**: pick a random 10% sample of cited claims, re-read the source line ranges, confirm each claim is supported. Report sample size, pass rate, and any failures by name.
- [ ] Every TODO/FIXME/HACK in source appears in a per-file note (grep source for `TODO|FIXME|HACK|XXX` and cross-check)
- [ ] Every cross-reference resolves (or is listed under "dangling references")
- [ ] Every file mentioned in docs/READMEs exists in the corpus (or is flagged missing)
- [ ] CONFLICTS.md, GLOSSARY.md, RED_TEAM.md, SELF_TEST.md exist (may be empty, but must exist)
- [ ] Red team pass executed by *different* subagents than the extractors. Failures demoted to LOW and listed.
- [ ] Self-test calibration run. Empirical pass rate reported.
- [ ] If anchor questions provided: each anchor answered with HIGH confidence using only `.context/`. Unanswerable anchors → Gaps.
- [ ] No per-file note has empty Purpose, Confidence, or "Does NOT cover"
- [ ] Coverage gate: spot-check pass rate ≥95% AND red team pass rate ≥90% AND self-test pass rate ≥90% AND every anchor answered AND zero LOW-confidence files unflagged in Gaps → only then mark `coverage: 100`

If any check fails: do not mark `coverage: 100`. List the failures in INDEX.md under "Gaps".

### 7. INDEX.md structure

```markdown
# Context Pack: <folder name>

Generated: <iso8601>
Files: <n> | Coverage: <100|partial>

## Summary
<3–5 sentences — what this corpus is, who uses it, what it's about>

## Map
<tree or grouped list — by directory or by inferred role>

## Hot files
1. [<path>](.context/files/<path>.md) — <inbound count> refs — <one line>
...

## Orphans (no inbound references)
- <path> — <one line>

## Open questions
- <aggregated from per-file notes>

## Dangling references
- <name> — mentioned in <file> but not resolved

## Gaps
- <any failed verification check, or "none">

## Verification
- [✓/✗] <each check from step 6>

## Confidence
- Overall: HIGH/MEDIUM/LOW
- Reason: <what would change this>
```

### 8. Update manifest

Write `manifest.json`:

```json
{
  "run": {
    "started_at": "...",
    "finished_at": "...",
    "coverage": 100,
    "files_scanned": <n>,
    "files_skipped_cached": <n>
  },
  "files": [
    {"path": "...", "sha256": "...", "size": ..., "mtime": "...", "scanned_at": "..."}
  ]
}
```

Set `coverage: 100` only if every verification check passed. Otherwise `coverage: partial` + `gaps: [...]`.

### 8.5. Digest pass (mandatory — the actionable layer)

After verification passes, write `.context/DIGEST.md` — a one-page TL;DR distinct from INDEX.md (which is the structural map). DIGEST is what the user actually reads to decide what to do next.

Adapt content to the corpus, but always include:

```markdown
# Digest: <folder name>

## TL;DR (3 sentences max)
<plain-English summary of what this corpus IS and where things stand right now>

## Key numbers
- <aggregated totals — sum of amounts, count of items, percentages>
- <example for finance: "Outstanding: 12.952 EUR across 5 invoices, 3 distinct AZ">

## Timeline
- <date>: <event> — [<source>](.context/files/<path>.md)
- ... chronological, only the events that matter

## Decision points / open actions
- <unresolved question or pending action>: who/what/by-when if known
- <example: "Schriftliche Berufungsurteils-Begründung — awaited">

## Risks / red flags
- <anything in the corpus that suggests caution, deadline, or surprise>

## Next likely steps
- <2-4 concrete things the user might want to do next given this corpus>
```

Rules:
- Keep DIGEST.md to one screen. If it's longer, it's failing its purpose.
- Every aggregated number must be derivable from the per-file notes — if a sum can't be cited from at least 3 underlying notes, omit it.
- Don't repeat content from INDEX.md verbatim. INDEX is structural, DIGEST is actionable.
- Do not invent action items. Only surface those grounded in the corpus.

### 9. Ask mode

When invoked as `ask` with a question:

- Require `.context/manifest.json` with `coverage: 100`. If missing or partial, refuse and tell the user to run build first.
- Read `INDEX.md`, `GLOSSARY.md`, `CONFLICTS.md`, and per-file notes whose Purpose or Key content matches the question (grep the notes, not source files).
- If the cached notes don't contain the answer, fall back to reading the cited source line ranges directly — never guess.
- Answer with citations to per-file notes AND the original source line ranges (`[<file>](.context/files/<path>.md) → src L<a>-L<b>`).
- If you cannot answer with ≥HIGH confidence from cached material, say so explicitly and list what's missing.

### 10. Diff mode

When invoked on a corpus that already has `.context/` from a prior 100% run:

- Compute current vs cached manifest. Identify added / modified / deleted files.
- Run steps 3–6 on added + modified only. Delete per-file notes for removed source files. Update `manifest.json`.
- **Drift detection**: for each modified file, identify per-file notes in OTHER files whose `References out` pointed at the changed line ranges. Flag those notes in `CHANGES.md` under "Possibly stale" — their citations may now be wrong. The skill does not auto-rewrite them; user decides whether to re-run those notes.
- Write `.context/CHANGES.md`:

```markdown
# Changes since <prior_run_iso8601>

## Added (<n>)
- <path> — <one-line purpose with citation>

## Modified (<n>)
- <path> — <what changed> — was: <old summary>, now: <new summary>

## Deleted (<n>)
- <path> — <was: old purpose>

## Impact
- Hot files affected: <list>
- New orphans: <list>
- New dangling references: <list>
- New conflicts in CONFLICTS.md: <count>

## Re-verification
- [✓/✗] All modified files re-passed verification
- [✓/✗] Cross-index rebuilt — old refs to deleted files cleaned up
```

CHANGES.md is overwritten on each diff run (it represents the latest delta only). Prior change history lives in `git log` of `.context/`.

### 11. Link mode (cross-pack)

When invoked as `/context-pack link <folder-a> <folder-b>`:

- Both folders must have `.context/manifest.json` with `coverage: 100`. Refuse if either is missing or partial.
- Read both INDEX.md + GLOSSARY.md.
- Find shared entities/terms (case-insensitive name match in glossaries; exact match in symbol-typed entries).
- Find cross-pack references: any `References out` in pack A that mentions a file path or entity present in pack B (or vice versa).
- Write `<folder-a>/.context/LINKS.md` (and a mirror in folder-b):

```markdown
# Cross-pack links: <folder-a> ↔ <folder-b>

## Shared entities
- <name>: A → [<note>](.context/files/<path>.md), B → <abs-path-to-other-pack>/files/<path>.md

## Cross-references
- A's [<file>](.context/files/<path>.md) refs B's <file>
- ...

## Conflicts across packs
- A claims <X> (src L<a>-L<b>); B claims <Y> (src L<c>-L<d>) — disagreement on <topic>
```

Link mode is idempotent — re-running updates LINKS.md but never touches per-file notes.

### 12. JSON sidecar

Alongside `INDEX.md`, write `.context/index.json` for programmatic consumption:

```json
{
  "generated_at": "...",
  "coverage": 100,
  "files_total": <n>,
  "metrics": {
    "spot_check_pass_rate": <%>,
    "red_team_pass_rate": <%>,
    "self_test_pass_rate": <%>
  },
  "anchors": [{"question": "...", "answered": true, "confidence": "HIGH"}],
  "hot_files": [{"path": "...", "inbound": <n>}],
  "orphans": ["..."],
  "open_questions_count": <n>,
  "conflicts_count": <n>,
  "gaps": ["..."]
}
```

This file is the canonical machine-readable summary; INDEX.md is the human-readable view of the same data.

## Reporting back to user

Terse. After completion, output:

```
Scanned: <n> files (<n> new/changed, <n> from cache)
Coverage: 100% | partial (<reason>)
Index: <folder>/.context/INDEX.md
Hot files: <top 3 paths>
Open questions: <count>
Gaps: <count or "none">
```

Then stop. The user reads INDEX.md.

## Rules

- **Never** delete or modify source files.
- **Never** write outside `.context/`.
- **Never** claim 100% coverage if any verification check failed.
- **Never** skip the verification checklist to save tokens.
- **Always** preserve per-file notes for unchanged files across runs (don't rewrite if hash matches).
- If the folder has >200 files, stop and ask the user to narrow scope. Do not silently sample.
- If a file is binary or unreadable, note it in `manifest.json` with `skipped: "binary"` or `skipped: "<error>"` and exclude it from coverage denominator.
