---
name: dev-handbook
description: >
  Orient agents and engineers inside the wordloop-platform monorepo by routing
  tasks to the minimum canonical docs, ./dev CLI output, and source files needed
  for setup, CLI workflows, submodules, code generation, testing, database
  design, Bet Design Studio work, architecture, and repository navigation. Use
  this skill whenever a user is getting started, asks how to do something in the
  platform, asks where knowledge lives, touches repository-wide workflows, or
  needs to understand how Wordloop fits together — even if they don't explicitly
  ask for the dev handbook. Load it before asking platform questions.
---

# Dev Handbook

**Wordloop** is an AI-powered meeting intelligence platform: transcription, speaker identification, and conversation insights. Three services — `wordloop-app` (Next.js :4001), `wordloop-core` (Go :4002), `wordloop-ml` (Python :4003) — coordinated through this monorepo.

Use this skill as a context router. Load only what the task requires; stop as soon as you can act.

## Submodule Warning

`services/` and `tools/` directories are Git submodules with their own upstream histories. Check state before editing:

```bash
git submodule status                        # See all pinned commits
git -C services/<name> status --short       # Check a specific submodule
```

Commits inside a submodule must be made upstream first; the monorepo then updates the pointer. Never treat submodule directories as ordinary folders when planning edits or commits.

| Submodule path | Owns |
|---|---|
| `services/wordloop-app` | Next.js frontend |
| `services/wordloop-core` | Go API, domain logic, DB migrations, OpenAPI/AsyncAPI |
| `services/wordloop-docs` | Documentation site and MCP corpus |
| `services/wordloop-ml` | Python/FastAPI ML service and pipelines |
| `tools/skill-factory` | Source skills, evals, and skill-to-doc map |

## Operating Contract

1. Start from the user's task. Load only the smallest docs/source set needed to act.
2. Prefer Wordloop docs MCP tools when available; fall back to local MDX under `services/wordloop-docs/content/docs/`.
3. Verify command names, flags, paths, and submodule state in the current repo before presenting them as facts.
4. Fix durable knowledge in docs or code, not in this skill. If this skill needs a platform fact to answer, that fact probably belongs in the docs too.
5. To edit a maintained skill: change `tools/skill-factory/skills/<skill>/SKILL.md` first, then run `./dev sync skills` to propagate to `.agents/skills/`.

## Context Budget

Stop loading once you can name the command, doc slug, source file, and verification step for the task. For most tasks that's one or two pages.

- **Orientation only:** `index` — service map, ports, and purpose in one short page.
- **Implementation work:** also load the relevant service skill (`core-engineer`, `app-engineer`, `ml-engineer`).
- **CLI commands:** run `./dev help` or `./dev <verb> --help` rather than relying on docs tables; they are the executable source of truth.
- **Bet design and review:** for document design, review, or delivery work (problem statements, pitches, TDD, milestones, slices), hand off to the `bet-lead` skill which routes to the specific doc section and template needed.

## Route By Task

| Task | Load first | Add only if needed | Verify with |
|---|---|---|---|
| Wordloop's purpose, service map, ports | `index` | `learn/architecture/overview` | repository layout |
| System architecture, service boundaries | `learn/architecture/overview` | `learn/architecture/system-workflows`, `learn/architecture/data-flow` | current docs and repo |
| First setup, local startup, prerequisites | `start/quickstart` | `learn/architecture/local-infrastructure`, `reference/configuration` | `./dev doctor`, `./dev start --help` |
| `./dev` command usage | `reference/cli` | `principles/delivery/devex` | `./dev help`, `./dev <verb> --help` |
| Maintaining or extending the `./dev` CLI | `reference/cli` | `principles/delivery/platform` | `scripts/cli/*.sh`, `./dev <verb> --help` after changes |
| Tests, lint, build, CI parity | `guides/run-tests` | `principles/foundations/testing`, service-specific docs | `./dev test --help`, `./dev lint --help`, `./dev build --help` |
| Codegen, clients, OpenAPI, AsyncAPI | `guides/code-generation` | `reference/api/core`, `reference/events/core-ws`, `reference/events/core-pubsub` | `./dev gen --help`, `specs/`, owning service scripts |
| Database schema, migrations, query design | `reference/database` | `guides/migrate-schema`, `principles/stack/postgres` | `services/wordloop-core/db/schema.sql`, `services/wordloop-core/cmd/migrate`, `./dev db dry-run`, `./dev db migrate`, service tests |
| Scaffolding bets, running bet CLI commands | `work` | `reference/cli` | `./dev new --help`, `./dev test bet <slug>` |
| Designing or reviewing bet documents | Hand off to `bet-lead` skill | — | — |
| Submodules, contribution flow, repo ownership | `start/first-contribution` | `.gitmodules`, `git submodule status` | submodule git status and upstream remotes |
| Docs/skill drift or skill maintenance | `guides/keep-docs-and-skills-in-sync` | `decisions/0005-docs-canonical-skills-execution-layer` | `tools/skill-factory/skills-docs-map.json`, `./dev docs health` |
| Troubleshooting local development | `operations/troubleshooting` | task-specific guide above | `./dev status`, `./dev logs --help` |

## Source-Of-Truth Checks

Before answering with platform facts:

- Check the doc slug exists via docs MCP or local MDX before citing it.
- Check CLI claims against current help output or `scripts/cli/`.
- Check workspace paths before linking or recommending edits.
- Check submodule boundaries before editing files under `services/` or `tools/`.
- If docs and code disagree, name which source disagrees and fix the canonical source when scope allows.

## Output Expectations

- Name the docs or source files that informed non-obvious guidance.
- Give exact commands only after verification; otherwise label them as inferred.
- Include prerequisites when a command depends on running services, generated specs, env files, or Docker.
- Keep responses task-sized. Do not summarize platform architecture unless the task requires it.
