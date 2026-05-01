---
name: technical-writer
description: >
  Engineer technical documentation as a first-class product feature using
  canonical docs plus agent-skill execution boundaries, dual-audience
  architecture (human + AI), Diátaxis information architecture,
  automation-first freshness governance, diagrams-as-code, and debt-aware
  architectural decisions. Use when creating or maintaining documentation
  principles, llms.txt indexes, Markdown exports, MCP documentation servers,
  skill-to-doc synchronization workflows, documentation health checks,
  stale-doc review policies, drift-correction workflows, C4 diagrams with
  Mermaid.js or D2, OpenAPI/AsyncAPI documentation, generated SDK/reference
  pipelines, Vale documentation linting, ADRs, documentation site
  information architecture, or writing style guides. Make sure to use this
  skill whenever a user mentions technical documentation, docs architecture,
  AI-native docs, human + AI documentation, llms.txt, MCP for documentation,
  docs and agent skills, skill drift, documentation freshness,
  documentation health checks, active docs, diagrams-as-code, C4 models,
  OpenAPI specs, SDK generation, Vale linting, ADRs, documentation quality,
  writing style guides, or documentation drift, even if they don't explicitly
  ask for a technical writer.
---

# Technical Writer

Documentation execution engineer for Wordloop's docs-as-product approach. This skill guides the engineering of technical documentation as a first-class product feature — canonical for humans and agents, active rather than passive, structured by reader intent, and governed by automation wherever facts can be checked mechanically.

## Operating Contract

1. **Separate knowledge from execution.** Durable engineering knowledge belongs in docs. Agent skills own triggering, context routing, tool use, safety gates, and verification steps.
2. **Design for two readers.** Every important page should work for a human in the docs UI and for an agent consuming Markdown, `llms.txt`, `llms-full.txt`, or MCP resources.
3. **Use Diátaxis to place content.** Tutorials teach, how-to guides solve, reference pages support lookup, and explanation pages build understanding. Do not mix jobs unless there is a deliberate reason.
4. **Make docs active.** Prefer pages with owner, freshness metadata, review cadence, source-of-truth boundaries, and health checks over passive prose.
5. **Automate first.** Use generation, link checks, frontmatter checks, stale-page checks, contract rendering, diagram checks, and skill-to-doc maps before relying on manual review.
6. **Control hallucination risk.** Verify standards, product versions, endpoint names, event names, commands, and architecture claims against local source files or official sources before presenting them as facts.

## Core Pillars

1. **Canonical Docs + Skill Execution** — The docs site is the canonical source for durable engineering knowledge. Skills are the agent execution layer. A skill should route an agent to canonical pages and define safe action, not become a shadow documentation site.

2. **Dual-Audience Architecture** — Every piece of documentation serves humans and AI agents. The structures that help humans scan and navigate also help agents retrieve and reason: explicit headings, semantic frontmatter, stable URLs, Markdown exports, `llms.txt`, and MCP resources.

3. **Diátaxis Information Architecture** — Organize content by user need. A how-to guide that tries to be a conceptual essay becomes weak for both humans and retrieval. A reference page that teaches a tutorial becomes hard to scan.

4. **Automation-First Governance** — Documentation quality is enforced through checks, not wishes. Human reviewers judge accuracy and usefulness; automation catches broken links, stale metadata, drift-prone generated references, and invalid skill-to-doc dependencies.

5. **Diagrams-as-Code** — Diagrams are text source committed to version control and reviewed alongside code. A diagram that cannot be diffed, linked, or drift-checked is supporting evidence at best, not canonical documentation.

6. **Debt-Aware Decisions** — Hard-to-reverse decisions are append-only ADRs with context, consequences, alternatives, and debt annotations. Do not rewrite accepted decisions to make history look cleaner.

## How to Use This Skill

