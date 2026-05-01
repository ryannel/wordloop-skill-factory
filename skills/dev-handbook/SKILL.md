---
name: dev-handbook
description: >
  Orient agents and engineers inside the wordloop-platform monorepo using
  canonical docs for setup, CLI workflows, submodules, specs, generation,
  testing, Bet Design Studio, operations, and skill sync. Use when the user
  asks how to start services, run tests, generate clients, find repo
  structure, use ./dev, work with submodules, scaffold bets, sync skills,
  understand platform workflows, troubleshoot local dev, or navigate the
  monorepo. Make sure to use this skill whenever a user is getting started,
  asking about repository conventions, or trying to figure out how to do
  something in the platform, even if they don't explicitly mention the
  "dev handbook."
---

# Dev Handbook

Repository navigation and workflow execution guide for wordloop-platform. This skill routes engineers and agents to the right `./dev` command, canonical docs page, or source file — without duplicating the handbook itself.

## Operating Contract

1. Use canonical docs for durable repo knowledge; do not duplicate the full handbook in the skill. The docs are the source of truth for how things work.
2. Route the user to the right `./dev` command, docs page, or source file. Be a map, not an encyclopedia.
3. Verify command names against the CLI before presenting them as facts. Commands change; the skill should not contain a stale command list.
4. Preserve submodule boundaries and source-of-truth ownership. Each submodule has an upstream — changes flow through the right repo.

## Core Pillars

1. **CLI-First Workflows** — The `./dev` script is the single entry point for all platform operations: starting services, running tests, generating code, syncing skills, and managing docs. Every workflow should be expressible as a `./dev` command. If a workflow requires manual steps outside the CLI, that's a gap to flag, not a workaround to document in the skill.

2. **Canonical Docs as Source of Truth** — Durable knowledge about how the platform works lives in the docs site, not in this skill, not in READMEs scattered across directories, and not in tribal knowledge. This skill routes to docs; it does not replace them. When docs and reality disagree, the fix goes into the docs (or the code), not into a skill-local workaround.

3. **Submodule Awareness** — The monorepo contains git submodules with their own upstream repositories. Editing a submodule file is not the same as editing a monorepo file — changes need to be committed upstream first, then the submodule pointer updated. Treating submodules as ordinary folders causes merge conflicts and lost work.

4. **Verify Before Advising** — CLI commands, file paths, and configuration values change. Before recommending a command, check `./dev <command> --help` or inspect the CLI scripts. Before naming a file path, confirm it exists. Stale advice is worse than no advice because it erodes trust in the handbook.

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `start/quickstart` — Local environment setup, prerequisites, first run.
- `start/first-contribution` — Contribution workflow, branch conventions, PR process.
- `reference/cli` — Complete `./dev` command reference.
- `guides/run-tests` — Test execution across all services.
- `guides/code-generation` — API client, schema, and reference generation.
- `work` — Bet Design Studio, workflow authoring, delivery lifecycle.

## Task Routing

- **Local setup/startup** → Read Quickstart and CLI reference. Verify prerequisite versions.
- **Tests/lint/build** → Read Run Tests guide and CLI reference. Check service-specific test commands.
- **API/client generation** → Read Code Generation guide. Run `./dev gen --help` to confirm available targets.
- **Bet/workflow authoring** → Read Work hub. Check templates and active bets.
- **Skill sync** → Read Keep Docs and Skills in Sync plus CLI reference.
- **Submodule work** → Identify the submodule's upstream repo. Read submodule conventions in First Contribution guide.
- **Troubleshooting** → Read operations/troubleshooting docs. Check `./dev` logs and service health.

## Safety Gates

- Do not invent `./dev` targets. Run `./dev <command> --help` or inspect CLI scripts in `scripts/cli/` to confirm commands exist.
- Do not edit submodules as if they are ordinary folders. Note the upstream ownership and the correct commit flow.
- Do not run generation manually outside documented `./dev gen` flows. Manual generation creates drift between the contract and generated code.
- Run `./dev docs health` after documentation or skill-map changes.
- Do not recommend workarounds for broken workflows without flagging the root cause.

## Hallucination Controls

Before presenting platform guidance as factual:

- Run `./dev <command> --help` to verify commands and flags.
- Check that file paths exist before recommending them.
- Check `services/wordloop-docs/content/docs/` for the current docs structure.
- Check `tools/skill-factory/skills-docs-map.json` for skill-to-docs mappings.
- Check submodule status with `git submodule status` before advising on submodule changes.
- Label any recommendation not verified against the current repo state as an inference.

## Output Expectations

- Advice includes specific `./dev` commands, file paths, and docs page slugs — not vague pointers.
- State which docs or source files informed non-obvious claims.
- Separate source-of-truth facts from recommendations and inferences.
- When a workflow has prerequisites (running services, environment variables), name them explicitly.
- Flag when docs appear stale or when a command doesn't match its documented behavior.

## Antipatterns

Reject these patterns:

- **Stale command lists** — Maintaining a list of `./dev` commands in the skill that goes out of sync with the CLI. The CLI is the source of truth; the skill routes to it.
- **Shadow handbook** — Duplicating quickstart instructions or workflow steps in the skill body instead of pointing to canonical docs.
- **Submodule as folder** — Editing files inside a submodule directory without understanding the upstream ownership and commit flow.
- **Workaround without root cause** — Documenting a manual workaround for a broken workflow without flagging the underlying issue for repair.
- **Assumed environment** — Giving advice that assumes specific tool versions, env vars, or running services without checking or mentioning the prerequisites.
