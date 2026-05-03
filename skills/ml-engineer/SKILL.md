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

ML service execution engineer for wordloop-ml. The doc site is the source of truth for architecture, patterns, and conventions — this skill handles routing and agent-specific guardrails only.

## Operating Contract

1. Read the relevant doc pages before making non-trivial changes. Don't rely on general ML/Python knowledge when Wordloop-specific patterns exist.
2. Preserve hexagonal layer boundaries. Domain and service layers must not import provider SDKs.
3. Never pass SDK types or unvalidated model output across layer boundaries.
4. Provider adapters are the translation layer — keep SDK details behind them.

## Routing

Read the smallest doc set that lets you act. Stop loading once you have the pattern, boundary rule, or command you need.

| Task | Read first | Also check |
|------|-----------|------------|
| Layer boundaries, DI wiring, architecture | `learn/services/ml/architecture` | `services/wordloop-ml/` for actual module layout |
| Python patterns, error handling, DI conventions | `learn/services/ml/implementation` | `pyproject.toml` for Python version and deps |
| ML principles, technology choices, anti-patterns | `principles/stack/ml-systems` | — |
| Async patterns, TaskGroup, lifespan, `asyncio.to_thread` | `principles/stack/python/async` | — |
| Timeouts, retries, circuit breakers, graceful shutdown | `principles/stack/python/resilience` | `principles/quality/reliability` |
| Transcription, diarization, embedding, RAG, streaming | `learn/services/ml/pipelines` | Provider implementations in `services/wordloop-ml/providers/` |
| MCP server, tools, resources, prompts | `principles/stack/python/mcp` | Existing MCP entrypoints in `services/wordloop-ml/` |
| Observability, tracing, structured logging | `learn/services/ml/implementation` (§ Trace-First) | `principles/quality/observability` |
| Testing strategy, fixtures, testcontainers | `principles/stack/python/testing` | `services/wordloop-ml/tests/` for existing fixtures |
| New provider / adapter | `learn/services/ml/architecture` (§ Providers) | Existing providers for SDK usage patterns |
| Python code documentation conventions | `principles/stack/python/documentation` | — |

## Safety Gates

- Do not invent model/provider behavior. Check code and configuration first.
- Do not pass SDK types into domain or service layers.
- Do not use unvalidated model output across code boundaries.
- Do not assume model availability or latency — wrap inference in timeouts, retries, and circuit breakers.
- Run targeted pytest suites after pipeline changes.

## Hallucination Controls

Before presenting ML or Python guidance as factual:

- Check `services/wordloop-ml/` for actual package structure and import patterns.
- Check `pyproject.toml` for Python version and dependency versions.
- Check existing providers before proposing new SDK patterns.
- Label general ML/Python knowledge (not Wordloop-specific) as an inference.