Match the user's need to the smallest relevant reference set. Most tasks touch 1-2 references, not the whole library.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Docs/Skills Governance | `references/docs-skills-governance.md` | Designing the boundary between canonical docs and agent skills, creating skill-to-doc maps, writing freshness policies, or correcting docs/skills drift. |
| Feature Design Studio | `references/feature-design-studio.md` | Authoring or reviewing any Feature Design Studio document (specification, discovery, design, implementation, validation). |
| AI-Native Documentation | `references/ai-native-documentation.md` | Implementing `llms.txt`, Markdown exports, HTTP discovery headers, MCP documentation servers, or metadata for agent consumption. |
| Diagrams-as-Code | `references/diagrams-as-code.md` | Creating or reviewing system diagrams with Mermaid.js or D2, applying C4 hierarchy, or checking diagram drift. |
| API Documentation Ecosystems | `references/api-documentation-ecosystems.md` | Writing OpenAPI or AsyncAPI specifications, building API playgrounds, configuring SDK generation, or optimizing time-to-first-call. |
| Writing Style & Tone | `references/writing-style-and-tone.md` | Authoring or reviewing technical prose, enforcing active voice, using the Inverted Pyramid, or reducing cognitive weight. |
| Quality Governance | `references/quality-governance.md` | Configuring Vale, integrating docs checks into CI/CD, enforcing terminology, or auditing docs quality. |
| Information Architecture | `references/information-architecture.md` | Designing docs site structure, onboarding paths, expert navigation, search, or content reuse. |
| Document Templates | `references/document-templates.md` | Writing READMEs, changelogs, runbooks, migration guides, release notes, error catalogs, troubleshooting guides, glossaries, or CONTRIBUTING files. |
| Code Documentation | `references/code-documentation.md` | Applying the documentation trust hierarchy: types, structural metadata, naming, tests, comments, and prose docs. |
| Debt-Aware ADRs | `references/debt-aware-adrs.md` | Writing or reviewing ADRs, superseding decisions, or recording principal/interest/multiplier tradeoffs. |
| Antipatterns | `references/antipatterns.md` | Reviewing docs for drift, shadow docs, static artifacts, prompt-shaped docs, or misleading generated/reference boundaries. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/foundations/documentation` — Documentation philosophy and docs/skills boundaries.
- `guides/keep-docs-and-skills-in-sync` — Skill-to-doc synchronization workflow.
- `guides/correct-documentation-drift` — Drift triage and correction workflow.
- `operations/documentation-freshness` — Active-doc freshness policy and review cadence.
- `principles/ai-native/agent-native-systems` — Agent-readable interface requirements.
- `decisions/0005-docs-canonical-skills-execution-layer` — The accepted architecture decision for docs/skills boundary.

## Task Routing

- **Docs/skills boundary work** → Read documentation principle and ADR 0005. Load `references/docs-skills-governance.md`.
- **New docs page** → Read documentation principle. Load `references/information-architecture.md` for placement and `references/document-templates.md` for structure.
- **AI-native docs (llms.txt, MCP)** → Load `references/ai-native-documentation.md`. Read agent-native-systems principle.
- **Diagram creation/review** → Load `references/diagrams-as-code.md`. Check existing diagram conventions.
- **API/event documentation** → Load `references/api-documentation-ecosystems.md`. Check OpenAPI/AsyncAPI specs.
- **ADR authoring** → Load `references/debt-aware-adrs.md`. Read existing ADRs for conventions.
- **Docs quality/linting** → Load `references/quality-governance.md`. Check existing Vale config.
- **Writing style review** → Load `references/writing-style-and-tone.md`.
- **Drift correction** → Read drift-correction guide. Load `references/antipatterns.md`.
- **Feature Design Studio docs** → Load `references/feature-design-studio.md`.

## Safety Gates

- Do not duplicate durable engineering knowledge in skills. Skills route to docs; they do not replace them.
- Do not hand-edit generated reference docs (API, database, SDK). Update the source and regenerate.
- Do not edit accepted ADRs. Supersede them with a new ADR that references the original.
- Do not publish diagrams as static images when text-source alternatives (Mermaid, D2) exist. Static diagrams cannot be diffed or drift-checked.
- Run `./dev docs health` after any documentation, skill-map, or frontmatter changes.

## Hallucination Controls

Before presenting factual guidance:

- Check local source files for package versions, commands, ports, paths, and generated artifacts.
- Check OpenAPI/AsyncAPI specs for endpoint and event claims.
- Check ADRs before claiming why a system decision exists.
- Check active TDD docs before claiming delivery intent.
- Check official sources before naming a standard or external specification.
- Say when a claim is an inference, and state what evidence supports it.

## Output Expectations

Good technical-writing work should produce one or more of:

- A docs page with clear purpose, owner, freshness metadata, source-of-truth boundaries, and links to related pages.
- A workflow with concrete steps, verification commands, and anti-patterns.
- An ADR with context, decision, consequences, alternatives, debt annotation, and verification.
- A health-check or generation path that makes future drift harder.
- A skill update that keeps the SKILL body concise and moves durable knowledge to docs.

## Antipatterns

Reject these patterns:

- **Shadow docs in skills** — durable policy copied into SKILL.md instead of canonical docs.
- **Prompt-shaped docs** — human docs written like agent system prompts.
- **Passive docs** — pages without owner, freshness metadata, or review path.
- **Unverifiable claims** — version, standard, API, event, or command claims not checked against source.
- **Manual generated reference** — hand-written tables for data that can be generated from contracts or code.
- **Mutable ADRs** — editing accepted decisions instead of superseding them.
- **Static diagrams as truth** — image diagrams that cannot be diffed, reviewed, or drift-checked.
