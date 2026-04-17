# Quality Governance

Documentation quality is enforced through automation, not manual review. Vale is the primary linting tool — a syntax-aware prose linter that scans Markdown files for style, terminology, and formatting violations. Integrated into Git hooks and CI/CD pipelines, Vale catches errors at `git push` speed, before they reach a human reviewer.

---

## Table of Contents
1. [Why Automate Quality](#why-automate-quality)
2. [Vale Configuration](#vale-configuration)
3. [Style Packages](#style-packages)
4. [Custom Rule Authoring](#custom-rule-authoring)
5. [Terminology Enforcement](#terminology-enforcement)
6. [Brand Consistency](#brand-consistency)
7. [Clarity Rules](#clarity-rules)
8. [Inclusive Language](#inclusive-language)
9. [Git Hook Integration](#git-hook-integration)
10. [CI/CD Pipeline Integration](#cicd-pipeline-integration)
11. [Managing False Positives](#managing-false-positives)
12. [Templates](#templates)

---

## Why Automate Quality

Human reviewers are essential for evaluating technical accuracy, logical flow, and pedagogical clarity. They are unreliable for enforcing mechanical consistency — the 200th instance of "Javascript" (should be "JavaScript") in a large docs site will be missed by even the most diligent reviewer.

Automated linting catches:
- **Terminology violations:** Using "route" instead of "endpoint," or "web app" instead of "web application."
- **Brand inconsistencies:** Inconsistent capitalization of product names.
- **Style drift:** Passive voice, hedging phrases, sentences that exceed length limits.
- **Inclusivity issues:** Language that inadvertently excludes or alienates readers.

By catching these mechanically, human reviewers can focus on what automation cannot: Is this explanation actually understandable? Is this example realistic? Does this guide solve the reader's real problem?

---

## Vale Configuration

### Installation

```bash
# macOS
brew install vale

# Linux
snap install vale

# Verify
vale --version
```

### The .vale.ini File

The `.vale.ini` file at the project root configures Vale's behavior:

```ini
# Where to find style rules
StylesPath = .vale/styles

# Minimum severity to report
MinAlertLevel = suggestion

# Style packages to download
Packages = Google, write-good

# Vocabularies for domain-specific terms
Vocab = ProjectTerms

# Apply these styles to all Markdown files
[*.md]
BasedOnStyles = Vale, Google, ProjectCustom

# Apply a stricter set to API documentation
[docs/api/*.md]
BasedOnStyles = Vale, Google, ProjectCustom, APIStrict
```

### Directory Structure

```
project-root/
├── .vale.ini
└── .vale/
    └── styles/
        ├── Google/              # Downloaded community style
        ├── write-good/          # Downloaded community style
        ├── ProjectCustom/       # Custom rules for this project
        │   ├── Terminology.yml
        │   ├── BrandNames.yml
        │   ├── Clarity.yml
        │   ├── SentenceLength.yml
        │   └── Inclusive.yml
        └── config/
            └── vocabularies/
                └── ProjectTerms/
                    ├── accept.txt   # Words to accept (no false positives)
                    └── reject.txt   # Words to always reject
```

### Running Vale

```bash
# Lint all documentation
vale docs/

# Lint a specific file
vale docs/getting-started.md

# Lint with JSON output (for CI parsing)
vale --output=JSON docs/

# Sync downloaded style packages
vale sync
```

---

## Style Packages

Use well-maintained community style packages as a baseline. Layer custom rules on top for project-specific standards.

| Package | Focus | Use Case |
|---------|-------|----------|
| **Google** | Google developer documentation style guide | Comprehensive style enforcement for developer docs |
| **Microsoft** | Microsoft Writing Style Guide | Alternative comprehensive baseline |
| **write-good** | General prose quality (passive voice, weasel words, adverbs) | Catch hedging and fluff |
| **alex** | Inclusive and sensitive language | Catch insensitive or exclusionary language |
| **Readability** | Grade-level readability scores | Ensure content is accessible to the target audience |

### Configuration

```ini
# .vale.ini
Packages = Google, write-good, alex

[*.md]
BasedOnStyles = Vale, Google, write-good, alex
# Disable specific rules that conflict with project style
Google.Passive = suggestion    # Downgrade from error to suggestion
write-good.TooWordy = warning
```

---

## Custom Rule Authoring

Custom rules are YAML files placed in the project's style directory. Each rule defines a pattern to match, the severity level, and the message to display.

### Rule Types

| Type | Purpose | Use |
|------|---------|-----|
| `existence` | Flag text that should not appear | Banned terms, profanity, deprecated terminology |
| `substitution` | Replace one term with another | Preferred terminology |
| `occurrence` | Limit how many times a pattern can appear | Maximum heading levels per section |
| `conditional` | Require a pattern when another is present | Acronym expansion on first use |
| `consistency` | Enforce consistent usage of alternatives | "e-mail" vs. "email" |
| `readability` | Score reading difficulty | Flesch-Kincaid or similar |

### Example: Custom Substitution Rule

```yaml
# .vale/styles/ProjectCustom/Terminology.yml
extends: substitution
message: "Use '%s' instead of '%s'."
level: error
ignorecase: true
swap:
  route: endpoint
  repo: repository
  config: configuration
  env var: environment variable
  auth: authentication
  infra: infrastructure
```

### Example: Custom Existence Rule

```yaml
# .vale/styles/ProjectCustom/NoHedging.yml
extends: existence
message: "Remove the hedging phrase '%s'. State facts directly."
level: warning
ignorecase: true
tokens:
  - it should be noted
  - please note that
  - it is important to
  - you might want to
  - it is generally recommended
  - in most cases
  - basically
  - could potentially
```

### Example: Sentence Length Rule

```yaml
# .vale/styles/ProjectCustom/SentenceLength.yml
extends: occurrence
message: "Sentence exceeds 35 words (%d). Split into shorter sentences."
level: warning
scope: sentence
max: 35
token: '\b\w+\b'
```

---

## Terminology Enforcement

Maintain a project vocabulary that standardizes terminology across all documentation.

### Vocabulary Files

```
# .vale/styles/config/vocabularies/ProjectTerms/accept.txt
# Terms that are correct and should not trigger alerts
Firestore
BigQuery
Testcontainers
OpenTelemetry
gRPC
OAuth
PKCE
CORS
WebSocket
```

```
# .vale/styles/config/vocabularies/ProjectTerms/reject.txt
# Terms that should never appear
Mongo  # Use "MongoDB"
k8s    # Use "Kubernetes"
JS     # Use "JavaScript"
```

### Terminology Table

Maintain a terminology table in the project's style guide for human reference. This table mirrors the automated rules:

| ❌ Incorrect | ✅ Correct | Rationale |
|-------------|-----------|-----------|
| route | endpoint | Standardized API terminology |
| repo | repository | Avoid informal abbreviation |
| web app | web application | Formal documentation standard |
| Javascript | JavaScript | Correct brand capitalization |
| Mongodb | MongoDB | Correct brand capitalization |

---

## Brand Consistency

Brand names have exact capitalization and spelling. Automated enforcement prevents the gradual erosion of brand consistency across a large documentation set.

### Brand Name Rule

```yaml
# .vale/styles/ProjectCustom/BrandNames.yml
extends: substitution
message: "Use the correct brand name: '%s' instead of '%s'."
level: error
ignorecase: false
swap:
  javascript: JavaScript
  typescript: TypeScript
  Typescript: TypeScript
  nodejs: Node.js
  node.js: Node.js
  Github: GitHub
  github: GitHub
  Mongodb: MongoDB
  mongodb: MongoDB
  Postgresql: PostgreSQL
  postgresql: PostgreSQL
  Openapi: OpenAPI
  openapi: OpenAPI
  Graphql: GraphQL
  graphql: GraphQL
  Kubernetes: Kubernetes
  kubernetes: Kubernetes
```

---

## Clarity Rules

Clarity rules enforce the writing style principles defined in the style guide.

### Passive Voice Detection

```yaml
# .vale/styles/ProjectCustom/ActiveVoice.yml
extends: existence
message: "Use active voice. '%s' uses passive construction."
level: warning
ignorecase: true
tokens:
  - is configured
  - is returned
  - is created
  - was deployed
  - are stored
  - were processed
  - be handled
  - been validated
```

### Filler Word Detection

```yaml
# .vale/styles/ProjectCustom/NoFiller.yml
extends: existence
message: "Remove the filler phrase '%s'."
level: suggestion
ignorecase: true
tokens:
  - in order to
  - as a matter of fact
  - at the end of the day
  - for the purpose of
  - at this point in time
  - due to the fact that
  - in the event that
  - on the other hand
```

---

## Inclusive Language

Audit documentation for language that may inadvertently exclude or alienate readers. This is both an ethical imperative and a practical one — documentation read by a global audience must be clear and welcoming to all.

### Inclusive Language Rule

```yaml
# .vale/styles/ProjectCustom/Inclusive.yml
extends: substitution
message: "Use inclusive language: '%s' instead of '%s'."
level: warning
ignorecase: true
swap:
  blacklist: blocklist
  whitelist: allowlist
  master: primary
  slave: replica
  sanity check: validation check
  dummy: placeholder
  guys: everyone / team
  manpower: workforce
  man-hours: person-hours
```

---

## Git Hook Integration

Configure a pre-commit Git hook to run Vale on changed Markdown files. This catches violations before code leaves the developer's machine.

### Pre-commit Configuration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/errata-ai/vale
    hooks:
      - id: vale
        name: vale-lint
        entry: vale
        language: system
        types: [markdown]
        args: ["--minAlertLevel=error"]
```

### Husky Configuration (Node.js projects)

```json
// package.json
{
  "lint-staged": {
    "**/*.md": ["vale --minAlertLevel=error"]
  },
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  }
}
```

---

## CI/CD Pipeline Integration

Integrate Vale into the CI pipeline so that documentation violations block PRs in the same way that linting errors block code changes.

### GitHub Actions

```yaml
name: Documentation Quality
on:
  pull_request:
    paths:
      - "docs/**/*.md"
      - "README.md"
      - ".vale/**"

jobs:
  vale-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: errata-ai/vale-action@v2
        with:
          files: docs/
          vale_flags: "--minAlertLevel=error"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### GitLab CI

```yaml
vale-lint:
  image: jdkato/vale:latest
  stage: lint
  script:
    - vale sync
    - vale --minAlertLevel=error docs/
  rules:
    - changes:
        - "docs/**/*.md"
        - ".vale/**"
```

---

## Managing False Positives

### Inline Suppression

Suppress specific rules for sections that intentionally violate style guidelines (e.g., quoting external content, showing anti-examples):

```markdown
<!-- vale Google.Passive = NO -->
This sentence uses passive voice intentionally to demonstrate the pattern.
<!-- vale Google.Passive = YES -->
```

### Vocabulary Exceptions

Add domain-specific terms to the vocabulary accept list rather than suppressing the spell-check rule globally. This preserves the rule's value for genuine misspellings while allowing legitimate technical terms.

### Rule Severity Calibration

Not every rule deserves `error` severity. Calibrate based on impact:

| Severity | Use When | CI Behavior |
|----------|----------|-------------|
| `error` | The violation is objectively wrong (brand name misspelling, banned term) | Block the PR |
| `warning` | The violation degrades quality but is debatable (passive voice, sentence length) | Report but do not block |
| `suggestion` | The violation is a style preference | Report in IDE only, not in CI |

---

## Documentation Testing

Prose linting (Vale) catches style issues. Documentation testing catches structural and functional failures — broken links, outdated code snippets, and dead references.

### Link Checking

Broken links erode trust. Automate link validation in CI:

```yaml
# GitHub Actions: link checking
- name: Check links
  uses: lycheeverse/lychee-action@v2
  with:
    args: >-
      --verbose
      --no-progress
      --accept 200,204
      --exclude-loopback
      docs/ README.md CONTRIBUTING.md
    fail: true
```

**Tools:**

| Tool | Language | Best For |
|------|----------|----------|
| **lychee** | Rust | Fast, CI-native, supports Markdown and HTML |
| **markdown-link-check** | Node.js | Simple, Markdown-only, good for pre-commit hooks |
| **htmltest** | Go | Static site output validation (Hugo, Docusaurus builds) |

### Code Snippet Validation

Code examples in documentation rot as the codebase evolves. Prevent this by testing snippets automatically:

**Strategy 1 — Extract and execute:**
Use tools like `codedown` to extract code blocks from Markdown, then run them as tests:

```bash
# Extract Go code blocks from docs and compile them
codedown go < docs/quickstart.md | go run -
```

**Strategy 2 — Unit tests as documentation source:**
Write documentation examples _inside_ the test suite, then include them in docs via snippet references:

```markdown
<!-- docs/quickstart.md -->
[example](../internal/examples/quickstart_test.go#L10-L25)
```

This inverts the problem: instead of testing docs, the docs reference tested code.

**Strategy 3 — Dry-run validation:**
For CLI-heavy documentation, validate commands execute without error:

```bash
# Validate all bash snippets in docs (dry-run where possible)
grep -h '```bash' -A 20 docs/*.md | grep '^\$' | sed 's/^\$ //' | bash -n
```

### Drift Detection

Configure CI to detect when source code changes diverge from documentation:

```yaml
# Flag PRs that change API endpoints but not docs
- name: Docs drift check
  run: |
    CHANGED_API=$(git diff --name-only origin/main | grep 'internal/entrypoints/')
    CHANGED_DOCS=$(git diff --name-only origin/main | grep 'docs/')
    if [ -n "$CHANGED_API" ] && [ -z "$CHANGED_DOCS" ]; then
      echo "::warning::API files changed but no documentation updated"
    fi
```

---

## Accessibility

Documentation that is inaccessible to users with disabilities is incomplete documentation. Accessibility is not a separate concern — it is a quality dimension that applies to every page, diagram, and interaction.

### Alt Text for Images and Diagrams

Every image and diagram includes descriptive alt text that conveys the same information as the visual:

```markdown
<!-- Good — describes the information, not the format -->
![System architecture showing the API gateway routing requests to three
backend services: auth, meetings, and ML pipeline](diagrams/architecture.png)

<!-- Bad — describes the image format, not the content -->
![Architecture diagram](diagrams/architecture.png)

<!-- Bad — empty alt text for informational content -->
![](diagrams/architecture.png)
```

**Rules:**
- **Informational images** (diagrams, screenshots) get descriptive alt text that conveys the key information
- **Decorative images** (borders, spacers) get empty alt text: `alt=""`
- **Complex diagrams** get a brief alt text plus a text description below the image or in a `<details>` block

### Color Contrast in Diagrams

Diagrams must be legible to users with color vision deficiencies:

- **Never use color alone** to convey meaning. Pair color with labels, patterns, or shapes
- **Minimum contrast ratio:** 4.5:1 for text on backgrounds (WCAG AA)
- **Test with simulation tools:** Use browser dev tools (Chrome: Rendering → Emulate vision deficiency) to verify diagrams under protanopia, deuteranopia, and tritanopia

### Table Accessibility

Markdown tables render with basic HTML. For complex tables with merged cells or multi-level headers, use explicit HTML with accessibility attributes:

```html
<table>
  <caption>API Error Codes</caption>
  <thead>
    <tr>
      <th scope="col">Code</th>
      <th scope="col">Message</th>
      <th scope="col">Resolution</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>AUTH_EXPIRED</td>
      <td>Token has expired</td>
      <td>Refresh the token</td>
    </tr>
  </tbody>
</table>
```

### Documentation Site Structure

When building documentation sites, ensure:

- **Semantic HTML landmarks:** Use `<nav>`, `<main>`, `<aside>`, `<footer>` — screen readers use these to navigate the page
- **Heading hierarchy:** A single `<h1>` per page, followed by `<h2>`, `<h3>` in order — never skip heading levels
- **Keyboard navigation:** All interactive elements (search, tabs, accordions, code copy buttons) are reachable via keyboard
- **Skip-to-content link:** The first focusable element on every page is a "Skip to main content" link
- **Focus indicators:** Interactive elements have visible focus outlines (never `outline: none` without a replacement)

---

## Templates

### Minimal .vale.ini

```ini
StylesPath = .vale/styles
MinAlertLevel = suggestion
Packages = Google

Vocab = ProjectTerms

[*.md]
BasedOnStyles = Vale, Google
```

### Full .vale.ini with Custom Rules

```ini
StylesPath = .vale/styles
MinAlertLevel = suggestion
Packages = Google, write-good, alex

Vocab = ProjectTerms

[*.md]
BasedOnStyles = Vale, Google, write-good, alex, ProjectCustom
Google.Passive = warning
write-good.TooWordy = suggestion

[docs/api/*.md]
BasedOnStyles = Vale, Google, ProjectCustom, APIStrict
APIStrict.OperationDescription = error

[CHANGELOG.md]
BasedOnStyles = Vale
```
