# Documentation Antipatterns

Antipatterns are recurring practices that appear productive but consistently lead to documentation that rots, misleads, or actively harms developer productivity. Recognizing and avoiding them is as important as following best practices — because a single antipattern can undermine an otherwise well-designed documentation system.

---

## Table of Contents
1. [Platform Fragmentation](#platform-fragmentation)
2. [Static/PDF Documentation](#staticpdf-documentation)
3. [Single-Audience Design](#single-audience-design)
4. [Manual SDK Maintenance](#manual-sdk-maintenance)
5. [Undocumented Diagrams](#undocumented-diagrams)
6. [Documentation Drift](#documentation-drift)
7. [Coverage-Without-Quality Linting](#coverage-without-quality-linting)
8. [Mutable ADRs](#mutable-adrs)
9. [Friction-Heavy Security Documentation](#friction-heavy-security-documentation)
10. [The Knowledge Silo](#the-knowledge-silo)
11. [Detection Checklist](#detection-checklist)

---

## Platform Fragmentation

### Symptom
Documentation, tools, and workflows are scattered across disconnected systems — Confluence pages, Google Docs, Notion databases, README files, Slack pins, and tribal knowledge. No single system is the canonical source of truth.

### Root Cause
Teams adopt documentation tools independently without a centralized standard. Each tool solves a local problem, but the aggregate effect is a fragmented landscape where no one knows which document is current.

### Consequence
- Developers cannot find authoritative information. They ask on Slack because searching four platforms is slower than asking a colleague.
- "Shadow Systems" emerge: teams build unofficial tracking methods (spreadsheets, personal notes) because the official docs are unreliable.
- New team members spend days discovering which documentation source is trustworthy and which is stale.

### Fix
- Establish one canonical documentation platform (Markdown in Git).
- Redirect all other sources to the canonical platform. If content exists in Confluence or Google Docs, migrate it or link to the canonical version.
- Enforce a "single source of truth" policy: if information exists in two places, one of them is wrong.

---

## Static/PDF Documentation

### Symptom
Documentation is produced as PDF manuals, exported Word documents, or static HTML snapshots that are distributed via email, shared drives, or download links.

### Root Cause
Legacy processes that treat documentation as a deliverable artifact rather than a living product. Regulatory or contractual requirements that mandate "document delivery" in traditional formats.

### Consequence
- Static artifacts cannot be version-controlled, indexed by AI agents, or updated via Git-based workflows.
- Users have no way to know whether their copy is current. Version confusion leads to incorrect information being applied.
- PDFs are invisible to search engines and AI tools. Content locked in PDFs is effectively undiscoverable.
- Feedback loops are broken: there is no way for a reader to report an error or suggest an improvement inline.

### Fix
- Migrate all documentation to Markdown in version control.
- If regulatory requirements mandate PDF delivery, generate PDFs from the Markdown source using automated build pipelines (Pandoc, Weasyprint). The Markdown remains the source of truth; the PDF is a generated artifact.
- Implement `llms.txt` to make the Markdown-based documentation discoverable by AI agents.

---

## Single-Audience Design

### Symptom
Documentation is designed exclusively for human readers. There is no `llms.txt`, no structured metadata, no MCP server, and no consideration for how AI agents consume the content.

### Root Cause
Documentation was designed before AI agents became primary consumers. The team has not updated its information architecture to reflect the dual-audience reality.

### Consequence
- AI agents resort to HTML scraping, which is brittle, slow, and produces unreliable context.
- AI-powered IDE integrations and coding assistants cannot leverage the documentation, forcing developers to switch between the IDE and the browser.
- The documentation misses the opportunity to be the authoritative source for AI-generated responses, ceding accuracy to whatever the AI model learned during training (which may be outdated or incorrect).

### Fix
- Implement `llms.txt` and `llms-full.txt` at the repository root.
-  Use semantic Markdown with explicit heading hierarchies.
- Add YAML frontmatter to every page for programmatic metadata access.
- Consider implementing an MCP documentation server for protocol-based retrieval.
- See `references/ai-native-documentation.md` for full implementation guidance.

---

## Manual SDK Maintenance

### Symptom
Client SDK libraries are written and maintained by hand. When the API changes, a developer manually updates the SDK code, documentation, and examples.

### Root Cause
The API specification is not treated as the single source of truth, or the team is unaware of automated SDK generation tools.

### Consequence
- SDKs drift from the API spec within days of any change. The TypeScript SDK accepts a parameter that the API no longer supports. The Python SDK is missing a new endpoint that was added two weeks ago.
- SDK bugs are reported by users, not caught by automation. The support burden grows linearly with the number of supported languages.
- Documentation of SDK methods is inconsistent across languages because each SDK is documented independently.

### Fix
- Adopt an automated SDK generator (Fern, Stainless, openapi-generator) that produces client libraries from the OpenAPI spec.
- Integrate SDK generation into the CI/CD pipeline: when the spec changes, SDKs are regenerated and published automatically.
- See `references/api-documentation-ecosystems.md` for pipeline configuration.

---

## Undocumented Diagrams

### Symptom
System architecture diagrams exist as PNG or JPEG images stored outside version control — in Confluence pages, Google Slides, Miro boards, or Lucidchart. They are not referenced from any Markdown documentation, and their source files (if they exist) are in proprietary formats.

### Root Cause
Diagrams were created in visual tools during design meetings and never converted to text-based formats. The tools used for creation (Lucidchart, Draw.io, Miro) produce binary files that cannot be meaningfully diffed or reviewed.

### Consequence
- Diagrams cannot be reviewed in pull requests. Architecture changes are merged without diagram updates because the diagram source is in a different tool.
- Drift between diagrams and implementation is invisible. There is no CI check that can compare a PNG against the codebase.
- Knowledge is locked in proprietary tool formats. If the team cancels the Lucidchart subscription, the diagram source is lost.

### Fix
- Convert existing diagrams to text-based formats (Mermaid.js or D2).
- Store diagram source files alongside the code they document, in version control.
- Establish PR review norms that require diagram updates when architecture changes.
- See `references/diagrams-as-code.md` for tooling and workflows.

---

## Documentation Drift

### Symptom
Documentation describes a system that used to exist. Endpoints documented in the API reference have been removed. Configuration options listed in the guide have been renamed. Tutorials reference CLI commands that no longer work.

### Root Cause
Documentation updates are not coupled to code changes. Developers change the code in one PR and plan to "update the docs later" — which never happens, because there is no structural mechanism to ensure it does.

### Consequence
- Users follow incorrect instructions and encounter confusing errors. Support tickets increase.
- Confidence in the documentation erodes. Users stop consulting the docs and go directly to reading the source code or asking on Slack — defeating the purpose of documentation entirely.
- AI agents that rely on the documentation produce incorrect answers, amplifying the impact of stale content.

### Fix
- **Couple docs to code:** Use CODEOWNERS rules to require documentation review when API surface, configuration, or architecture changes.
- **Automate staleness detection:** Compare documentation file modification dates against related source code dates. Flag stale pages in CI.
- **Generate from specs:** For API documentation, generate from the OpenAPI spec so that drift is structurally impossible.
- **Test documentation:** Run code snippets from the docs against the current API/SDK in CI. If they fail, the docs are wrong.

---

## Coverage-Without-Quality Linting

### Symptom
Vale is installed and running in CI, but it uses only default or community rules without any project-specific customization. The linting passes, but the documentation still contains inconsistent terminology, incorrect brand names, and tone violations.

### Root Cause
The team installed Vale as a checkbox compliance measure without investing in custom rules that reflect the project's actual standards.

### Consequence
- A false sense of quality: "Vale is green" does not mean the documentation is good.
- Inconsistencies that custom rules would catch (e.g., "route" vs. "endpoint," "JavaScript" vs. "Javascript") persist across the entire docs site.
- The linting tool adds friction (CI time, review noise) without proportionate value.

### Fix
- Author custom Vale rules for project-specific terminology, brand names, and tone.
- Configure vocabularies for domain-specific terms.
- Calibrate rule severity: errors for objectively wrong content, warnings for style preferences, suggestions for optional improvements.
- See `references/quality-governance.md` for custom rule templates and configuration.

---

## Mutable ADRs

### Symptom
When an architectural decision changes, the team edits the original ADR to reflect the new decision. There is no historical record of the previous decision or the reasoning behind the change.

### Root Cause
The team treats ADRs as living documents (like design docs) rather than immutable records (like a financial ledger).

### Consequence
- Decision history is lost. No one can reconstruct why the original decision was made or what changed.
- Patterns and trends are invisible. If the team repeatedly changes database choices, the pattern is hidden because each edit overwrites the previous context.
- Accountability is eroded. When a decision goes wrong, no one can trace back to the original reasoning and constraints.

### Fix
- Adopt append-only governance: accepted ADRs are never modified.
- Create a new ADR with "Supersedes: ADR-XXX" when a decision changes.
- Preserve rejected ADRs to document why alternatives were not chosen.
- See `references/debt-aware-adrs.md` for the full ADR lifecycle and templates.

---

## Friction-Heavy Security Documentation

### Symptom
Security documentation and access procedures are so complex that developers avoid following them. Authentication guides span 20 pages. API key generation requires manual approval from three teams.

### Root Cause
Security policies prioritize thoroughness over usability. Documentation authors assume that more detail means better security, when in practice, excessive friction drives engineers to find workarounds.

### Consequence
- Developers bypass security procedures because following them is too slow, creating security vulnerabilities.
- "Alert Fatigue" numbs engineers to important security warnings because they are buried among dozens of minor alerts.
- New developers cannot onboard securely without hand-holding from a security team member.

### Fix
- Apply the same writing principles to security documentation as to all other documentation: Inverted Pyramid, active voice, copy-paste commands.
- Replace lengthy prose with decision trees: "If you need X, do Y. If you need Z, do W."
- Automate security-sensitive workflows (key rotation, access provisioning) so the documentation describes a one-command process rather than a multi-page procedure.
- Replace friction-heavy authentication (static MFA codes) with low-friction alternatives (hardware-based FIDO2 keys, number-matching push notifications).

---

## The Knowledge Silo

### Symptom
Critical system knowledge exists only in the heads of specific engineers. When those engineers are unavailable (vacation, departure, different timezone), the team cannot operate the system confidently.

### Root Cause
Knowledge transfer is treated as a social process (mentoring, pair programming) rather than a documentation process (writing it down). The team relies on oral tradition instead of written records.

### Consequence
- Bus factor of 1: a single person's departure creates a knowledge crisis.
- Incident response is slow when the domain expert is unavailable.
- New team members take months to become productive because they must extract knowledge through conversation rather than reading.

### Fix
- Mandate ADRs for every significant decision. The act of writing the ADR forces knowledge externalization.
- Write runbooks for operational procedures. If a procedure requires expert knowledge to execute, it is not documented well enough.
- Conduct "documentation interviews" where a non-expert attempts to follow existing documentation and records where they get stuck. The failure points are the documentation gaps.

---

## Detection Checklist

Use this checklist to assess whether a project's documentation has active antipatterns:

```markdown
## Documentation Health Check

### Fragmentation
- [ ] Is there exactly one canonical source for each piece of information?
- [ ] Are all documentation sources in version control?

### Currency
- [ ] When was the oldest page last updated? (Red flag: > 6 months)
- [ ] Is there automated staleness detection in CI?
- [ ] Are documentation updates required in code review for API/architecture changes?

### Dual-Audience
- [ ] Does llms.txt exist at the repository root?
- [ ] Does every page use semantic heading hierarchy (H1 → H2 → H3)?
- [ ] Is there YAML frontmatter on every documentation page?

### API Ecosystem
- [ ] Are SDKs generated from the specification, not maintained manually?
- [ ] Is there a live API playground?
- [ ] Does CI validate that the API server conforms to the specification?

### Visual Engineering
- [ ] Are all architecture diagrams in text-based formats (Mermaid, D2)?
- [ ] Are diagram source files in version control?
- [ ] Is there a drift detection mechanism?

### Governance
- [ ] Are ADRs append-only (never modified after acceptance)?
- [ ] Does every accepted ADR have a review date?
- [ ] Is Vale running in CI with project-specific custom rules?

### Debt Management
- [ ] Is there a documentation debt inventory?
- [ ] Is 15-20% of sprint capacity allocated to debt remediation?
- [ ] Are friction metrics (MTTC, MTTFV) being tracked?
```
