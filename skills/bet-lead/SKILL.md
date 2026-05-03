---
name: bet-lead
description: >
  Guide the design and delivery of Wordloop bets — from problem statement through
  pitch, TDD architecture, milestones, slices, and delivery. Use this skill
  whenever a user is writing or reviewing a problem statement, pitch, TDD,
  UI design, data flow, contract, schema, milestone, or slice document. Also use
  when scaffolding a new bet, running bet progress tests, reviewing bet documentation
  quality, planning milestone decomposition, designing domain slices, or asking how
  the bet process works — even if they don't explicitly mention "bet-lead." Load it
  before answering questions about bet lifecycle, TDD structure, or slice design.
---

# Bet Lead

Execution router for Wordloop bet design and delivery. All bet knowledge lives in the `work` doc and its linked templates — this skill routes you to the right section and enforces agent-specific behaviour.

## Operating Contract

1. Start from the user's position in the bet lifecycle. Load only the doc section that matches the current phase.
2. Use `./dev` CLI as the source of truth for scaffolding. Never hand-create files the CLI would scaffold.
3. Validate documents against their upstream inputs using the Document Chain Integrity table in the `work` doc.
4. Prefer the Wordloop docs MCP tools when available; fall back to local MDX under `services/wordloop-docs/content/docs/work/`.

## Context Routing

Load the `work` doc, then navigate to the relevant section. For most tasks, one section is enough.

| Task | Doc section to read | Also read |
|---|---|---|
| Bet lifecycle overview | `work` — full page | — |
| Writing or reviewing a problem statement | `work` § Problem Statement + its review checklist | Template: `work/_template/problem-statement.mdx` |
| Writing or reviewing a pitch | `work` § Pitch + its review checklist | Template: `work/_template/pitch.mdx` |
| Writing or reviewing a UI design | `work` § UI Design + its review checklist | Template: `work/_template/tdd/ui-design.mdx` |
| Writing or reviewing a data flow | `work` § Data Flows + its review checklist | Template: `work/_template/tdd/data-flow.mdx` |
| Designing or reviewing contracts | `work` § Contracts & Schemas + review checklists | Template: `work/_template/.scaffolds/contract-{rest,websocket,pubsub}.mdx` |
| Designing or reviewing schemas | `work` § Contracts & Schemas + schema review checklist | Template: `work/_template/.scaffolds/schema.mdx` |
| Decomposing into milestones | `work` § Designing milestones + review checklist | Template: `work/_template/.scaffolds/milestone.mdx` |
| Writing or reviewing slices | `work` § Anatomy of a well-written slice + review checklist | Template: `work/_template/.scaffolds/slice.mdx` |
| Reviewing document chain integrity | `work` § Document Chain Integrity | Upstream doc for the document being reviewed |
| Running or understanding bet tests | `work` § Bet Test Suite | `tests/bets/README.md` |
| Completing or archiving a bet | `work` § Completing a Slice + § Bet Operations | `principles/foundations/testing` |
| Contract design principles | `principles/system-design/api-design` | `principles/system-design/real-time` for WebSocket/streaming |

## MCP Tools

When the Wordloop docs MCP server is available, use these tools to inspect bet state without reading local files:

| Tool | Use |
|---|---|
| `list_bets(status)` | List bets by lifecycle stage: `problem-statement`, `pitch`, `active`, `delivered`, `all` |
| `get_bet_manifest(slug)` | Full bet overview: pitch + TDD directory tree + status |
| `get_bet_tdd_element(slug, element_path)` | Fetch a specific TDD document (e.g., `milestones/01-upload/slice-core-upload.mdx`) |
| `get_page(slug)` | Read any docs page by slug (e.g., `work` for How We Work) |

## Safety Gates

- Do not hand-create files the CLI scaffolds. Verify commands against `./dev new --help`.
- Do not write implementation detail in design documents. Each document level has a clear boundary defined in the `work` doc.
- Do not skip lifecycle stages. Each stage earns its existence by building on the previous one.
- Do not slice horizontally. See the vertical slicing test in `work` § Anatomy of a well-written slice.
- Do not mark a slice complete without the full completion checklist in `work` § Completing a Slice.

## Hallucination Controls

- Check `./dev new --help` and `./dev test --help` for CLI commands before recommending them.
- Check the bet's current documents via MCP or local MDX before commenting on their content.
- Check template files under `services/wordloop-docs/content/docs/work/_template/` before advising on document sections.
- Check `tests/bets/<slug>/` for existing test files before advising on test structure.
- Label recommendations based on general product design knowledge (not Wordloop-specific patterns) as inferences.

## Output Expectations

- Scaffolding guidance includes the exact CLI command.
- Document reviews identify specific gaps against the Document Chain Integrity table — not generic writing feedback.
- Slice specifications follow the six-part anatomy defined in the `work` doc.
- Test guidance distinguishes between bet progress tests (temporary) and permanent service tests.
