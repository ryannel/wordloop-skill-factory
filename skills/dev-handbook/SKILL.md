---
name: dev-handbook
description: The definitive orientation guide for working inside the wordloop-platform monorepo. Covers the bespoke `./dev` CLI (lifecycle, testing, generation, database), Git submodule topology, the `/specs` contract directory, Terraform infrastructure (Empty Shell pattern), GitHub Actions CI/CD, system tests with mock servers, and the skill-factory sync workflow. Make sure to use this skill whenever a user asks how to start services, run tests, generate API clients, understand the database schema, deploy to GCP, add a new CLI command, debug CI failures, find where something lives in the repo, or generally orient themselves in the codebase — even if they don't explicitly say "repo" or "platform." Also invoke when the user mentions `./dev`, docker-compose, terraform, system tests, smoke tests, submodules, specs folder, or code generation pipelines.
license: MIT
metadata:
  version: "1.1.0"
  domain: repository-management
  triggers: repo setup, developer cli, ./dev, dev script, codebase architecture, git submodules, wordloop-platform, CI/CD, github actions, infrastructure test, dev-handbook, specs, code generation, terraform, system tests, smoke tests, docker compose, skill sync
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
