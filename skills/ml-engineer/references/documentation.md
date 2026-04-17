# Documentation

## Table of Contents
- [Philosophy: Structure Over Comments](#philosophy-structure-over-comments)
- [Pydantic as Documentation](#pydantic-as-documentation)
- [FastAPI Endpoint Documentation](#fastapi-endpoint-documentation)
- [What to Document](#what-to-document)
- [What NOT to Document](#what-not-to-document)
- [Module-Level Documentation](#module-level-documentation)
- [Inline Comments](#inline-comments)
- [In-Code Markers](#in-code-markers)
- [Service README Template](#service-readme-template)

---

## Philosophy: Structure Over Comments

Comments are promises that no linter can verify. When code changes, `mypy` catches type errors and `pytest` catches behavioral regressions — but stale docstrings silently mislead. In a codebase where AI agents drive significant code velocity, comment drift is not hypothetical; it is the default outcome.

**The hierarchy for Python documentation:**
1. **Type annotations** — `mypy`/`pyright` reject incorrect types (zero drift risk)
2. **Pydantic `Field(description=...)`** — part of the code; flows into OpenAPI, Swagger, validation errors
3. **Naming** — descriptive function/variable names (refactor > comment)
4. **Test names** — executable documentation verified by CI
5. **Docstrings** — one-line summaries for route handlers (Swagger) and complex public APIs
6. **Inline "why" comments** — last resort for genuinely non-obvious decisions

Everything at levels 1-4 is verified by tooling. Levels 5-6 are human promises with drift risk. Minimize them ruthlessly.

---

## Pydantic as Documentation

In a FastAPI service, **Pydantic models are the primary documentation layer.** `Field(description=...)` generates OpenAPI specs, Swagger UI descriptions, and validation error messages — all from a single source of truth that cannot drift from the code because it IS the code.

### The Pattern

```python
from pydantic import BaseModel, Field
from typing import Annotated

class MeetingCreate(BaseModel):
    """Request payload for creating a new meeting."""

    title: Annotated[str, Field(
        max_length=200,
        description="Human-readable meeting title",
        examples=["Sprint Planning"],
    )]
    host_id: Annotated[str, Field(
        description="UUID of the user hosting the meeting",
        pattern=r"^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
    )]
    scheduled_at: Annotated[datetime | None, Field(
        default=None,
        description="ISO 8601 datetime for scheduled start. Omit for instant meetings.",
        examples=["2025-09-17T14:00:00Z"],
    )]
```

### Why This Is Superior to Docstrings

| Approach | Drift Risk | Tooling Integration |
|----------|-----------|-------------------|
| Docstring listing fields | **High** — nothing verifies accuracy | None — just text |
| `Field(description=...)` | **Zero** — it IS the code | OpenAPI spec, Swagger UI, validation errors, IDE hover |

**Rule:** Never duplicate `Field(description=...)` content in a class docstring. The class docstring gets one line describing the model's role — nothing more.

```python
# GOOD — minimal docstring + structural metadata
class MeetingCreate(BaseModel):
    """Request payload for creating a new meeting."""
    title: Annotated[str, Field(description="Human-readable meeting title")]

# BAD — duplicating Field metadata in the docstring
class MeetingCreate(BaseModel):
    """Request payload for creating a new meeting.

    Attributes:
        title: Human-readable meeting title, max 200 chars.
        host_id: UUID of the hosting user.
    """
```

---

## FastAPI Endpoint Documentation

FastAPI extracts route handler docstrings and renders them in Swagger UI. This is the only reason route handlers need docstrings.

### The Pattern

```python
@router.post("/meetings", response_model=MeetingResponse, status_code=201)
async def create_meeting(
    body: Annotated[MeetingCreate, Body()],
    auth: Annotated[AuthContext, Depends(require_auth)],
) -> MeetingResponse:
    """Creates a new meeting. Requires `meetings:write` scope."""
```

### Rules

- **One line is usually enough.** The `summary` in the decorator and the one-line docstring appear in Swagger UI. Elaborate only when the behavior has a non-obvious aspect.
- **Do not document parameters** in the docstring — they are documented via Pydantic `Field` and `Annotated`.
- **Markdown is supported** — use bold for emphasis when needed.

### When to add more than one line

```python
async def create_meeting(...) -> MeetingResponse:
    """Creates a new meeting. Requires `meetings:write` scope.

    If `scheduled_at` is provided, the meeting starts in `scheduled`
    status. Otherwise it starts immediately in `active` status.
    """
```

The status branching logic is invisible in the type signature — it justifies a second sentence.

---

## What to Document

| What | How | Why |
|------|-----|-----|
| **Pydantic model fields** | `Field(description=...)` | Flows into OpenAPI, Swagger, validation errors |
| **Pydantic model class** | One-line docstring | Describes the model's domain role |
| **Route handler** | One-line docstring | Appears in Swagger UI |
| **Complex public function** | Google-style docstring if types alone don't convey the contract | Consumer needs to know edge cases, error conditions |
| **Side effects** | Note in docstring if invisible in the signature | Event publishing, cache invalidation, external calls |
| **Module purpose** | Module docstring when the filename isn't self-explanatory | Orients developers navigating the codebase |

---

## What NOT to Document

**Obvious `__init__` with typed parameters:**
```python
# BAD — types say everything
class MeetingService:
    def __init__(self, repo: MeetingRepository, notifier: Notifier) -> None:
        """Initializes MeetingService with repo and notifier."""
        self._repo = repo
        self._notifier = notifier

# GOOD — skip the __init__ docstring
class MeetingService:
    """Orchestrates meeting lifecycle operations."""

    def __init__(self, repo: MeetingRepository, notifier: Notifier) -> None:
        self._repo = repo
        self._notifier = notifier
```

**Simple functions where types tell the story:**
```python
# BAD — restating the signature
def get_meeting(meeting_id: str) -> Meeting | None:
    """Gets a meeting by its ID, returning Meeting or None."""

# GOOD — the signature IS the documentation
def get_meeting(meeting_id: str) -> Meeting | None:
    ...
```

**Properties with obvious types:**
```python
# BAD — noise
@property
def title(self) -> str:
    """Returns the meeting title."""
    return self._title

# GOOD — skip it
@property
def title(self) -> str:
    return self._title
```

**Private helper functions with clear names:**
```python
# Skip — the name + types tell the story
def _build_speaker_embedding(audio_chunk: AudioChunk) -> np.ndarray:
    ...
```

**Args/Returns sections that repeat type hints:**
```python
# BAD — duplicating type information
def process(audio_url: str, diarize: bool = True) -> TranscriptResult:
    """Processes audio.

    Args:
        audio_url (str): URL of the audio file.
        diarize (bool): Whether to diarize. Defaults to True.

    Returns:
        TranscriptResult: The transcript result.
    """

# GOOD — if the types + naming are clear, no docstring needed
# If you DO write one, add only what types cannot convey:
def process(audio_url: str, diarize: bool = True) -> TranscriptResult:
    """Processes audio and returns a structured transcript.

    Raises:
        AudioDownloadError: If the URL is unreachable.
        DiarizationError: If diarization is requested but HF_TOKEN is missing.
    """
```

---

## Module-Level Documentation

Module docstrings orient developers navigating the codebase. Keep them to 1-3 lines.

**Good — brief orientation:**
```python
"""Meeting domain entities and business rules.

Domain entities have no knowledge of persistence, transport, or external services.
"""
```

**Skip when the filename is self-explanatory:**
```python
# internal/provider/postgres_repository.py
# The filename says "postgres_repository" — no docstring needed
```

**`__init__.py` — only if it re-exports:**
```python
"""Core domain layer exports."""
from .meeting import Meeting
from .participant import Participant
```

---

## Inline Comments

Inline comments within function bodies are justified **only** for genuinely non-obvious "why" that cannot be expressed through naming or structure:

```python
async def transcribe(self, audio: AudioSegment) -> TranscriptResult:
    audio = audio.resample(16000) if audio.sample_rate != 16000 else audio

    # Batch size 16 determined via GPU memory profiling on T4 instances.
    segments = self._model.transcribe(audio.numpy(), batch_size=16)

    # HACK(alice): WhisperX returns negative start times on clips <2s.
    # Clamp to zero until upstream fix. See whisperX#XXX.
    for seg in segments:
        seg.start = max(0.0, seg.start)

    return TranscriptResult(segments=segments)
```

The resample call needs no comment — the conditional expression is clear. The batch size and the clamping hack are genuinely non-obvious and justify comments.

---

## In-Code Markers

```python
# TODO(bob): Switch to streaming transcription when WhisperX v4 adds support.
# Current batch approach buffers full audio in memory. Issue #123.

# FIXME(carol): Race condition on concurrent requests for same audio_id.
# Needs distributed lock via Redis. Issue #456.

# HACK(dave): Pyannote v3.1 returns empty segments for audio < 500ms.
# Remove after v3.2 upgrade.
```

Always include `(username)` and an issue reference.

---

## Service README Template

```markdown
# [Service Name]

One-sentence description of what this ML service does.

[![CI](badge-url)](ci-url)
[![Coverage](badge-url)](coverage-url)

## Architecture

\```
internal/
├── core/
│   ├── domain/       # Business entities, value objects, domain errors
│   ├── gateway/      # Outbound port interfaces (typing.Protocol)
│   └── service/      # Application services (orchestration)
├── entrypoints/      # Driving adapters (FastAPI routes, MCP tools)
└── provider/         # Driven adapters (DB, ML models, external APIs)
\```

## Quick Start

\```bash
git clone https://github.com/org/[service].git && cd [service]
cp .env.example .env
uv sync
uv run python -m [service]
\```

API: `http://localhost:8000` | Docs: `http://localhost:8000/docs`

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8000` | FastAPI server port |
| `DATABASE_URL` | — | PostgreSQL connection string |
| `HF_TOKEN` | — | HuggingFace token for model downloads |
| `LOG_LEVEL` | `info` | Logging level |
| `OTEL_EXPORTER_ENDPOINT` | — | OpenTelemetry collector |

## Testing

\```bash
uv run pytest                    # All tests
uv run pytest tests/unit/        # Domain layer (no Docker)
uv run pytest tests/integration/ # Provider layer (requires Docker)
\```

## License

[MIT](LICENSE)
```
