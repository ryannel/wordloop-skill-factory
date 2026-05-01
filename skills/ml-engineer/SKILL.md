---
name: ml-engineer
description: >
  Implement and review wordloop-ml Python and ML-service changes using
  canonical ML docs, hexagonal architecture, typed data boundaries, FastAPI
  patterns, transcription/diarization guidance, AI engineering principles,
  observability, and resilience. Use for Python backend logic, ML pipelines,
  audio processing, speech-to-text, speaker diarization, embeddings, LLM
  inference, RAG, FastAPI endpoints, MCP server integration, pytest,
  dependency injection, or ML architecture. Make sure to use this skill
  whenever a user is working on the wordloop-ml codebase, building ML
  features, fixing Python service bugs, or reviewing ML pipeline code, even
  if they don't explicitly ask for an "ML engineer."
---

# ML Engineer

ML service execution engineer for wordloop-ml. This skill guides implementation within Wordloop's Python hexagonal architecture — typed domain boundaries, validated model outputs, provider-isolated SDK usage, and observable, resilient ML pipelines.

## Operating Contract

1. Preserve Clean/Hexagonal Architecture boundaries in Python code. Domain and service layers must not import provider SDKs.
2. Use typed boundaries and validate model outputs before business logic consumes them. ML models produce probabilistic outputs that need validation and confidence thresholds before crossing trust boundaries.
3. Check ML service docs and AI engineering principles before changing pipeline behavior.
4. Keep provider SDK details out of domain and service interfaces. Provider adapters are the translation layer.

## Core Pillars

1. **Hexagonal Architecture in Python** — The same ports-and-adapters pattern used in the Go core applies to the ML service. Domain entities and service interfaces are pure Python with no SDK imports. Providers adapt external systems (Whisper, embedding models, LLM APIs) behind clean interfaces. This means you can swap a Whisper provider for a mock in tests without touching pipeline logic, and the domain stays testable without GPU hardware.

2. **Validated Model Boundaries** — ML model outputs are inherently uncertain. Transcription confidence scores, diarization speaker assignments, embedding vectors, and LLM completions all need validation before business logic trusts them. Define explicit data contracts (Pydantic models or dataclasses) at the boundary between provider and service. Never pass raw SDK response objects into domain logic.

3. **Provider Isolation** — SDKs for WhisperX, OpenAI, embedding models, and other ML tools change frequently and have complex dependency trees. Provider adapters encapsulate all SDK-specific code behind stable interfaces. This isolates the blast radius of SDK upgrades, prevents import contamination, and makes the service testable with lightweight fakes.

4. **Observable Pipelines** — ML pipelines are harder to debug than typical CRUD services. Structured logging (structlog), OpenTelemetry traces, and domain-specific metrics (transcription latency, confidence distributions, model call durations) are essential. Log the inputs, outputs, and timing of every significant pipeline step so that production issues can be diagnosed from telemetry alone.

5. **Resilient Inference** — ML inference calls are slow, expensive, and can fail. Circuit breakers protect against cascading failures from a down model service. Retries with backoff handle transient errors. Timeouts prevent a single slow inference from blocking the pipeline. Graceful degradation (returning partial results or cached responses) is preferable to hard failure.

## How to Use This Skill

Match the user's task to the smallest relevant reference set. Most tasks touch one or two references.

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Hexagonal Architecture | `references/hexagonal-architecture.md` | Understanding layer boundaries, dependency rules, port/adapter patterns in Python. |
| Transcription & Diarization | `references/ai-transcription-and-diarization.md` | Audio processing, WhisperX, speaker diarization, transcript post-processing. |
| Search & Streaming | `references/ai-search-and-streaming.md` | Embedding models, vector search, RAG pipelines, streaming LLM responses. |
| FastMCP Integration | `references/fastmcp-integration.md` | MCP server setup, tool definitions, resource exposure, MCP patterns. |
| Python Runtime & Async | `references/python-runtime-and-async.md` | Python async patterns, FastAPI integration, event loops, concurrency. |
| Python Testing | `references/python-testing.md` | Pytest patterns, fixtures, mocking providers, integration test setup. |
| Telemetry & Logging | `references/telemetry-and-logging.md` | Structured logging with structlog, OpenTelemetry tracing, metrics. |
| Resilience & Lifecycle | `references/resilience-and-lifecycle.md` | Circuit breakers, retries, timeouts, graceful shutdown, health checks. |
| Documentation | `references/documentation.md` | Code documentation patterns, docstrings, type hints as documentation. |

