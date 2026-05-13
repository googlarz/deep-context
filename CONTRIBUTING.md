# Contributing to deep-context

A Claude Code skill that builds verified, comprehensive context packs from a folder — with citations, contradiction detection, and self-test calibration. Contributions welcome.

## Before you start

- Check [open issues](https://github.com/googlarz/deep-context/issues) and [discussions](https://github.com/googlarz/deep-context/discussions)
- For workflow changes, open a discussion first

## What to contribute

- **Workflow improvements** — better steps, stronger verification gates
- **Edge cases** — scenarios where context extraction goes wrong (contradictions, large files, binary assets)
- **README examples** — concrete before/after prompts
- **Bug reports** — include the folder structure and prompt that triggered the issue

## Submitting a PR

1. Fork → branch from `main`
2. Changes to `SKILL.md` must preserve the existing section structure
3. Every new step must be verifiable — an agent must be able to check it passed
4. Open the PR with a description of what problem it solves and how you tested it

## Skill format

Follow the [agent-skills anatomy](https://github.com/addyosmani/agent-skills): Overview → When to Use → Process → Common Rationalizations → Red Flags → Verification. Don't remove sections.
