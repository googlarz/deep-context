# deep-context

> You have years of notes, journals, contracts, or code. What do they actually say? `deep-context` reads every file, extracts verified, cited notes, detects contradictions, and produces a one-page digest you can trust. A [Claude Code](https://claude.com/claude-code) skill.

## The problem

You have a folder that matters — years of personal notes, a research archive, a dossier of contracts, a codebase you inherited. You ask Claude to "read everything and tell me what it says." Claude reads some of it, tells you something confident, and you have no idea whether it's accurate, what it missed, or whether two files contradict each other.

`deep-context` fixes this. It reads every file, extracts structured notes with **mandatory source citations**, detects contradictions across documents, runs an independent adversarial check, and tells you explicitly what it found, what it didn't, and why each claim should be believed.

## What it produces

```
<folder>/.context/
├── DIGEST.md         ← read this first: one-page TL;DR with aggregated numbers, timeline, risks
├── INDEX.md          ← structural map: hot files, orphans, contradictions, verification report
├── CONFLICTS.md      ← claims that disagree across documents
├── GLOSSARY.md       ← terms and entities appearing in ≥3 files
├── RED_TEAM.md       ← independent adversarial re-read findings
├── SELF_TEST.md      ← empirical accuracy: questions answered from notes alone, graded against source
├── manifest.json     ← SHA-256 hashes + coverage status + quality metrics
└── files/
    └── <relpath>.md  ← one cited note per source file
```

## Who uses this

### Personal knowledge and journals

You've been keeping notes for years — journals, daily logs, research notes, meeting summaries, an Obsidian vault, exported Notion pages. You want to know: what patterns emerge? What decisions have I revisited? What contradicts what I said last year?

```
> Build context for ~/notes/2020-2024
```

deep-context reads every file, cross-indexes references between entries, flags where your views on a topic shifted over time (logged as contradictions), and produces a timeline and digest of what your notes actually contain — cited, so you can verify any claim against the source entry.

### Contracts, legal, and financial documents

A missed clause or a hallucinated deadline is expensive. deep-context was refined on a real German legal corpus and handles PDFs with sub-page citations (`p3 ¶2`), cross-document entity tracking, and semantic role/obligation analysis (catches "Party A is licensor in the contract but payer in the invoice").

```
> Build context for ~/dossier --domain=legal
```

### Research and literature

A folder of papers, transcripts, or reports. What are the key findings? Which papers contradict each other? What terms does the literature use that aren't in your own notes?

```
> Build context for ~/research/lithium-batteries --domain=research
```

### Code you didn't write

Point it at a repo or module directory. It maps inbound/outbound dependencies, flags orphaned files, extracts TODOs and open questions, and tells you what the code does — not what it says it does.

```
> Build context for ~/projects/legacy-api/src
```

## Why not just ask Claude to read the folder?

That works fine for casual exploration. The failure mode is invisible: you get a confident answer, you don't know what was missed, and you can't verify any specific claim without re-reading the source yourself.

Use deep-context when the cost of a confidently-wrong answer is higher than the cost of extra tokens:

- Every claim has a verifiable source citation — no claim without a pointer
- An independent subagent tries to *disprove* notes, not confirm them
- A source-derived omission probe asks "what's in the raw files that the notes don't cover?"
- `coverage: partial` is the honest default when something's incomplete — the tool tells you what it missed rather than pretending it got everything

The tradeoff: roughly 3–10× the tokens of a casual read, in exchange for an artifact that tells you what it knows, what it doesn't, and why each claim should be believed.

## Modes

| Mode | Trigger | What happens |
|---|---|---|
| **build** | `Build context for <folder>` | Full run: reads every file, extracts notes, verifies, writes pack |
| **build with anchors** | folder + 2–5 questions | Build, then verify the pack can answer each question at HIGH confidence |
| **ask** | `Ask <folder>: <question>` | Answer from cached notes with dual citations (note + source location) |
| **diff** | any rebuild after changes | Re-scans changed files only; writes `CHANGES.md`; flags dependent notes for refresh |
| **link** | `Link <folder-a> <folder-b>` | Cross-links two packs — shared entities, cross-pack contradictions |
| **serve** | `Serve <folder>` | Generates `server.py`: a local MCP server other Claude sessions can query |
| **watch** | `--watch` on any build | Polls for changes and auto-triggers diff |

## Quality gates

`coverage: 100` requires all of these. If any fails, the pack reports `coverage: partial` and lists every gap explicitly.

- Every claim cites a source (`L12-L18` for text, `p3 ¶2` for PDFs, `Sheet1!B2` for spreadsheets)
- Spot-check audit: ≥95% of sampled claims verified against source, CI lower bound ≥85%
- Red-team pass: ≥90% of notes survive adversarial re-read
- Self-test accuracy: ≥90% on questions answered from notes alone (factual + cross-file synthesis, reported separately)
- Omission probe: ≥90% of raw source slices — prioritizing dates, amounts, names, decisions — are covered in notes
- For PDFs: `pages_read == pages_total`, end-of-document signature verified
- DIGEST.md: all 6 sections present and non-empty

## Installation

```bash
git clone https://github.com/googlarz/deep-context.git ~/.claude/skills/deep-context
```

After that, the skill is available in any Claude Code session. Trigger it naturally:

```
> Scan this folder and tell me what's in it: ~/Documents/contracts/2024
> Build context for the notes I took this year
> What does my research folder actually say about sleep and HRV?
```

**PDFs require `pdfinfo`:**
```bash
brew install poppler   # macOS
apt install poppler-utils  # Linux
```

## Flags

- `--domain=<legal|code|sales|research|finance|medical>` — loads domain-specific extraction (structured fields for contracts, lab values, API surfaces, etc.)
- `--yes` — skip the pre-build cost estimate prompt (for scripted or CI use)
- `--no-anchors` — skip the anchor question prompt
- `--watch` — poll for changes after build and auto-diff
- `--partial-ok` — allow `ask` mode on partial-coverage packs (answers capped at LOW confidence, with explicit warning)

## The 200-file limit

The verification ceremony — adversarial re-read, omission probe, self-test calibration — earns its cost at ≤200 files. Above that, the sample rates become statistically thin and the token cost becomes prohibitive.

**If your corpus is larger:** use the folder-of-folders approach. Split into thematic subdirectories (by year, by topic, by project), run deep-context on each, then use `link` mode to connect the packs. The linked packs share a GLOSSARY and CONFLICTS.md that spans all of them.

A native hierarchical mode (sub-packs → meta-pack) is on the roadmap.

## The `.context/` format

The output directory follows an open schema defined in [`CONTEXT_SCHEMA.md`](CONTEXT_SCHEMA.md). Any tool that writes compliant `.context/` output — a health tracking skill, a legal MCP, a code-review agent — produces artifacts that deep-context can verify and query. The format is designed to be the layer between "raw files" and "trusted answers," independent of how the notes were produced.

## Limitations

- **Hard cap: 200 files.** See above for workaround.
- **PDFs require `pdfinfo`** (poppler-utils). Without it, page-count safety checks cannot run.
- **Coverage `partial` is normal** when the corpus references external documents. The skill flags what's missing — it doesn't hallucinate.
- **This is a workflow specification**, not compiled code. Claude Code executes the steps; quality scales with the model running it.
- **Medical-mode redaction is best-effort**, not HIPAA-grade. See the SKILL.md disclaimer before using with regulated health data.

## Provenance

Built over a series of sessions, refined through a real run on a 7-PDF German legal corpus. Four rounds of adversarial review (Codex) drove the major spec fixes: PDF sub-page citations, CI reporting on pass rates, atomic medical scrub, omission probe bias toward high-stakes content, note integrity hashing.

## License

[CC BY-NC 4.0](LICENSE) — free for personal and open-source use with attribution.

**Commercial use** (consulting, client work, commercial products): free, but requires a brief notification. Email [googlarz@gmail.com](mailto:googlarz@gmail.com) with your company name and use case — a commercial license is granted by return. No payment, just acknowledgment.
