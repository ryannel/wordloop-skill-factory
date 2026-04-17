# Information Architecture

Documentation sites serve two fundamentally different user groups with opposing needs: new users who need guided linear paths, and experienced users who need fast, targeted retrieval. Designing for both requires a segmented architecture that serves each group through dedicated pathways while sharing a single source of truth.

---

## Table of Contents
1. [The Two User Groups](#the-two-user-groups)
2. [Onboarding Path: New Users](#onboarding-path-new-users)
3. [Expert Path: Experienced Users](#expert-path-experienced-users)
4. [Progressive Disclosure](#progressive-disclosure)
5. [Navigation Design](#navigation-design)
6. [Semantic Search](#semantic-search)
7. [Time-to-First-Value Optimization](#time-to-first-value-optimization)
8. [Content Hierarchy Patterns](#content-hierarchy-patterns)
9. [Micro-Interactions and Feedback](#micro-interactions-and-feedback)
10. [Site Map Template](#site-map-template)

---

## The Two User Groups

| Attribute | New Users | Experienced Users |
|-----------|-----------|-------------------|
| **Goal** | Learn the basics, get something working | Find a specific answer, verify a detail |
| **Behavior** | Read linearly, follow tutorials step-by-step | Search, scan, jump to the relevant section |
| **Tolerance** | Low frustration threshold, will abandon if confused | High frustration threshold, but will leave if the answer takes too long to find |
| **Content need** | Tutorials, quickstarts, glossaries | API references, configuration tables, code snippets |
| **Navigation** | Guided (Next/Previous, checklists) | Direct (search, sidebar, deep links) |

Designing for one group at the expense of the other is a documentation antipattern. A site that is all tutorials frustrates experts. A site that is all reference frustrates newcomers. The solution is segmentation: parallel pathways that share the same underlying content but present it through different lenses.

---

## Onboarding Path: New Users

The onboarding path guides new users from zero knowledge to their first successful interaction. It is linear, sequential, and teaches by doing — not by describing.

### Design Principles

- **Show, don't tell:** Every concept is demonstrated through a concrete, runnable example. Explanations follow demonstrations, not the reverse.
- **One concept per step:** Each step in a tutorial introduces exactly one new idea. A step that introduces authentication *and* pagination is two steps.
- **Observable outcomes:** Every step produces a visible result the user can verify: a server response, a rendered page, a passing test. Steps without observable outcomes leave the user uncertain whether they are on the right track.
- **Error anticipation:** Document the most common errors at each step, with explicit recovery instructions. New users encountering undocumented errors will abandon the tutorial.

### Tutorial Structure

```markdown
# Tutorial: [Goal in User's Words]

## What You'll Build
[One paragraph + screenshot/diagram of the finished result]

## Prerequisites
- [Specific version of required tool]
- [Account/access requirement]
- [Estimated time: X minutes]

## Step 1: [Verb + Object]
[Instruction]
[Code block or command]
[Expected outcome: what the user should see]

## Step 2: [Verb + Object]
...

## What You Learned
[Summary of concepts covered]

## Next Steps
- [Link to the next tutorial]
- [Link to the relevant reference docs]
```

### Interactive Tutorials

Augment static tutorials with interactive elements:

- **Click-through guides:** Overlay-based walkthroughs that highlight UI elements and guide the user through the interface. Use tools like Intro.js or Shepherd.js.
- **Embedded sandboxes:** Code sandboxes (StackBlitz, CodeSandbox) embedded directly in the tutorial page so the user can modify and run code without leaving the docs.
- **Personalized checklists:** Track the user's progress through onboarding steps. Show completed steps with checkmarks and highlight the next action. Persist progress across sessions.

---

## Expert Path: Experienced Users

The expert path provides fast, targeted access to specific information. Experts arrive with a question and need the answer with minimal friction.

### Design Principles

- **Search-first:** Experts use search, not navigation. The search experience is the primary interface for experienced users.
- **Scannable structure:** Pages use headings, tables, and code blocks that are optimized for scanning rather than linear reading.
- **Zero context-switching:** All information needed to answer a question is on a single page. Avoid requiring experts to click through three pages to find a configuration option, its default value, and its effect.
- **Side-by-side code blocks:** When showing code in multiple languages, use tabbed or side-by-side layouts so experts can compare implementations without scrolling.

### Reference Page Structure

```markdown
# [Resource/Feature Name]

> [One-line summary: what this is and when to use it]

## Overview
[2-3 sentences expanding the summary]

## Configuration

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `timeout` | `duration` | `30s` | Request timeout for downstream calls |
| `retries` | `integer` | `3` | Maximum retry attempts |
| ...    | ...  | ...     | ...         |

## Usage Examples

### [Common Use Case 1]
```[language]
[Complete, runnable code example]
```

### [Common Use Case 2]
```[language]
[Complete, runnable code example]
```

## Error Reference

| Error Code | Cause | Resolution |
|------------|-------|------------|
| `TIMEOUT_EXCEEDED` | Request exceeded the configured timeout | Increase `timeout` or optimize the downstream service |
| ...        | ...   | ...        |

## Related
- [Link to related resource]
- [Link to relevant ADR]
```

---

## Progressive Disclosure

Progressive disclosure layers information so that the reader sees the most essential content first and can optionally drill into deeper detail. This serves both audiences: new users are not overwhelmed by advanced options, and experts can quickly drill into specifics.

### Implementation Patterns

| Pattern | Mechanism | Use Case |
|---------|-----------|----------|
| **Collapsed sections** | `<details>` / accordion UI | Advanced configuration options, verbose error explanations |
| **Tabbed content** | Tab groups (language-specific examples, platform-specific instructions) | Code snippets in multiple languages |
| **Layered pages** | Quickstart → Guide → Reference → Internals | Different depths of the same topic |
| **Inline expansion** | "Learn more" links that expand content in place | Definitions, background context |

### Layered Content Hierarchy

Structure documentation so that each layer adds depth:

```
Layer 1: Quickstart     → "Get it running in 5 minutes" (copy-paste commands)
Layer 2: Guide          → "Understand how it works" (concepts + examples)
Layer 3: Reference      → "Find the exact option" (tables, schemas, API details)
Layer 4: Internals      → "Understand the architecture" (ADRs, design docs)
```

A reader can stop at any layer and still have a functional understanding. Each layer is self-contained — it does not assume the reader has read the previous layer, though it may link to it for deeper context.

---

## Navigation Design

### Sidebar Navigation

- **Group by user task**, not by implementation structure. "Authentication" is a better sidebar entry than "AuthService" because it describes what the user is trying to do, not how the code is organized.
- **Limit depth to three levels.** Deeper nesting becomes unusable. If the sidebar hierarchy goes beyond three levels, the information architecture needs restructuring.
- **Indicate content type** with subtle visual cues: icons for tutorials (📘), reference (📋), guides (📖), and ADRs (📐).

### Breadcrumb Navigation

Every page includes a breadcrumb trail showing the path from the root. Breadcrumbs serve two purposes:
1. **Orientation:** The reader always knows where they are in the hierarchy.
2. **Navigation:** The reader can jump to any parent level without using the sidebar.

### Next/Previous Links

Tutorial and Guide pages include explicit Next and Previous links at the bottom. These reinforce the linear path for new users while being ignorable by experts.

---

## Semantic Search

Keyword-based search is insufficient for technical documentation. Implement semantic search that retrieves results based on meaning rather than exact word matches.

### Requirements

- **Natural language queries:** Users can search for "how do I authenticate with the API?" and get results for the authentication guide, even if the page title is "OAuth 2.1 Configuration."
- **Snippet extraction:** Search results show the most relevant paragraph from the page, not just the page title and description.
- **Faceted filtering:** Users can filter results by content type (tutorial, reference, guide, ADR), API version, or topic area.
- **Ranking by relevance** (not recency): A comprehensive guide on authentication should rank higher than a changelog entry that mentions authentication in passing.

### Implementation Options

| Tool | Approach | Best For |
|------|----------|----------|
| **Algolia DocSearch** | Managed semantic search with Markdown indexing | Fast setup, good default relevance |
| **Typesense** | Open-source, self-hosted vector + keyword hybrid search | Privacy-sensitive or self-hosted requirements |
| **Orama** | Client-side, zero-dependency search | Small-to-medium documentation sets |

---

## Time-to-First-Value Optimization

Time-to-First-Value (TTFV) measures the elapsed time from a user's first visit to the documentation until they achieve a meaningful outcome (e.g., a running application, a successful API call, a configured service). Every information architecture decision is evaluated against its impact on TTFV.

### TTFV Reduction Strategies

| Strategy | Impact | Implementation |
|----------|--------|----------------|
| **Landing page quickstart** | The first thing a new user sees is a 3-step quickstart, not a feature list | Place the quickstart above the fold on the docs home page |
| **Prerequisite reduction** | Minimize setup requirements | Auto-provision sandbox environments, provide Docker-based setups |
| **Copy-paste fidelity** | Every code snippet works when pasted directly | Test all snippets in CI against the current API/SDK version |
| **Progress indicators** | Show the user how far they are in the tutorial | Step counters, progress bars, checklists |
| **Escape hatches** | Provide "skip to the end" links for users who want to see the final result | Link to a working example repository at the top of every tutorial |

---

## Content Hierarchy Patterns

### The Diátaxis Framework

Organize content into four distinct types based on user needs:

| Type | User Need | Content Approach |
|------|-----------|-----------------|
| **Tutorial** | Learning | Step-by-step, guided, hands-on |
| **How-to Guide** | Solving a problem | Goal-oriented, practical, assumes knowledge |
| **Reference** | Information retrieval | Accurate, complete, structured |
| **Explanation** | Understanding | Conceptual, context-giving, discursive |

Each content type is a separate section in the documentation site. Content does not mix types — a tutorial that interrupts the walkthrough with a lengthy conceptual explanation violates the framework and confuses the reader.

---

## Micro-Interactions and Feedback

Documentation sites benefit from subtle micro-interactions that provide instant visual feedback and reduce psychological friction:

- **Hover previews:** Hovering over a cross-reference link shows a tooltip preview of the target page's summary.
- **Copy confirmation:** Clicking a "copy" button on a code snippet shows a brief "Copied!" confirmation.
- **Scroll progress:** A progress indicator at the top of long pages shows how far the reader has scrolled.
- **Collapse memory:** Accordion sections remember their open/closed state across page loads.
- **Theme synchronization:** Documentation respects the user's system light/dark mode preference and allows manual override.

These interactions are not decorative — they reduce the micro-frictions that accumulate during an extended documentation session and cause reader fatigue.

---

## Site Map Template

```
docs/
├── index.md                    # Landing page with quickstart
├── getting-started/
│   ├── quickstart.md           # 5-minute first experience
│   ├── installation.md         # Platform-specific setup
│   └── first-project.md        # Guided first project tutorial
├── tutorials/
│   ├── tutorial-1.md           # Step-by-step guided learning
│   ├── tutorial-2.md
│   └── tutorial-3.md
├── guides/
│   ├── authentication.md       # Goal-oriented: "How to authenticate"
│   ├── error-handling.md       # Goal-oriented: "How to handle errors"
│   └── deployment.md           # Goal-oriented: "How to deploy"
├── reference/
│   ├── api/
│   │   ├── openapi.yaml        # Specification (source of truth)
│   │   └── endpoints.md        # Generated endpoint reference
│   ├── configuration.md        # Complete option reference
│   ├── cli.md                  # CLI command reference
│   └── errors.md               # Error code catalog
├── architecture/
│   ├── overview.md             # System context & container diagrams
│   ├── adrs/                   # Architecture Decision Records
│   │   ├── index.md            # ADR log (chronological)
│   │   ├── 001-database.md
│   │   └── 002-auth.md
│   └── diagrams/               # Diagram source files
│       ├── context.mmd
│       └── containers.d2
├── llms.txt                    # AI-native documentation index
├── llms-full.txt               # Aggregated full documentation
└── glossary.md                 # Project-specific term definitions
```

---

## Content Reuse & Single-Sourcing

Documentation grows faster than teams can maintain it. Content reuse prevents the duplication that causes documentation drift — where the same concept is explained differently in three places, with two of the explanations now outdated.

### The Problem

Without content reuse, the same installation instructions appear in the README, the quickstart guide, and the contributing guide. When the install process changes, one copy gets updated and two do not. Users encounter contradictory instructions and lose trust.

### Principles

**Write once, reference everywhere.** Any content that appears in more than one place is a candidate for extraction into a shared module.

**Modules are self-contained.** A reusable content block makes sense in isolation. It has its own heading and does not assume the reader has read surrounding context.

**Link, don't duplicate.** When full inclusion is not supported by the toolchain, link to the canonical source rather than copying the content.

### Implementation Patterns

**MDX Partials (Docusaurus, Nextra):**
```mdx
// docs/_shared/prerequisites.mdx
## Prerequisites
- Go >= 1.26
- Docker >= 24.0
- `make` installed

// docs/quickstart.md
import Prerequisites from './_shared/prerequisites.mdx';

# Quickstart
<Prerequisites />
```

**Markdown Includes (MkDocs):**
```markdown
# Quickstart

{%% include "shared/prerequisites.md" %%}
```

**Snippet References (GitHub-flavored linking):**
```markdown
<!-- Reference a specific section from another file -->
See [Prerequisites](../CONTRIBUTING.md#prerequisites) for setup instructions.
```

### What to Reuse

| Content Type | Reuse Pattern | Example |
|-------------|---------------|---------|
| **Prerequisites** | Shared partial imported into quickstart, tutorials, contributing guide | Software version requirements |
| **Environment variables** | Single reference table imported wherever configuration is discussed | `.env` file documentation |
| **Common warnings** | Admonition partial reused across relevant pages | "This operation is destructive and cannot be undone" |
| **API authentication** | Auth setup steps shared between quickstart and API reference | OAuth token acquisition flow |
| **Code examples** | Tested code in the test suite, referenced by line range in docs | Working examples that CI validates |

### What NOT to Reuse

Not all duplication is bad. Sometimes similar content serves different audiences and should be written independently:

- **Tutorials vs. reference:** A tutorial may explain authentication step-by-step for beginners, while the reference describes the same mechanism tersely for experts. These serve different purposes and should not be the same content module.
- **Context-dependent content:** If a paragraph only makes sense in the context of one specific page, extracting it into a reusable module creates a brittle dependency.

