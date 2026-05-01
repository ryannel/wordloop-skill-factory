---
name: dev-handbook
description: The definitive orientation guide for working inside the wordloop-platform monorepo. Covers the bespoke `./dev` CLI (lifecycle, testing, generation, database, authoring), Git submodule topology, the `/specs` contract directory, the Bet Design Studio workflow (problem statements, pitches, active bets, TDD architecture), Terraform infrastructure (Empty Shell pattern), GitHub Actions CI/CD, system tests with mock servers, and the skill-factory sync workflow. Make sure to use this skill whenever a user asks how to start services, run tests, generate API clients, understand the database schema, deploy to GCP, add a new CLI command, debug CI failures, scaffold a new feature, find where something lives in the repo, or generally orient themselves in the codebase — even if they don't explicitly say "repo" or "platform." Also invoke when the user mentions `./dev`, docker-compose, terraform, system tests, smoke tests, submodules, specs folder, code generation pipelines, problem statements, pitches, bets, bet design, data flows, UI design, TDD, ways of working, `./dev new`, or `./dev archive`.
license: MIT
metadata:
  version: "1.3.0"
  domain: repository-management
  triggers: repo setup, developer cli, ./dev, dev script, codebase architecture, git submodules, wordloop-platform, CI/CD, github actions, infrastructure test, dev-handbook, specs, code generation, terraform, system tests, smoke tests, docker compose, skill sync, problem statement, pitch, bet, bet design, data flow, ui design, tdd, ways of working, ./dev new, ./dev archive
  role: navigator
  scope: orientation
---

# Repo Expert — Wordloop Platform

You are the definitive navigator for the `wordloop-platform` monorepo. This repository is the control plane: it hosts infrastructure definitions, CI/CD pipelines, cross-service system tests, and orchestrates four application submodules plus a skill-factory toolchain.

## Repository Map

```
wordloop-platform/
├── dev                           # CLI entrypoint (bash)
├── scripts/cli/                  # Modular CLI modules (lifecycle, quality, infra, tasks, doctor)
├── docker-compose.yml            # Infrastructure graph + profile-gated app services
├── services/
│   ├── wordloop-core/            # [submodule] Go REST API          → :4002
│   ├── wordloop-ml/              # [submodule] Python ML engine     → :4003
│   ├── wordloop-app/             # [submodule] Next.js frontend     → :4001
│   └── wordloop-docs/            # [submodule] Fumadocs site        → :4000
├── tools/skill-factory/          # [submodule] AI agent skills (source of truth)
├── .agents/skills/               # Synced copy consumed by agents
├── specs/                        # Aggregated OpenAPI + AsyncAPI schemas
├── .dev/                         # Runtime state: PIDs, logs, schema.sql
├── infrastructure/
│   ├── bootstrap.sh              # Idempotent GCP project setup
│   └── terraform/                # All IaC (Cloud Run, SQL, Pub/Sub, IAM, etc.)
├── tests/
│   ├── system/                   # Cross-service integration tests (Python/Pytest)
│   └── infrastructure/           # Smoke tests + CLI self-tests (Bash)
└── .github/workflows/deploy.yml  # Unified CI/CD pipeline
```

## Git Submodules

Every service and the skill-factory are independent Git repositories, pinned as submodules. This means:

- **Each submodule has its own CI, issues, and release cadence.** The platform repo's CI/CD only triggers builds when a submodule's content changes inside `services/`.
- **Clone with `--recurse-submodules`** or run `git submodule update --init --recursive` after cloning.
- **Commits to submodules** happen in their own repos. To update the platform's pin, commit the new submodule reference in the platform repo.

| Submodule | Language | Port | Purpose |
|-----------|----------|------|---------|
| `services/wordloop-core` | Go | 4002 | REST API, domain logic, PostgreSQL, Pub/Sub publisher |
| `services/wordloop-ml` | Python | 4003 | Transcription (AssemblyAI), speaker embeddings, LLM inference |
| `services/wordloop-app` | Next.js | 4001 | Web frontend (Clerk auth, SWR, Tailwind v4) |
| `services/wordloop-docs` | Next.js | 4000 | Fumadocs documentation site, OpenAPI/AsyncAPI explorer |
| `tools/skill-factory` | Mixed | — | Source of truth for all AI agent skills |

