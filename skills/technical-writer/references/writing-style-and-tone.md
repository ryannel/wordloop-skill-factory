# Writing Style & Tone

Technical writing is a precision discipline. Every word is a design decision — it either reduces the reader's cognitive load or adds to it. The goal is not elegant prose; it is efficient, unambiguous information transfer. The same rules that serve human readers also serve AI agents: clear structure, explicit meaning, and zero fluff.

---

## Table of Contents
1. [Core Voice: Assertive and Assistive](#core-voice-assertive-and-assistive)
2. [Active Voice Mandate](#active-voice-mandate)
3. [The Inverted Pyramid](#the-inverted-pyramid)
4. [Plain English](#plain-english)
5. [Cognitive Weight Reduction](#cognitive-weight-reduction)
6. [Hedging Elimination](#hedging-elimination)
7. [Sentence Structure](#sentence-structure)
8. [Documentation-Specific Patterns](#documentation-specific-patterns)
9. [Before/After Examples](#beforeafter-examples)

---

## Core Voice: Assertive and Assistive

Technical documentation uses an **assertive** and **assistive** tone. It states facts directly. It provides actionable guidance. It does not hedge, apologize, or equivocate.

The reader has arrived at this page because they need to accomplish a task. The documentation's job is to help them accomplish it as quickly and confidently as possible. Every sentence either moves the reader toward their goal or gets out of the way.

### Voice Characteristics

| Characteristic | Meaning | Example |
|---------------|---------|---------|
| **Direct** | State facts without qualification | "The API returns a 404 error" (not "The API might return a 404 error") |
| **Instructional** | Tell the reader what to do | "Set the timeout to 30 seconds" (not "You may want to consider setting the timeout") |
| **Confident** | Assert knowledge without hedging | "This configuration enables caching" (not "This configuration should enable caching") |
| **Concise** | Use the fewest words that convey the meaning | "Run `npm install`" (not "You will need to run the command `npm install` in order to install the dependencies") |

---

## Active Voice Mandate

Use active voice in all documentation. Active voice identifies the actor, the action, and the target — in that order. Passive voice obscures the actor, creating ambiguity about who or what performs the action.

### The Rule

| ✅ Active Voice | ❌ Passive Voice |
|-----------------|------------------|
| The API returns a JSON response | A JSON response is returned |
| Configure the timeout in `config.yaml` | The timeout should be configured |
| The SDK validates the request body | The request body is validated by the SDK |
| Run the migration script before deploying | The migration script should be run |

### When Passive Voice Is Acceptable

Passive voice is acceptable only when the actor is genuinely unknown or irrelevant:

- "The file was corrupted during transfer" (the actor is unknown)
- "Deprecated endpoints are removed after two major versions" (the actor is the system, which is obvious)

If you can identify the actor, use active voice.

---

## The Inverted Pyramid

The Inverted Pyramid is a structural pattern borrowed from journalism. It places the most critical information at the top and progressively layers supporting detail below. This structure serves both human readers (who scan from the top) and AI agents (who may truncate from the bottom).

### Structure

```
┌─────────────────────────────────┐
│          CRITICAL INFO          │  ← Answer the question immediately
│     (What do I need to know?)   │
├─────────────────────────────────┤
│       SUPPORTING DETAIL         │  ← How does it work? What are the options?
│   (How does this work?)         │
├─────────────────────────────────┤
│     BACKGROUND / CONTEXT        │  ← Why was this designed this way?
│   (Why is it this way?)         │
└─────────────────────────────────┘
```

### Application

- **Page level:** The first paragraph of every page answers the page's core question. A reader who reads only the first paragraph should understand the essential message.
- **Section level:** Each section opens with a summary sentence that captures the section's key point.
- **Paragraph level:** The first sentence of each paragraph contains the paragraph's main idea. Supporting sentences follow.

### Example

**❌ Bottom-up structure (buries the answer):**
> The team evaluated several approaches to authentication including session tokens, API keys, and OAuth. After considering the security requirements and the need for third-party integrations, we decided to use OAuth 2.1 with PKCE for all public-facing endpoints.

**✅ Inverted Pyramid (leads with the answer):**
> Use OAuth 2.1 with PKCE for all public-facing endpoint authentication. This approach was selected over session tokens and API keys because it satisfies both the security requirements and the need for third-party integrations.

---

## Plain English

Use simple, common words. Technical documentation is read by native and non-native English speakers, by developers with varying expertise levels, and by AI models that interpret text most reliably when it is unambiguous.

### Word Substitutions

| ❌ Complex | ✅ Simple |
|-----------|----------|
| utilize | use |
| facilitate | help |
| leverage | use |
| in order to | to |
| prior to | before |
| subsequent to | after |
| commence | start |
| terminate | stop / end |
| endeavor | try |
| ascertain | find out / determine |
| aforementioned | this / that |
| notwithstanding | despite |
| heretofore | until now |

### Jargon Management

- **Define terms on first use:** When a domain-specific term is necessary (e.g., "idempotent"), define it inline on first use and add it to the glossary.
- **Prefer descriptive phrases:** Use "exactly-once delivery" instead of "idempotent message processing" when the audience may not know the term.
- **Avoid acronym overload:** Spell out acronyms on first use. Do not assume the reader knows that TTFV means "Time-to-First-Value."

---

## Cognitive Weight Reduction

Every element on a documentation page has a cognitive cost. Readers expend mental energy processing words, structure, and layout. The goal is to minimize the total cognitive weight while preserving all necessary information.

### Strategies

| Strategy | Implementation |
|----------|----------------|
| **Front-load keywords** | Start sentences with the most important noun or verb. "Firestore stores meeting data" not "For the purpose of storing meeting data, we use Firestore." |
| **One idea per sentence** | Each sentence conveys exactly one point. If a sentence contains "and" or "but" connecting two distinct ideas, split it. |
| **One topic per paragraph** | Each paragraph covers one concept. If a paragraph discusses both configuration and troubleshooting, split it. |
| **Lists over prose** | When presenting three or more items, use a bullet list. Lists are faster to scan and easier for agents to parse. |
| **Tables over lists** | When items have multiple attributes, use a table. Tables expose structure that prose and lists hide. |
| **Code over description** | When explaining a configuration, show the config file. "Set `timeout: 30s` in `config.yaml`" is clearer than a paragraph of prose. |

### Scanning Optimization

Most readers scan documentation — they do not read linearly. Optimize for scanning:

- **Bold key terms** at the start of list items so readers can find relevant items without reading every word.
- **Use descriptive headings** that answer a question: "How to configure authentication" not "Configuration" or "Section 3."
- **Include tldr summaries** at the top of long pages: a 2-3 sentence summary that tells the reader whether this page answers their question.

---

## Hedging Elimination

Hedging phrases reduce reader confidence and introduce ambiguity. They signal uncertainty from the author, which the reader interprets as uncertainty about the system.

### Common Hedges to Remove

| ❌ Hedging | ✅ Direct |
|-----------|----------|
| "It should work" | "It works" (or describe the conditions under which it does not) |
| "You might want to" | "Do this" (or explain when to do it) |
| "This could potentially cause" | "This causes" (state the condition) |
| "Basically, it..." | State the fact directly |
| "In most cases" | "By default" (or specify the exception) |
| "It is generally recommended" | "Do this" (or explain who recommends it and why) |
| "Please note that" | Remove the filler; state the note directly |
| "It is important to remember" | State the fact; if it is important, the reader will see why |

### Why This Matters

When documentation says "the API should return a 200 response," the reader does not know whether it *does* return 200 or whether there are conditions where it does not. The hedge creates doubt. Either the API returns 200 (state it as fact) or it sometimes does not (describe the conditions).

---

## Sentence Structure

### Length

- **Target:** 15-25 words per sentence. Shorter sentences are easier to parse for both humans and AI agents.
- **Maximum:** 35 words. Sentences longer than this should be split.
- **Headlines and summaries:** 8-12 words.

### Structure Patterns

- **Imperative for instructions:** "Run the migration script." "Set the environment variable." "Deploy the service."
- **Declarative for descriptions:** "The API uses OAuth 2.1 for authentication." "Firestore stores meeting data as documents."
- **Conditional for branching:** "If the request fails, the SDK retries three times." Place the condition first so the reader can skip the sentence if the condition does not apply to them.

### Parallelism

When listing items or describing steps, use parallel grammatical structure:

**❌ Non-parallel:**
- Configure the database
- You need to set up authentication
- Running the migration is important

**✅ Parallel:**
- Configure the database
- Set up authentication
- Run the migration

---

## Documentation-Specific Patterns

### Procedure Steps

Number each step. Start each step with an imperative verb. Include the expected outcome.

```markdown
1. Run the database migration:
   ```bash
   npm run migrate
   ```
   The output shows `Migration complete: 3 tables created`.

2. Set the API key environment variable:
   ```bash
   export API_KEY="your-key-here"
   ```

3. Start the development server:
   ```bash
   npm run dev
   ```
   The server starts on `http://localhost:3000`.
```

### Admonitions

Use admonitions (notes, warnings, tips) sparingly. When everything is a note, nothing is. Reserve them for information that genuinely changes the reader's behavior:

- **Note:** Background context that helps understanding but is not critical to completing the task.
- **Tip:** A shortcut or optimization the reader might not discover on their own.
- **Warning:** An action that could cause data loss, downtime, or irreversible state changes.
- **Caution:** A common mistake that causes confusion or subtle bugs.

### Cross-References

Link to other documentation pages when the reader needs that information to continue. Do not link speculatively ("for more information, see X, Y, and Z"). Every link should answer the question: "Does the reader need this right now?"

---

## Before/After Examples

### Example 1: Configuration Guide

**❌ Before:**
> In order to properly configure the application for production use, you will need to ensure that the following environment variables have been set. It is generally recommended that you use a `.env` file for this purpose, although there are other approaches that could potentially work as well. The `DATABASE_URL` variable should be set to point to your production database instance. You might also want to configure the `LOG_LEVEL` variable, which controls the verbosity of the application's logging output.

**✅ After:**
> Set these environment variables in a `.env` file before deploying to production:
>
> | Variable | Required | Description |
> |----------|----------|-------------|
> | `DATABASE_URL` | Yes | Connection string for the production database |
> | `LOG_LEVEL` | No | Logging verbosity: `debug`, `info`, `warn`, `error` (default: `info`) |

### Example 2: Error Documentation

**❌ Before:**
> If you receive an error that says something like "Connection refused", this might indicate that the database server is not running or that the connection parameters are incorrect. You should probably check that the database is up and verify the connection string.

**✅ After:**
> **Error: `Connection refused`**
>
> The application cannot reach the database. Check:
> 1. The database server is running: `pg_isready -h localhost -p 5432`
> 2. The `DATABASE_URL` matches the database's host, port, and credentials
> 3. No firewall rules block the connection

### Example 3: Feature Description

**❌ Before:**
> The real-time synchronization feature basically allows multiple users to collaborate on the same document simultaneously. When one user makes a change, it is basically propagated to all other connected users in real-time. This is facilitated through the use of WebSocket connections that maintain a persistent connection between the client and the server.

**✅ After:**
> Real-time synchronization propagates document changes to all connected users via WebSocket. When one user edits a document, the server broadcasts the change to every other connected client within 50ms.
