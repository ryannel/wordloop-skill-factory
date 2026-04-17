# Document Templates

## Table of Contents
- [README.md](#readmemd)
- [CHANGELOG.md](#changelogmd)
- [Conventional Commits](#conventional-commits)
- [Runbooks](#runbooks)
- [Migration Guides](#migration-guides)
- [Release Notes](#release-notes)
- [Error Catalogs](#error-catalogs)
- [Troubleshooting Guides](#troubleshooting-guides)
- [Glossaries](#glossaries)
- [CONTRIBUTING.md](#contributingmd)

---

## README.md

A README is the front door of every project. Visitors decide within ten seconds whether the project is worth their time. The structure below is designed for maximum scannability — critical information first, detail later.

### The 10-Second Rule

The opening of a README answers three questions immediately:

1. **What** does this project do?
2. **Why** does it matter?
3. **Who** is it for?

If a visitor cannot answer these questions by the time they finish the first screen, the README has failed.

### Structure

```markdown
# Project Name

One-sentence value proposition explaining what, why, and for whom.

<!-- Badges: trust signals only — build status, coverage, license, version -->
[![CI](https://img.shields.io/github/actions/workflow/status/org/repo/ci.yml)](...)
[![Coverage](https://img.shields.io/codecov/c/github/org/repo)](...)
[![License](https://img.shields.io/github/license/org/repo)](...)

<!-- Visual proof: screenshot, GIF, or architecture diagram -->
![Demo](docs/assets/demo.gif)

## Quick Start

Minimal copy-paste commands to go from zero to running:

\```bash
git clone https://github.com/org/repo.git
cd repo
make run
\```

## Features

| Feature | Description |
|---------|-------------|
| **Feature A** | One-line explanation of the value delivered |
| **Feature B** | One-line explanation of the value delivered |

## Usage

\```go
// Minimal working example
client := repo.NewClient(ctx)
result, err := client.DoSomething(ctx, input)
\```

## Documentation

- [Architecture Guide](docs/architecture.md)
- [API Reference](docs/api-reference.md)
- [Contributing](CONTRIBUTING.md)

## License

[MIT](LICENSE)
```

### Principles

**Show, don't tell.** A screenshot or architecture diagram communicates more than five paragraphs of description. Place visual proof immediately after the tagline and badges.

**Badges signal trust, not vanity.** Include badges that indicate project health — build status, test coverage, license, and latest version. Avoid badges that communicate low activity (download counts on new projects) or visual clutter.

**Keep it short.** The README is the entrypoint, not the entire documentation. Link to `docs/` for deep content. If the README exceeds two screens of scrolling, content should be extracted into separate documentation.

**Quickstart is copy-paste ready.** Every command in the Quick Start section is a complete, runnable line. No "replace X with your value" — provide working defaults or environment variable instructions.

---

## CHANGELOG.md

A changelog is a chronological, human-readable record of notable changes. It is written for users and maintainers, not machines. Raw git log output is not a changelog.

### Standard: Keep a Changelog

Follow the Keep a Changelog convention. All entries are grouped under version headers with ISO 8601 dates.

### Change Categories

| Category | When to Use |
|----------|-------------|
| **Added** | New features or capabilities |
| **Changed** | Modifications to existing functionality |
| **Deprecated** | Features marked for removal in future releases |
| **Removed** | Features removed in this release |
| **Fixed** | Bug fixes |
| **Security** | Vulnerability patches (encourages users to upgrade) |

### Template

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added
- User authentication via OAuth 2.1 ([#142](link))

## [1.2.0] - 2026-03-15

### Added
- WebSocket support for real-time notifications ([#138](link))
- Rate limiting middleware with configurable thresholds ([#135](link))

### Changed
- Migrated database driver from pgx/v4 to pgx/v5 ([#140](link))

### Fixed
- Connection pool exhaustion under sustained load ([#139](link))

### Security
- Updated Go to 1.26.1 to address CVE-2026-XXXX ([#141](link))

## [1.1.0] - 2026-02-01
...

[Unreleased]: https://github.com/org/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/org/repo/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/org/repo/releases/tag/v1.1.0
```

### Principles

**Write for humans.** Entries explain the user impact, not the implementation detail. "Added WebSocket support for real-time notifications" is useful. "Refactored handler to use gorilla/websocket" is not.

**Maintain an `[Unreleased]` section.** Track changes as they merge, not at release time. This makes release preparation a simple cut-and-rename operation.

**Link everything.** Every entry links to the PR or issue. Version headers link to the comparison diff on the hosting platform.

**Align with Semantic Versioning.** Breaking changes trigger a major version bump. New features trigger minor. Bug fixes trigger patch.

---

## Conventional Commits

Conventional Commits are a structured format for commit messages that enables automated changelog generation and semantic version determination.

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | Purpose | SemVer Impact |
|------|---------|---------------|
| `feat` | A new feature | Minor |
| `fix` | A bug fix | Patch |
| `docs` | Documentation-only changes | None |
| `style` | Formatting, whitespace (no logic change) | None |
| `refactor` | Code restructuring (no behavior change) | None |
| `perf` | Performance improvement | Patch |
| `test` | Adding or correcting tests | None |
| `chore` | Build process, tooling, dependencies | None |
| `ci` | CI/CD configuration changes | None |

### Breaking Changes

Signal a breaking change in one of two ways:

```
feat!: remove deprecated /v1/users endpoint

BREAKING CHANGE: The /v1/users endpoint has been removed.
Use /v2/users instead. See the migration guide at docs/migration-v2.md.
```

### Style Rules

- **Imperative, present tense:** "add feature" not "added feature" or "adds feature"
- **No capitalized first letter** in the description (after the colon)
- **No period** at the end of the description
- **Scope is optional** but useful for monorepos: `feat(auth): add OAuth 2.1 support`

### Automation Chain

```
Conventional Commits → semantic-release/release-please → CHANGELOG.md + SemVer tag + Release
```

Well-formed commits are the input to automated release pipelines. The commit format is not bureaucracy — it is the API for the release system.

---

## Runbooks

A runbook is a tactical, step-by-step guide for resolving a specific operational issue. It is designed for a stressed, time-pressured responder. Every sentence is an action — not an explanation.

### Template

```markdown
# Runbook: [Issue Title]

**ID:** RUN-XXX
**Severity:** P1 / P2 / P3
**Last Validated:** YYYY-MM-DD
**Owner:** @team-name

## Trigger Conditions

When to execute this runbook:

- Alert `[AlertName]` fires with threshold > X
- User reports symptom: "[Specific symptom description]"
- Dashboard `[DashboardName]` shows metric above/below threshold

## Immediate Actions (First 15 Minutes)

1. **Verify the alert is genuine:**
   \```bash
   kubectl get pods -n production -l app=service-name
   \```
2. **Stabilize the system:**
   \```bash
   kubectl rollout undo deployment/service-name -n production
   \```
3. **Notify stakeholders:**
   - Post in `#incidents` with the alert link
   - Update status page if user-facing

## Diagnosis

Check the following in order:

1. **Recent deployments:**
   \```bash
   kubectl rollout history deployment/service-name -n production
   \```
2. **Resource pressure:**
   \```bash
   kubectl top pods -n production -l app=service-name
   \```
3. **Error logs:**
   \```bash
   kubectl logs -n production -l app=service-name --since=15m | grep ERROR
   \```

## Remediation

### If cause is [Cause A]:
1. Step-by-step fix...
2. Verify with: `[verification command]`

### If cause is [Cause B]:
1. Step-by-step fix...
2. Verify with: `[verification command]`

## Rollback

If remediation fails or worsens the situation:

\```bash
kubectl rollout undo deployment/service-name -n production
\```

## Escalation

| Condition | Contact | Channel |
|-----------|---------|---------|
| Cannot diagnose within 30 min | @senior-engineer | #incidents |
| Data loss suspected | @engineering-lead | Phone + #incidents |

## Post-Incident

- [ ] Update this runbook with any new steps discovered
- [ ] File a post-incident review
- [ ] Create follow-up tickets for permanent fixes
```

### Principles

**Write for the worst moment.** The reader is stressed, potentially sleep-deprived, and operating under time pressure. Commands are copy-paste ready. Steps are numbered. Decisions are explicit.

**Every runbook is a checklist.** Prose is minimized. The format is: do X, verify with Y, if Z then escalate. Cognitive load is the enemy.

**Runbooks rot.** Include a "Last Validated" date and require periodic dry runs. A stale runbook during a real incident is worse than no runbook.

---

## Migration Guides

A migration guide helps users transition from one version, API, or system to another. It anticipates failure and provides rollback paths.

### Template

```markdown
# Migration Guide: v1 → v2

**Timeline:** v1 deprecated YYYY-MM-DD, sunset YYYY-MM-DD
**Estimated effort:** X hours for a typical integration

## Breaking Changes

| Change | v1 (Before) | v2 (After) | Impact |
|--------|-------------|------------|--------|
| Auth method | API Key header | OAuth 2.1 | All clients must update auth |
| Endpoint path | `/v1/users` | `/v2/users` | URL change required |
| Response format | XML | JSON | Response parsing must update |

## Step-by-Step Migration

### 1. Update Authentication
\```diff
- headers: { "X-API-Key": apiKey }
+ headers: { "Authorization": `Bearer ${token}` }
\```

### 2. Update Endpoint URLs
\```diff
- const url = "https://api.example.com/v1/users"
+ const url = "https://api.example.com/v2/users"
\```

### 3. Update Response Parsing
\```diff
- const data = parseXML(response.body)
+ const data = response.json()
\```

## Rollback Plan

If the migration causes issues:

1. Revert to the previous client library version
2. v1 endpoints remain available until [sunset date]
3. Contact support at [channel] for assistance

## Verification

After migrating, verify your integration:

\```bash
curl -H "Authorization: Bearer $TOKEN" https://api.example.com/v2/users
# Expected: 200 OK with JSON response
\```

## FAQ

**Q: Can I use both v1 and v2 simultaneously?**
A: Yes, during the transition period both versions are active.

**Q: What happens after the sunset date?**
A: v1 endpoints will return 410 Gone.
```

---

## Release Notes

Release notes are user-facing, curated summaries optimized for communication rather than completeness. They differ from changelogs in tone and audience.

### Template

```markdown
# Release Notes — v2.0.0

**Release Date:** YYYY-MM-DD

## Highlights

### 🚀 Real-Time Collaboration
Collaborate on documents simultaneously with your team.
Changes sync instantly across all connected clients.

### 🔒 OAuth 2.1 Authentication
We've upgraded our authentication system to OAuth 2.1,
providing stronger security with PKCE support.

## Improvements
- Reduced API response times by 40% through query optimization
- Added dark mode support across all dashboard views

## Bug Fixes
- Fixed an issue where file uploads over 10MB would timeout
- Resolved a race condition in the notification system

## Breaking Changes

> **Action Required:** If you use API v1, please migrate to v2
> before [date]. See the [Migration Guide](docs/migration-v2.md).

## Deprecations
- The `/v1/users` endpoint is now deprecated and will be
  removed in v3.0.0

---
*Full changelog: [v1.5.0...v2.0.0](link)*
```

### Principles

**Lead with value.** Highlights section showcases the most impactful changes first. Use emoji sparingly for visual scanning.

**Breaking changes are unmissable.** Use callout blocks or banners. Link to the migration guide directly.

**Link to the raw changelog.** Release notes are curated — include a link to the full changelog for users who need the complete picture.

---

## Error Catalogs

An error catalog is a lookup reference for every error code the system produces. It serves both end users (debugging) and developers (implementing error handling).

### Template

```markdown
# Error Reference

## Error Format

All API errors follow this structure:

\```json
{
  "error": {
    "code": "AUTH_TOKEN_EXPIRED",
    "message": "Your authentication token has expired.",
    "details": "Tokens are valid for 24 hours. Request a new token via POST /auth/token.",
    "request_id": "req_abc123"
  }
}
\```

## Authentication Errors

### AUTH_TOKEN_EXPIRED
| Field | Value |
|-------|-------|
| **HTTP Status** | 401 Unauthorized |
| **Message** | Your authentication token has expired. |
| **Cause** | The bearer token has exceeded its 24-hour validity window. |
| **Resolution** | Request a new token via `POST /auth/token` with your refresh token. |

### AUTH_INSUFFICIENT_SCOPE
| Field | Value |
|-------|-------|
| **HTTP Status** | 403 Forbidden |
| **Message** | Your token does not have the required scope for this operation. |
| **Cause** | The token was issued with limited scopes that do not include the requested resource. |
| **Resolution** | Request a new token with the `admin:write` scope. |

## Validation Errors

### VALIDATION_FIELD_REQUIRED
...
```

### Principles

**Problem-Cause-Solution.** Every error entry answers three questions: what happened, why it happened, and how to fix it. Never produce an error catalog that only lists codes and messages without resolution steps.

**Never blame the user.** "Your authentication token has expired" is neutral. "You used an invalid token" is accusatory. Maintain a helpful, professional tone.

**Include `request_id`.** Every error response includes a unique identifier that the user can provide to support for debugging. Document this convention.

---

## Troubleshooting Guides

Troubleshooting guides help users self-diagnose and resolve issues. They are organized by symptom, not by internal system component.

### Template

```markdown
# Troubleshooting: [Category]

## [Symptom: What the user observes]

**You see:** Error message or unexpected behavior description
**This means:** Brief explanation of the underlying cause

### Diagnostic Steps

1. **Check [thing]:**
   \```bash
   command to check
   \```
   - If output shows X → proceed to Solution A
   - If output shows Y → proceed to Solution B

### Solution A: [Fix description]

\```bash
steps to fix
\```

**Verify:** Run `[verification command]` — expected output is X.

### Solution B: [Fix description]

...

### Still stuck?

If none of the solutions above resolve the issue:
1. Collect the output of `[diagnostic command]`
2. Open an issue at [link] with the diagnostic output attached
```

### Principles

**Organize by symptom, not by component.** Users know what they see ("API returns 500"), not which internal subsystem is failing. Title troubleshooting entries with the visible symptom.

**Decision trees over paragraphs.** "If X then do Y" is actionable. "The system may sometimes exhibit behavior Z due to various factors" is not.

---

## Glossaries

A glossary is a centralized vocabulary reference that eliminates ambiguity across documentation.

### Template

```markdown
# Glossary

| Term | Definition | Context | Aliases |
|------|-----------|---------|---------|
| **Meeting** | A recorded conversation between two or more participants | Core domain entity — the primary resource in the API | Session, Recording |
| **Synthesis** | An AI-generated summary of a meeting's key points and action items | Output of the ML pipeline — not editable by users | Summary, Recap |
| **Diarization** | The process of segmenting audio by speaker identity | ML pipeline stage — occurs after transcription | Speaker separation |
```

### Principles

**Single source of truth.** The glossary defines the canonical term. All documentation uses that term consistently. Aliases are listed for discoverability but not used in prose.

**Domain terms, not dictionary words.** Only include terms that have a specific meaning in the project's context. "User" may not need a glossary entry; "Synthesis" (with a domain-specific meaning) does.

**Integrate with Vale.** Maintain a Vale substitution rule that enforces glossary terms. If documentation uses an alias where the canonical term should appear, the linter flags it.

---

## CONTRIBUTING.md

A contributing guide removes friction for new contributors. It answers every question a first-time contributor would ask, in the order they would ask it.

### Template

```markdown
# Contributing to [Project Name]

Thank you for your interest in contributing.
This guide covers everything you need to go from zero to merged PR.

## Development Setup

### Prerequisites

- [Language] >= [version]
- [Tool] >= [version]

### Getting Started

\```bash
git clone https://github.com/org/repo.git
cd repo
[install command]
[run command]
\```

## Making Changes

### Branch Naming

\```
feat/short-description
fix/issue-number-description
docs/what-changed
\```

### Commit Messages

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

\```
feat(auth): add OAuth 2.1 token refresh
fix(api): handle null response from upstream service
docs(readme): update quickstart instructions
\```

### Code Style

[Language-specific style guide link or brief rules]

Formatting is enforced automatically:

\```bash
[format command]
[lint command]
\```

## Submitting a Pull Request

1. Fork the repository and create your branch from `main`
2. Make your changes with tests
3. Run the test suite: `[test command]`
4. Push your branch and open a Pull Request
5. Fill in the PR template — describe what and why

### PR Review

- PRs require at least one approving review
- CI must pass (lint, test, build)
- Keep PRs focused — one feature or fix per PR

## Reporting Issues

Use the [issue tracker](link) and include:

- Steps to reproduce
- Expected vs. actual behavior
- Environment details (OS, language version, etc.)
```