## The `./dev` CLI

All local development is driven through a single entrypoint: `./dev`. It delegates to modular Bash scripts in `scripts/cli/` and manages both Docker infrastructure and native background processes.

The reason `./dev` exists (rather than raw `docker compose` or `make`) is that it coordinates a hybrid execution model: infrastructure runs in Docker, but application services run natively as background processes with PID tracking and log capture under `.dev/pids/` and `.dev/logs/`. This gives you hot-reload (Go via Air, Python via uvicorn, Next.js via `pnpm dev`) while keeping infrastructure stable in containers.

### Quick Reference

| Category | Command | What it does |
|----------|---------|--------------|
| **Lifecycle** | `./dev start all` | Boots Docker infra (Postgres, Pub/Sub, Storage, OTel) then starts Core, ML, App, Docs as native processes |
| | `./dev start infra core` | Only infra + Core (useful when debugging ML in an IDE) |
| | `./dev stop all` | Gracefully kills native processes and tears down Docker |
| | `./dev status` | Dashboard showing port status, PIDs, and URLs for every component |
| | `./dev logs core ml` | Tails `.dev/logs/` for native services (supports mixed Docker + native tailing) |
| **Quality** | `./dev build all` | Simulates CI Docker builds locally for all services |
| | `./dev test core` | Go unit tests (`go test ./...`) |
| | `./dev test ml` | Python unit tests (`pytest tests/unit`) |
| | `./dev test app` | Frontend tests (`pnpm vitest run`) |
| | `./dev test system` | Full cross-service integration tests (see System Tests below) |
| | `./dev test smoke` | Infrastructure health checks (`tests/infrastructure/smoke_test.sh`) |
| | `./dev lint all` | Runs `go vet`, `ruff check`, `eslint` across all services |
| **Database** | `./dev db dump` | Boots Postgres if needed, applies latest Go migrations, then exports a clean `pg_dump` to `.dev/schema.sql` — read this file to quickly understand the full Core database structure |
| | `./dev db migrate` | Apply pending migrations |
| | `./dev db shell` | Interactive `psql` console |
| | `./dev db prod-proxy start` | Opens a Cloud SQL Auth Proxy tunnel to the live production database on `:5433` |
| **Generation** | `./dev gen all` | Runs the full generation pipeline (see Code Generation below) |
| **Authoring** | `./dev new problem-statement <slug>` | Creates a problem statement at `work/problem-statements/<slug>.mdx` and registers in sidebar |
| | `./dev new pitch <slug>` | Promotes a problem statement into a pitch at `work/pitches/<slug>.mdx` (carries problem content forward) |
| | `./dev new bet <slug>` | Promotes a pitch into an active bet at `work/<slug>/` with TDD scaffolding and test suite |
| | `./dev new milestone <bet-slug> <milestone>` | Scaffolds integration milestone documentation and automated test stubs |
| | `./dev new slice <bet-slug> <milestone> <domain> <slice>` | Scaffolds a specific domain slice and testing boundaries |
| | `./dev new contract <bet-slug> <service> <protocol>` | Scaffolds a new API contract layout |
| | `./dev new schema <bet-slug> <service> <tech>` | Scaffolds a new database schema planning doc |
| | `./dev archive bet <slug>` | Archives a delivered bet (docs + tests, hides from sidebar, preserves history) |
| | `./dev test bet <slug>` | Runs the bet progress test suite on demand |
| **Utilities** | `./dev sync skills` | Diffs `tools/skill-factory/skills` against `.agents/skills` and syncs changes |
| | `./dev dash obs` | Opens the .NET Aspire Dashboard (OTel traces, metrics, logs) |
| | `./dev dash docs` | Opens the Fumadocs site |
| | `./dev doctor` | Validates system dependencies, Docker, ports, and tool versions |
| | `./dev setup env` | Copies `.env.example` templates to active `.env` files for all services |
| | `./dev setup install` | Installs pnpm and uv dependencies across all TS and Python services |

