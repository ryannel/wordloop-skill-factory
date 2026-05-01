# Docs/Skills Governance

Docs and skills are separate surfaces with different jobs. The docs site is the canonical home for durable engineering knowledge; skills are the execution layer that tells an agent when to load that knowledge, which tools to use, and how to verify its work.

---

## Table of Contents
1. [Boundary Rule](#boundary-rule)
2. [What Belongs in Docs](#what-belongs-in-docs)
3. [What Belongs in Skills](#what-belongs-in-skills)
4. [Skill-to-Doc Mapping](#skill-to-doc-mapping)
5. [Freshness and Active Docs](#freshness-and-active-docs)
6. [Drift Correction](#drift-correction)
7. [Hallucination Controls](#hallucination-controls)
8. [Templates](#templates)

---

## Boundary Rule

Use this rule for every docs/skills decision:

> Docs hold knowledge. Skills control execution.

A docs page should still make sense to a human who never uses an agent. A skill should still be useful to an agent because it gives concise routing, safety, and verification instructions.

---

## What Belongs in Docs

Move content to docs when it is durable, human-useful, and not primarily about agent behaviour:

- Engineering principles and architectural constraints.
- Service handbooks and implementation conventions.
- Workflow guides, runbooks, and operational procedures.
- ADRs and decision history.
- Glossary and domain vocabulary.
- API, event, schema, error, CLI, and configuration reference.
- Diagrams, data flows, and design rationale.
- Freshness policy and source-of-truth rules.

Docs should include enough structure for machine readers: descriptive headings, frontmatter, stable slugs, internal links, and Markdown-compatible content.

---

## What Belongs in Skills

Keep content in skills when it controls agent behaviour:

- Triggering and should-trigger/should-not-trigger boundaries.
- Which canonical docs pages to load for a task.
- Tool choice, command sequencing, and safety constraints.
- Verification steps and expected checks.
- Skill eval instructions and examples.
- Agent-only interaction rules.
- Bundled scripts that make repeated actions deterministic.

A skill can summarize docs context, but it should not duplicate whole policies that belong in the docs site.

---

## Skill-to-Doc Mapping

Maintain a machine-readable map from skills to canonical docs pages. The map exists to support review and health checks; it does not prove semantic correctness.

Each entry should include:

- `skill`: skill name.
- `source_path`: source-of-truth path in `tools/skill-factory`.
- `consumed_path`: synced path under `.agents/skills`.
- `canonical_docs`: docs slugs the skill depends on.
- `secondary_docs`: optional task-specific docs slugs.
- `owner`: review owner.

Review the map when:

- A skill adds or removes docs-reading instructions.
- A canonical docs page is renamed, split, or removed.
- Durable policy moves from a skill into docs.
- A docs page becomes relevant to agent execution.

---

## Freshness and Active Docs

Active docs declare ownership and review cadence. A stale page is not automatically wrong, but it is lower-trust until reviewed.

Use these defaults:

| Surface | Review window |
|---|---:|
| Principles | 6 months |
| Service handbooks | 3 months |
| Runbooks | 3 months |
| How-to guides | 6 months |
| API/event reference | Every contract change |
| Active bet TDD docs | Every material implementation change |
| Delivered bet docs | Historical |
| ADRs | Historical |
| Agent skills | Every skill or mapped docs change |

Do not update `last_reviewed` unless the claims have been checked against their source of truth.

---

## Drift Correction

Correct drift with this sequence:

1. Capture the conflicting claims with file paths or docs slugs.
2. Classify each surface: generated reference, runtime source, active guidance, historical record, or agent execution.
3. Identify the source of truth.
4. Decide whether code, docs, specs, diagrams, skills, or ADRs need to change.
5. Update all affected surfaces in one change.
6. Add a regression guard: health check, generated artifact check, test, trace assertion, or skill eval.
7. Run verification commands.

Do not fix drift by editing the first page that looks wrong.

---

## Hallucination Controls

Use these controls when writing or reviewing docs with AI assistance:

- Verify standards against official sources before naming them.
- Verify version claims against package files, module files, Dockerfiles, or runtime config.
- Verify endpoint claims against OpenAPI and event claims against AsyncAPI.
- Verify architecture claims against code, deployment config, diagrams, ADRs, or active TDD docs.
- Require exact file paths and commands in workflows.
- Label inferences as inferences and name the evidence.
- Add a health check when a repeated hallucination pattern is cheap to detect.

---

## Templates

### Skill docs dependency entry

```json
{
  "skill": "technical-writer",
  "source_path": "tools/skill-factory/skills/technical-writer/SKILL.md",
  "consumed_path": ".agents/skills/technical-writer/SKILL.md",
  "canonical_docs": [
    "principles/foundations/documentation",
    "guides/keep-docs-and-skills-in-sync",
    "guides/correct-documentation-drift",
    "operations/documentation-freshness"
  ],
  "secondary_docs": [
    "principles/ai-native/agent-native-systems",
    "decisions/0005-docs-canonical-skills-execution-layer"
  ],
  "owner": "docs-platform"
}
```

### Review note

```markdown
## Docs/skills review

- Changed docs pages:
- Affected skills:
- Skill-to-doc map updated: yes/no
- Health check run:
- Skill evals needed: yes/no, because ...
- Source-of-truth claims checked:
```
