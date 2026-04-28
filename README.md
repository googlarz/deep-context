# deep-context

> Build verified, comprehensive context packs from a folder — with citations, contradiction detection, self-test calibration, and a one-page actionable digest. A [Claude Code](https://claude.com/claude-code) skill.

## What it does

Point `deep-context` at a folder (≤200 text files). It reads every file, extracts structured per-file notes with **mandatory source citations**, cross-indexes references, detects contradictions, runs a red-team pass and self-test calibration, and produces:

- **`INDEX.md`** — structural map: hot files, orphans, dangling references, verification report.
- **`DIGEST.md`** — one-page TL;DR: aggregated numbers, timeline, decision points, risks, next steps. **This is what you actually read.**
- **Per-file notes** with line- or page-level citations to the source.
- **`CONFLICTS.md`** — claims that disagree across documents.
- **`GLOSSARY.md`** — terms/entities appearing in ≥3 files.
- **`RED_TEAM.md`** — independent skeptical re-read findings.
- **`SELF_TEST.md`** — empirical accuracy on 10–20 self-generated questions.
- **`manifest.json`** + **`index.json`** — machine-readable summaries with hashes and metrics.

## Why

Most context-loading is RAG: retrieve at query time, rediscover knowledge each call. `deep-context` does the opposite — it **compiles** a persistent, citation-grounded artifact once, then queries it.

It's designed for the cases where "scan everything thoroughly" is actually tractable: a focused dossier, a small repo, a research folder, a dossier of contracts. At ≤200 files, full reads are cheap; the value is in **verified completeness** and an **actionable digest**, not retrieval latency.

## When to use this over built-in context tools

The default context-loading paths in agents (raw `Read`, glob+grep, naive RAG retrieval, "just paste it in") are usually fine for casual exploration — and they're cheaper. But they share a failure mode that matters when the corpus is consequential: **you don't know what was missed, what was hallucinated, or whether the answer the agent gave you is actually grounded in the source.**

Use `deep-context` when the cost of a confidently-wrong answer is higher than the cost of extra tokens:

- **Legal / contracts / regulatory** — a missed clause or a hallucinated deadline is expensive
- **Financial dossiers** — an aggregated total that secretly double-counts or skips a currency conversion is worse than no total at all
- **Medical / clinical** — built-ins have no PHI awareness; this skill has explicit (best-effort) redaction policy
- **Audit, due diligence, M&A** — you need an artifact you can defend, with citations
- **Research synthesis where the conclusions will be cited** — you need to know what the corpus actually says vs what the agent imagined
- **Any "trust me" output you'd be embarrassed to find was wrong**

What you're paying for vs the built-ins:
- **Mandatory source citations on every claim** — no claim without a verifiable pointer
- **Independent red-team pass** — a separate agent tries to *disprove* claims, not confirm them
- **Source-derived omission probe** — a fourth audit direction that asks "what's NOT in the notes that should be?" — closes the structural blind spot in note-only verification
- **Hostile-content guard** — corpus content cannot inject instructions into the extractor
- **Symlink/realpath boundary** — scan stays inside the folder you pointed at, period
- **Coverage gate that admits uncertainty** — `coverage: partial` is the honest default when something's incomplete; `coverage: 100` is gated on multiple measured pass rates

The tradeoff is straightforward: roughly 3–10× the tokens of a casual scan, in exchange for an artifact that tells you what it knows, what it doesn't, and why each claim should be believed. If the corpus doesn't matter, don't use this. If it does, the extra cost is the cheapest part of the workflow.

## Modes

| Mode | Trigger | Action |
|------|---------|--------|
| **build** | folder + "scan/ingest/build context" | Full workflow — reads every file, writes the pack |
| **build with anchors** | folder + 3–5 anchor questions | Build, then verify each anchor is HIGH-confidence answerable |
| **ask** | existing `.context/` + a question | Answer from cached notes with dual citations |
| **diff** | rebuild on changed files only | Cache hit + writes `CHANGES.md`; runs drift detection |
| **link** | two folders | Cross-link two packs with shared entities + cross-pack conflicts |

## Quality gates (`coverage: 100` requires all of these)

- Every claim carries a source citation (`L<a>-L<b>` for text, `(p<n> ¶<m>)` for PDFs).
- Spot-check audit ≥ 95% pass rate
- Red-team independent re-read ≥ 90% pass rate (subagent for ≥10 files; inline for smaller corpora, flagged)
- Self-test calibration ≥ 90% empirical accuracy
- Every anchor question answerable from notes alone
- For PDFs: `pages_read == pages_total`, end-of-document signature verified
- Zero LOW-confidence files unflagged in Gaps

If any gate fails, coverage is `partial` — gaps explicitly listed.

## Installation

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/googlarz/deep-context.git ~/.claude/skills/deep-context
```

Or symlink if you want to track upstream changes:

```bash
git clone https://github.com/googlarz/deep-context.git
ln -s "$(pwd)/deep-context" ~/.claude/skills/deep-context
```

After installation, the skill is discoverable in any Claude Code session. Invoke it by passing a folder path with phrasing like "scan this folder" or "build context for `<path>`".

## Usage

```
> Build context for /path/to/folder
```

Optional flags:

- `--domain=<legal|code|sales|research|finance|medical>` — load a domain-specific extraction template.
- Anchor questions — provide 2–5 questions and the pack will be verified to answer each.

## Output structure

```
<folder>/.context/
├── INDEX.md              # structural map + verification report
├── DIGEST.md             # one-page actionable TL;DR
├── ANCHORS.md            # user-supplied anchor questions
├── CHANGES.md            # diff-mode output (only if rebuilding)
├── CONFLICTS.md          # contradictions across documents
├── GLOSSARY.md           # cross-document terms
├── RED_TEAM.md           # adversarial verification
├── SELF_TEST.md          # empirical accuracy calibration
├── LINKS.md              # cross-pack links (link mode only)
├── manifest.json         # SHA-256 hashes + run metadata
├── index.json            # machine-readable summary
└── files/
    └── <relpath>.md      # one note per source file, mirrors source tree
```

## Limitations

- **Hard cap: 200 files.** Beyond that, the verification ceremony stops earning its cost; use a retrieval-based approach instead.
- **PDFs require `pdfinfo`** (poppler-utils) for page-count probing. Install via `brew install poppler` on macOS.
- **Coverage `partial` is normal** when the corpus references documents not in the folder. The skill flags those — it doesn't hallucinate.
- **Skill is a workflow specification**, not code. The Claude Code agent executes the steps; quality depends on the model running it.

## Provenance

Built incrementally across one long session, refined through a real run on a 7-PDF German legal corpus. Lessons from that run drove four real spec fixes that v1 missed (PDF skip rule, citation format, red-team threshold, anchor prompt). Plus a follow-up round adding sub-page anchors, page-count safety, domain hints, and the digest pass.

The spec is at the point of diminishing returns — next improvements should come from running on different corpus types (code, transcripts, research notes), not more design.

## License

MIT — see [LICENSE](LICENSE).