## The `/specs` Directory

The `/specs` directory at the repository root is the centralized, aggregated collection of all API contracts across the platform. It contains:

- **`core-openapi.json`** — The full OpenAPI 3.x specification for the Go Core API. Use this to understand every REST endpoint, request/response shape, and authentication requirement without reading Go routing code.
- **`ml-openapi.json`** — The OpenAPI spec for the Python ML service. Covers transcription, speaker embedding, and LLM endpoints.
- **`core-asyncapi-ws.yaml`** — AsyncAPI v3 spec defining all WebSocket event types between the frontend and Core.
- **`core-asyncapi-pubsub.yaml`** — AsyncAPI v3 spec defining all Pub/Sub event types between Core and ML.

This folder is the best starting point when you need to understand what data flows between services, what endpoints exist, or what event payloads look like. Read these files freely for context.

> [!WARNING]
> `/specs` is for reading and understanding contracts. Do not manually run code generation tools (like Orval, openapi-generator, or custom scripts) against these files. Client generation is tightly orchestrated across all three services via `./dev gen clients`, which ensures consistency. Running generation manually will produce clients that drift from the coordinated pipeline.

## Code Generation (`./dev gen`)

The platform uses a contract-first workflow: schemas are the source of truth, and typed clients are generated from them. The generation pipeline has strict ordering because later stages depend on earlier ones.

| Target | Command | What happens |
|--------|---------|--------------|
| `api` | `./dev gen api` | Compiles OpenAPI from Core (Go) and ML (Python) source code into `/specs`. Copies AsyncAPI YAML files. |
| `events` | `./dev gen events` | Parses AsyncAPI specs and generates typed event structs/classes for Go, TypeScript, and Python. Depends on `api`. |
| `clients` | `./dev gen clients` | Hydrates API clients — App (TS via Orval), Core (Go), ML (Python) — from `/specs`. Depends on `api`. |
| `docs` | `./dev gen docs` | Regenerates Fumadocs OpenAPI reference pages from `/specs`. Depends on `api`. |
| `all` | `./dev gen all` | Runs `api` → `events` → `clients` → `docs` in sequence. |

## Bet Design Studio

The Bet Design Studio is a structured section in the docsite (`/docs/work`) where work is designed, architected, and tracked through delivery. All content lives in `services/wordloop-docs/content/docs/work/`. The full guidance lives in `work/index.mdx` — read that file for the authoritative "How We Work" reference.

### The Lifecycle

Work moves through discrete stages. At each promotion, the CLI moves content forward and updates the sidebar.

| Phase | What Happens | Where it lives | CLI command |
|-------|-------------|----------------|-------------|
| **1. Problem Statement** | Observed pain, evidence, appetite judgment | `work/problem-statements/<slug>.mdx` | `./dev new problem-statement <slug>` |
| **2. Pitch** | Problem + rough solution, rabbit holes, no-gos | `work/pitches/<slug>.mdx` | `./dev new pitch <slug>` |
| **3. Active Bet** | Pitch promoted, TDD designed, execution begins | `work/<slug>/` | `./dev new bet <slug>` |
| **4. Delivered** | Archived with full history | `work/delivered/<slug>/` | `./dev archive bet <slug>` |

### Problem Statements

Problem statements live in `work/problem-statements/`. They capture observed pain — real, evidenced, specific — alongside an appetite judgment. The appetite is an opportunity cost judgment made *before* the solution is defined: how much time is this problem worth?

**Platform and infrastructure problems are valid problem statements.** The "who experiences it" can be the engineering team, the system's reliability, or the business's compliance posture — not just end-users. Feature bets routinely surface infrastructure gaps (e.g., "we have no event backplane between Core pods"). The right response is to extract the gap as its own problem statement rather than expanding the feature bet. The feature bet declares the constraint explicitly; the platform bet solves it.

A problem statement contains three sections: **Problem** (what, who, evidence), **Appetite** (time budget + reasoning), **Why Now** (why this cycle, not later). Templates are in `work/_template/problem-statement.mdx`.

### Pitches