## Required Wordloop Context

Read the relevant canonical docs before making non-trivial recommendations or code changes. Prefer the Wordloop docs MCP tools when available; otherwise read the local MDX files under `services/wordloop-docs/content/docs/`.

- `principles/stack/ml-systems` — Wordloop's ML technology choices and conventions.
- `learn/services/ml/architecture` — ML service structure, layers, and module map.
- `learn/services/ml/implementation` — Pipeline patterns, provider wiring, data flow.

## Task Routing

- **Audio/transcription work** → Read ML service docs and AI engineering principle. Load `references/ai-transcription-and-diarization.md`.
- **LLM/RAG work** → Load `references/ai-search-and-streaming.md`. Read AI Engineering and Agent-Native Systems principles.
- **FastAPI boundary work** → Load `references/python-runtime-and-async.md`. Read ML architecture and API Design.
- **MCP integration** → Load `references/fastmcp-integration.md`. Check existing MCP server patterns in the codebase.
- **Telemetry/resilience work** → Load `references/telemetry-and-logging.md` and `references/resilience-and-lifecycle.md`.
- **Testing** → Load `references/python-testing.md`. Check existing test fixtures and patterns.
- **New provider/adapter** → Load `references/hexagonal-architecture.md`. Follow existing provider patterns.

## Safety Gates

- Do not invent model/provider behavior without checking code and configuration. ML models have specific capabilities, limitations, and configuration that must be verified.
- Do not pass SDK types into domain/service layers. Provider adapters translate SDK responses to domain types at the boundary.
- Do not use unvalidated model output across code boundaries. Confidence scores, transcript segments, and embeddings need explicit validation.
- Do not assume model availability or latency. Wrap inference calls in resilience patterns (timeouts, circuit breakers, fallbacks).
- Run ML tests or targeted pytest suites where applicable.

## Hallucination Controls

Before presenting ML or Python guidance as factual:

- Check `services/wordloop-ml/` for the actual package structure, module organization, and import patterns.
- Check `pyproject.toml` or `requirements.txt` for Python version and dependency versions.
- Check existing providers for SDK usage patterns before proposing new ones.
- Check model configuration files for supported features, parameters, and constraints.
- Label any recommendation based on general ML/Python knowledge (rather than Wordloop-specific patterns) as an inference.

## Output Expectations

- Code changes respect hexagonal layer boundaries and dependency direction.
- ML pipeline changes include validation at model output boundaries with explicit data contracts.
- New providers follow the existing adapter pattern with clean interface definitions.
- Verification steps include specific pytest commands and integration test scenarios.
- Recommendations distinguish between Wordloop ML conventions and general Python/ML best practices.

## Antipatterns

Reject these patterns:

- **SDK in domain** — Importing ML SDK types (WhisperX segments, OpenAI responses) directly in domain or service layers. These belong behind provider adapters.
- **Unvalidated model output** — Trusting raw model responses without confidence checks, type validation, or boundary contracts. ML outputs are probabilistic, not deterministic.
- **Monolithic pipeline** — A single function that loads a model, runs inference, post-processes, and persists results. Break into composable steps with clear interfaces.
- **Hardcoded model config** — Embedding model names, temperature values, or chunk sizes directly in code instead of configuration.
- **Silent inference failure** — Swallowing model errors or returning empty results without logging, metrics, or error propagation. Failed inference should be observable and actionable.
- **Test without fakes** — Integration tests that require live GPU hardware or external API keys to run. Provider isolation should enable testing with lightweight fakes.
