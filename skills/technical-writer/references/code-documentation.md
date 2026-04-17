# Code Documentation

## Table of Contents
- [The Documentation Hierarchy](#the-documentation-hierarchy)
- [Structure Over Comments](#structure-over-comments)
- [When Comments Are Justified](#when-comments-are-justified)
- [When Comments Are Harmful](#when-comments-are-harmful)
- [In-Code Markers](#in-code-markers)
- [Language-Specific Conventions](#language-specific-conventions)

---

## The Documentation Hierarchy

Comments are promises that no compiler can verify. When an AI agent or human changes the code beneath a comment, nothing warns that the comment is now a lie. This makes comments a **liability with maintenance cost** — every comment is technical debt that must be actively maintained.

The principle is simple: **structure first, comments as a last resort.**

### The Trust Hierarchy

```
HIGHEST TRUST — compiler or runtime verified, cannot drift
│
├── 1. Type signatures
│      Go types, TypeScript interfaces, Python type hints
│      → The compiler rejects incorrect types
│
├── 2. Structural metadata
│      Pydantic Field(description=...), Zod .describe(),
│      OpenAPI schemas, Go struct tags
│      → Part of the code itself; used by runtime/tooling
│
├── 3. Naming
│      Descriptive function names, variable names, constants
│      → Self-documenting; refactoring > commenting
│
├── 4. Test names
│      describe("when user is unauthenticated", ...)
│      → Executable documentation; CI verifies them
│
│   ── TRUST BOUNDARY ──
│      Below here, nothing verifies accuracy.
│      Every item is a potential future lie.
│
├── 5. Contract comments on public APIs
│      GoDoc, TSDoc, docstrings
│      → Useful for tooling (pkgsite, Swagger, IDE hover)
│      → But drift risk increases with comment volume
│
├── 6. Inline "why" comments
│      → Justified only for genuinely non-obvious decisions
│
├── 7. TODO/FIXME markers
│      → Actionable and trackable
│
└── 8. Prose documentation (README, docs/)
       → Highest drift risk; furthest from the code
│
LOWEST TRUST — no verification mechanism
```

**The rule:** Exhaust every option above the trust boundary before reaching for anything below it. If you find yourself writing a comment that explains *what* code does, stop and refactor the code to be self-explanatory instead.

---

## Structure Over Comments

The strongest documentation is code that doesn't need explanation. Before writing a comment, ask: **"Can I make this unnecessary?"**

### Types as Documentation

In typed languages, the type system is documentation the compiler verifies:

```typescript
// UNNECESSARY — the signature says everything
/** Takes a user ID string and returns a User or null */
function getUser(id: string): Promise<User | null>

// The signature alone IS the documentation
function getUser(id: string): Promise<User | null>
```

```python
# UNNECESSARY — type hints say it all
def get_user(user_id: str) -> User | None:
    """Gets a user by their ID and returns User or None."""
    ...

# The signature IS the documentation
def get_user(user_id: str) -> User | None:
    ...
```

### Structural Metadata as Documentation

Pydantic `Field()` and Zod `.describe()` are superior to comments because they are **part of the code** — they flow into OpenAPI specs, validation errors, and IDE tooltips. They cannot drift because they ARE the code:

```python
# STRUCTURAL — this IS documentation AND code
class MeetingCreate(BaseModel):
    title: Annotated[str, Field(max_length=200, description="Meeting title")]
```

```typescript
// STRUCTURAL — flows into validation errors and IDE hover
const meetingSchema = z.object({
  title: z.string().min(1).max(200).describe("Meeting title"),
});
```

### Naming as Documentation

```python
# BAD — needs a comment to explain
d = 86400  # seconds in a day

# GOOD — the name IS the documentation
SECONDS_PER_DAY = 86400
```

```python
# BAD — unclear function, so you add a comment
def proc(d, f):
    """Processes audio segment and converts to output format."""
    ...

# GOOD — the signature tells the story, comment is unnecessary
def process_audio_segment(segment: AudioSegment, format: OutputFormat) -> TranscriptChunk:
    ...
```

### Structure as Documentation

Small, focused functions with clear responsibilities eliminate comment need:

```python
# BAD — a long function that needs paragraph comments
def process_meeting():
    # Step 1: Download audio from storage...
    # ... 50 lines ...
    # Step 2: Transcribe audio segments...
    # ... 60 lines ...

# GOOD — the structure tells the story
def process_meeting():
    audio = download_audio(meeting_id)
    transcript = transcribe_segments(audio)
    speakers = identify_speakers(transcript)
    return Meeting(transcript=transcript, speakers=speakers)
```

---

## When Comments Are Justified

Comments are justified only when **no structural alternative exists** — when the information genuinely cannot be expressed through types, naming, metadata, or tests.

### The Justified Set

| Situation | Why a Comment Is Needed | Example |
|-----------|------------------------|---------|
| **Concurrency semantics** | Types cannot express thread safety | `// Safe for concurrent use by multiple goroutines.` |
| **Non-obvious workarounds** | Future maintainers need to know this is intentional, not a bug | `// Workaround for pgx v5 bug #1234 — remove after v5.7 upgrade` |
| **Regulatory/business constraints** | Domain rules that code alone cannot convey | `// Floor rounding per regulatory requirement REG-2024-047` |
| **Performance justifications** | Why a seemingly suboptimal approach was chosen | `// Batch size 16 determined via GPU memory profiling on T4 instances` |
| **Empty function bodies** | An empty catch or no-op looks like a mistake | `// Intentionally empty — errors logged by middleware` |
| **Regular expressions** | Regex is never self-documenting | `// Matches ISO 8601 dates: YYYY-MM-DD` |
| **Deprecation markers** | Directs users to the replacement | `// Deprecated: Use NewClient instead.` |
| **API contract comments** | One-sentence summary for tooling (pkgsite, Swagger, IDE hover) — only on truly public/exported APIs | `// Create provisions a new meeting.` |

### The Litmus Test

Before writing any comment, ask these four questions in order:

1. **Can I rename something** to eliminate the need for this comment?
2. **Can I extract a function** whose name explains this logic?
3. **Can I add a type or structural metadata** that makes this comment redundant?
4. **Will this comment still be accurate** after the next AI-assisted refactor?

If the answer to #1-3 is "yes" → refactor instead of commenting.
If the answer to #4 is "probably not" → the comment will become a lie. Don't write it.

---

## When Comments Are Harmful

Comments have a maintenance cost. Every comment is a promise that must be kept in sync with the code. Bad comments are **actively worse** than no comments because they mislead.

### Never Write These

**Restating the code:**
```go
// BAD — says exactly what the code says
// Increment counter by one
counter++

// Get the user by ID
user, err := s.repo.GetUser(ctx, userID)
```

**Duplicating type information:**
```typescript
// BAD — TypeScript types already express this
/** The name of the user (string) */
name: string;

/** Whether the user is active (boolean) */
isActive: boolean;
```

**Duplicating Pydantic Field metadata:**
```python
# BAD — Field(description=...) already handles this
class User(BaseModel):
    """User model with name and email."""
    name: str = Field(description="The user's display name")  
    # The user's display name  ← REDUNDANT, already in Field()
```

**Journal comments:**
```python
# BAD — this is what version control is for
# 2024-01-15: Added retry logic (jane)
# 2024-02-20: Fixed edge case with empty input (bob)
```

**Closing-brace comments:**
```go
// BAD — if you need these, the function is too long
if condition {
    // ... 80 lines ...
} // end if condition
```

**Commented-out code:**
```python
# BAD — this is what version control is for
# def old_implementation():
#     ...
```

**Generic AI-generated comments:**
```typescript
// BAD — AI-generated noise that restates the obvious
/** MeetingCard component that displays a meeting card */
export function MeetingCard({ meeting }: MeetingCardProps) {
```

### The Maintenance Math

Every comment you write today commits a future developer (or agent) to:
- Read it and decide if it's still accurate
- Update it if the code changes
- Delete it if it becomes misleading
- Or worse: leave it and let it mislead the next person

If the comment adds less value than this maintenance cost, it should not exist.

---

## In-Code Markers

In-code markers are the one exception to comment skepticism. They are **actionable, searchable, and trackable** — closer to issue tickets than documentation.

### Standard Markers

| Marker | Meaning | Example |
|--------|---------|---------|
| `TODO(username)` | A planned improvement | `// TODO(alice): Replace with batch insert when pgx adds support` |
| `FIXME(username)` | A known bug or correctness issue | `// FIXME(bob): Race condition under high concurrency — issue #247` |
| `HACK(username)` | A workaround that should be replaced | `// HACK(carol): Workaround for upstream bug in v2.3 — remove after upgrade` |

### Rules

**Always include ownership.** `TODO` without a name is an orphan nobody will fix. `TODO(username)` has accountability.

**Always include context.** `TODO: fix this` is useless. `TODO(alice): Replace with streaming parser when data volumes exceed 1GB — current in-memory approach will OOM` is actionable.

**Link to issue trackers** for non-trivial items: `// FIXME(bob): see #247`

**CI can enforce.** Configure linting to detect unattributed markers and flag them in review.

---

## Language-Specific Conventions

This reference covers universal principles. For language-specific conventions (what to document, structural alternatives, tooling), load the documentation reference from the relevant engineering skill:

| Language | Skill | Key Philosophy |
|----------|-------|---------------|
| **Go** | `core-engineer/references/documentation.md` | One-sentence contract on exports for pkgsite/gopls. Skip obvious getters. Document concurrency semantics. |
| **Python** | `ml-engineer/references/documentation.md` | `Field(description=...)` is the primary documentation layer. One-line docstring on route handlers for Swagger. Skip obvious `__init__`. |
| **TypeScript/React** | `app-engineer/references/documentation.md` | Types + Zod `.describe()` are the primary documentation layer. TSDoc only when types genuinely can't convey intent. |