Pitches live in `work/pitches/`. A pitch links a validated problem to a rough solution proposal — concrete enough to execute against, but staying away from micro-detail. Creating a pitch via CLI carries the problem statement content forward automatically.

A pitch must contain:
- **Problem** and **Why Now** — carried from the problem statement
- **Proposed Solution** — fat-marker sketch, diagrams encouraged, implementation detail is not
- **Rabbit Holes** — approaches already ruled out. Include plausible-looking approaches that would blow the appetite *and* infrastructure assumptions the bet makes (e.g., "we assume sticky sessions, not a backplane")
- **No-Gos** — capabilities explicitly out of scope. Include both obvious exclusions *and* natural extensions users would reasonably expect but that don't belong in this version (these are the most important no-gos because they're the ones most likely to creep in)

Templates are in `work/_template/pitch.mdx`.

### Active Bets — Content Structure

Active bets live in `work/<slug>/`. Each bet is a directory:

```
work/<slug>/
├── pitch.mdx                  ← Promoted pitch (carried from pitches/)
├── tdd/                       ← Technical Design Doc (directory)
│   ├── index.mdx              ← Overview, success criteria, constraints
│   ├── ui-design.mdx          ← Screen walkthrough with states and interactions
│   ├── data-flow.mdx          ← Service-level data flow diagram
│   ├── milestones/            ← Integration milestones (scaffolded)
│   ├── domain-slices/         ← Testable domain slices (scaffolded)
│   ├── contracts/             ← API contracts per boundary (scaffolded)
│   └── schemas/               ← Database schema plans (scaffolded)
└── meta.json
```

### TDD Documents — What Each One Contains

Templates for all TDD documents are in `work/_template/tdd/`. Read the template before writing a new instance — it contains the structural guidance and scope checks.

**UI Design** (`ui-design.mdx`) — What the user sees and does. Organised by screen, each with: wireframe, layout, states table, key interactions. Then user journeys (ASCII flow diagrams) and edge cases. Scope check: if you're specifying which service owns the logic, you've gone too far — that belongs in Data Flow. For features with live connections, every failure mode gets its own named row in the States table (Degraded, Connectivity Lost, Reconnected — not just "Error"). Edge cases should prompt for: concurrent access (multi-tab, multi-user), session boundaries (tab close, token expiry), resource limits, and background/foreground behaviour.

**Data Flow** (`data-flow.mdx`) — Maps every user action from the UI Design through service boundaries. Starts with a **system context graph** (topology, not sequence), then groups flows into logical **Parts** with numbered sequence diagrams. Diagrams use descriptive operation labels — never endpoint paths, header names, or field names (those belong in Contracts). Failure mode flows are **required** for every significant service boundary. Closes with two required sections: **Design Decisions** (tradeoffs, constraints, alternatives ruled out — including infrastructure constraints, scope deferrals, performance choices, lifecycle policies) and **Boundary Inventory** (5 columns: Boundary | Flows | From → To | Protocol | Data shape — each row becomes a contract).

**Contracts** (`contracts/`) — Agreed API contracts (REST, WebSocket, Pub/Sub) derived from the Boundary Inventory. Scaffolded via `./dev new contract`.

**Schemas** (`schemas/`) — Database schemas (PostgreSQL, object storage) derived from the data objects defined in UI Design and the persistence decisions in Data Flow. Scaffolded via `./dev new schema`.

### Wireframe Images

Wireframes for bets are stored in `public/images/bets/<slug>/`. Reference them as `![Alt text](/images/bets/<slug>/screen_name.png)`.

### Test-Driven Delivery

Every domain slice and milestone in the TDD must define test cases. Tests serve dual purpose:
1. **Service/system tests** (permanent) — in the service repo or `tests/system/`
2. **Bet progress suite** (temporary) — in `tests/bets/<slug>/`, run on demand

The bet progress suite starts all-red, goes green as slices ship, and is archived with the bet.

### Scaffolding Architecture (TDD)

Progressively generate architecture components using the Golden Path CLI. This ensures test stubs and documentation stay synchronised.

```bash
# Scaffolds milestones and domains
./dev new milestone <bet-slug> <milestone-slug>
./dev new slice <bet-slug> <milestone-slug> <domain> <slice-slug>

# Scaffolds architectural boundaries
./dev new contract <bet-slug> <service> <protocol>
./dev new schema <bet-slug> <service> <database-tech>
```

### Running Bet Progress Tests

```bash
./dev test bet <slug>
```

### Archiving a Delivered Bet

```bash
./dev archive bet <slug>
```

Moves the bet directory to `_archive/` and the test suite to `tests/bets/_archive/<slug>/`. The full design history is preserved and remains accessible by URL.

## System & Smoke Tests

### System Tests (`./dev test system`)

Located in `tests/system/`. These are true end-to-end integration tests that boot the entire service topology via a dedicated `docker-compose.test.yml` using isolated ports (Core on `:5002`, ML on `:5003`, Postgres on `:5433`).

The test harness:
- Builds mock servers (`tests/system/mock_assemblyai/`, `tests/system/mock_openai/`) as Docker images to simulate third-party APIs.
- Uses pytest fixtures (`conftest.py`) that orchestrate Docker Compose, wait for health endpoints, seed test data, and reset state between tests.
- Tests cover batch transcription flows, real-time WebSocket streaming, and offline processing.

### Smoke Tests (`./dev test smoke`)

Located in `tests/infrastructure/smoke_test.sh`. Pure Bash scripts that verify infrastructure health — checking that Docker services respond, ports are bound, and basic HTTP health endpoints return 200.

### CLI Tests (`./dev test cli`)

Located in `tests/infrastructure/test_cli.sh`. Self-tests for the `./dev` CLI itself — verifying that command routing, help text, and argument parsing work correctly.

## Infrastructure & Terraform

All GCP infrastructure lives in `infrastructure/terraform/`. A companion `infrastructure/bootstrap.sh` script handles one-time project setup (state bucket creation, API enablement, org policy overrides).

### The Empty Shell Pattern

Terraform provisions Cloud Run services using a placeholder container image (`us-docker.pkg.dev/cloudrun/container/hello`). It configures everything *around* the container — IAM, secrets, networking, scaling, database connections — but never touches application code. This separation exists because:

- Terraform applies never fail due to application bugs or startup probe failures.
- There's no dependency on local Docker builds or architecture mismatches (ARM vs AMD64).
- Application deployments happen exclusively through GitHub Actions, keeping infrastructure and code delivery independent.

Every Cloud Run service block includes `lifecycle { ignore_changes = [template[0].containers[0].image] }` so Terraform won't revert images that CI/CD has updated.

### Key Terraform Files

| File | Purpose |
|------|---------|
| `cloudrun-core.tf` | Core API service + migration job |
| `cloudrun-ml.tf` | ML service |
| `cloudrun-app.tf` | Next.js frontend |
| `cicd.tf` | Workload Identity Federation for keyless GitHub Actions auth |
| `database.tf` | Cloud SQL PostgreSQL |
| `pubsub.tf` | Pub/Sub topics and subscriptions |
| `secrets.tf` | Secret Manager references |
| `iam.tf` | Service accounts and least-privilege bindings |

## CI/CD Pipeline

Deployments are governed by a single workflow: `.github/workflows/deploy.yml`. It triggers on pushes and PRs to `main`.

### How it works

1. **Change detection**: `dorny/paths-filter` identifies which submodules or terraform files changed, so only affected services are built and deployed.
2. **Terraform**: If infrastructure changed, the pipeline authenticates to GCP via Workload Identity Federation (keyless — no service account keys stored in GitHub) and runs `terraform plan` on PRs or `terraform apply` on merges to `main`.
3. **Service deployment**: For each changed service, the pipeline builds a Docker image, pushes it to Artifact Registry (`europe-west1-docker.pkg.dev/wordloop-ai/wordloop`), and runs `gcloud run deploy`.
4. **Database migrations**: The Core deployment job runs `gcloud run jobs execute wordloop-core-migrate` before deploying the new service image, ensuring the schema is always ahead of the code.

The key mental model: Terraform owns the *infrastructure shape* (what services exist, what secrets they can access, what databases they connect to). GitHub Actions owns the *application code* (what version of the code is running). They never overlap.
