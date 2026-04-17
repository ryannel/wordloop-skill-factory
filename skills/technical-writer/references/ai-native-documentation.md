# AI-Native Documentation

Documentation must serve AI agents as a first-class audience. This means providing structured, machine-readable indexes alongside human-navigable interfaces. The following patterns ensure AI tools can discover, retrieve, and reason about documentation without resorting to brittle HTML scraping.

---

## Table of Contents
1. [The llms.txt Standard](#the-llmstxt-standard)
2. [The llms-full.txt Aggregate](#the-llms-fulltxt-aggregate)
3. [Model Context Protocol (MCP)](#model-context-protocol-mcp)
4. [Semantic Markup for Dual Consumption](#semantic-markup-for-dual-consumption)
5. [HTTP Discovery Headers](#http-discovery-headers)
6. [Markdown Export Convention](#markdown-export-convention)
7. [Templates](#templates)

---

## The llms.txt Standard

`llms.txt` is a plain-text Markdown file hosted at the root of a domain (e.g., `example.com/llms.txt`). It acts as a curated, AI-friendly roadmap — pointing language models to the most authoritative content on a site. Unlike `robots.txt` (which tells crawlers what to avoid) or `sitemap.xml` (which lists everything), `llms.txt` is an editorial index: it prioritizes signal over noise.

### Structure

A standard `llms.txt` follows this format:

```markdown
# Project Name

> A one-paragraph summary of the project, its purpose, and its primary audience.

## Documentation

- [Getting Started](https://docs.example.com/getting-started.md): First-time setup and quickstart guide
- [API Reference](https://docs.example.com/api/): Complete REST API documentation
- [Architecture Overview](https://docs.example.com/architecture.md): System design and component relationships

## Guides

- [Authentication](https://docs.example.com/guides/auth.md): OAuth 2.1 and API key authentication
- [Webhooks](https://docs.example.com/guides/webhooks.md): Event subscription and delivery patterns

## API Specifications

- [OpenAPI Spec](https://docs.example.com/openapi.yaml): Machine-readable API specification
- [AsyncAPI Spec](https://docs.example.com/asyncapi.yaml): Event-driven API specification
```

### Authoring Rules

- **Required elements:** An H1 heading (the project name) and a blockquote summary. These two elements allow an AI agent to immediately understand what the project is and what the documentation covers.
- **Prioritize canonical content:** List the most authoritative, high-signal documents first. The ordering is itself information — it tells the agent which resources are most important.
- **Use descriptive link text:** Every link includes a short description that explains the content at that URL. Bare URLs without descriptions force the agent to fetch and parse the page to understand its purpose.
- **Link to Markdown exports:** Prefer `.md` URLs over HTML when available. Markdown is lighter to parse and produces higher-quality context for language models.
- **Keep it current:** `llms.txt` is a living document. Stale links or missing sections degrade agent confidence. Include `llms.txt` updates in the same PR that adds or removes documentation.

---

## The llms-full.txt Aggregate

`llms-full.txt` is an optional companion file that provides the entire documentation set in a single, concatenated Markdown file. It exists for AI agents that benefit from having the full corpus in a single context window rather than fetching individual pages.

### Structure

```markdown
# Project Name — Full Documentation

> A comprehensive, single-file reference for the entire documentation set.
> Generated automatically from the source documentation. Do not edit manually.

---

## Getting Started

[Full content of the Getting Started page]

---

## API Reference

[Full content of the API Reference page]

---

[...continues for all documented pages]
```

### Generation

Generate `llms-full.txt` automatically in the documentation build pipeline. Concatenate all source Markdown files in the order defined by `llms.txt`, separated by horizontal rules. Include a generation timestamp in the file header so agents can determine freshness.

```bash
# Example: Build script that generates llms-full.txt
#!/bin/bash
echo "# Project Name — Full Documentation" > llms-full.txt
echo "" >> llms-full.txt
echo "> Generated: $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> llms-full.txt
echo "" >> llms-full.txt

for file in $(cat llms.txt | grep -oP '\(.*?\.md\)' | tr -d '()'); do
  echo "---" >> llms-full.txt
  cat "$file" >> llms-full.txt
  echo "" >> llms-full.txt
done
```

### Size Management

Large documentation sets may exceed practical token limits. Apply these strategies:

- **Section-level chunking:** Split `llms-full.txt` into domain-specific aggregates (e.g., `llms-api.txt`, `llms-guides.txt`) when the full file exceeds 100,000 tokens.
- **Compression by reference:** For extremely large reference tables or changelogs, include a summary with a link to the full content rather than inlining the entire text.
- **Priority ordering:** The most important content appears first, so even if an agent truncates the file, the most critical information is preserved.

---

## Model Context Protocol (MCP)

The Model Context Protocol provides a structured, server-based interface for AI tools to retrieve documentation resources programmatically. Unlike `llms.txt` (which is a static file), MCP enables real-time, authenticated access to the current state of documentation.

### Architecture

```
┌─────────────┐     JSON-RPC 2.0     ┌─────────────┐
│  MCP Host    │ ◄──────────────────► │  MCP Server  │
│  (AI IDE)    │   (Streamable HTTP   │  (Docs)      │
│              │    or STDIO)         │              │
└─────────────┘                      └──────┬───────┘
                                            │
                                     ┌──────▼───────┐
                                     │  Documentation│
                                     │  Sources      │
                                     │  (Markdown,   │
                                     │   OpenAPI,    │
                                     │   ADRs)       │
                                     └──────────────┘
```

### Core Primitives

An MCP documentation server exposes three primitive types:

| Primitive | Purpose | Documentation Use |
|-----------|---------|-------------------|
| **Resources** | Read-only data the AI can retrieve | Documentation pages, API specs, ADR logs, diagram sources |
| **Tools** | Executable functions the AI can invoke | Search documentation, validate links, check spec freshness |
| **Prompts** | Reusable templates for structured queries | "Explain this API endpoint", "Summarize this ADR" |

### Resource Design

Structure MCP resources to mirror the documentation hierarchy:

```json
{
  "resources": [
    {
      "uri": "docs://project/getting-started",
      "name": "Getting Started Guide",
      "mimeType": "text/markdown",
      "description": "First-time setup and quickstart for the Project API"
    },
    {
      "uri": "docs://project/api/openapi",
      "name": "OpenAPI Specification",
      "mimeType": "application/yaml",
      "description": "Machine-readable REST API specification (OpenAPI 3.1)"
    },
    {
      "uri": "docs://project/adrs",
      "name": "Architecture Decision Log",
      "mimeType": "text/markdown",
      "description": "Chronological log of all Architecture Decision Records"
    }
  ]
}
```

### Tool Design

Expose search and validation tools so agents can query documentation dynamically:

```json
{
  "tools": [
    {
      "name": "search_docs",
      "description": "Semantic search across all documentation pages",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string", "description": "Natural language search query" },
          "scope": { "type": "string", "enum": ["all", "api", "guides", "adrs"], "description": "Limit search to a documentation section" }
        },
        "required": ["query"]
      }
    },
    {
      "name": "validate_links",
      "description": "Check all documentation links for broken references",
      "inputSchema": {
        "type": "object",
        "properties": {
          "path": { "type": "string", "description": "Documentation path to validate" }
        }
      }
    }
  ]
}
```

### Transport

- **STDIO:** Use for local MCP servers that run as CLI tools alongside the developer's IDE.
- **Streamable HTTP:** Use for remote documentation servers hosted alongside the documentation site.

---

## Semantic Markup for Dual Consumption

The same structural elements that assist screen readers also enable AI crawlers to extract accurate context. Apply these patterns to every documentation page:

### Heading Hierarchy

- **One H1 per page:** The H1 is the page title. It tells both humans and agents what this page is about.
- **Strict hierarchy:** H2 follows H1, H3 follows H2. Never skip levels (e.g., H1 → H3). Skipping breaks both outline generation and agent section extraction.
- **Descriptive headings:** Headings describe content, not structure. Use "Configuring Authentication" instead of "Section 3.2".

### Structural Patterns

- **Code blocks with language identifiers:** Always specify the language (```python, ```yaml, ```bash). Agents use the language identifier to determine how to interpret and apply the code.
- **Tables for structured data:** Use Markdown tables for any data that has rows and columns. Prose descriptions of structured data are harder for agents to parse.
- **Frontmatter metadata:** Include YAML frontmatter in documentation pages with fields like `title`, `description`, `last_updated`, and `audience`. This enables programmatic filtering and sorting.

```yaml
---
title: Authentication Guide
description: Configure OAuth 2.1, API keys, and service account authentication
last_updated: 2025-11-15
audience: [developers, operators]
tags: [auth, security, oauth]
---
```

---

## HTTP Discovery Headers

Enable automated discovery of `llms.txt` by including an HTTP header on every documentation page:

```
Link: </llms.txt>; rel="llms-txt"
```

This allows AI crawlers and tools to discover the index file even when they land on an arbitrary documentation page, without needing to guess the root URL.

### Implementation

Configure the HTTP header in your web server or static site generator:

```nginx
# Nginx
add_header Link '</llms.txt>; rel="llms-txt"' always;
```

```yaml
# Netlify (_headers file)
/*
  Link: </llms.txt>; rel="llms-txt"
```

---

## Markdown Export Convention

Every documentation page hosted as HTML should have a corresponding clean Markdown export available at a predictable URL (e.g., appending `.md` to the page URL or serving from a `/raw/` prefix).

This convention enables AI agents to retrieve the lightweight Markdown source rather than parsing complex HTML with navigation chrome, JavaScript widgets, and styling markup.

**Implementation:** Most static site generators (Docusaurus, MkDocs, Nextra) can be configured to serve the source Markdown alongside the rendered HTML. Configure the build to copy `.md` source files to a predictable output path.

---

## Templates

### Minimal llms.txt

```markdown
# [Project Name]

> [One-paragraph project description: what it does, who it's for, and why.]

## Documentation

- [Getting Started]([URL]): [Description]
- [Core Concepts]([URL]): [Description]
- [API Reference]([URL]): [Description]

## Specifications

- [OpenAPI Spec]([URL]): Machine-readable REST API specification
```

### MCP Server Scaffold (TypeScript)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "project-docs",
  version: "1.0.0",
});

// Expose documentation as a resource
server.resource("getting-started", "docs://project/getting-started", async (uri) => ({
  contents: [{
    uri: uri.href,
    mimeType: "text/markdown",
    text: await readFile("docs/getting-started.md", "utf-8"),
  }],
}));

// Expose search as a tool
server.tool("search_docs", { query: { type: "string" } }, async ({ query }) => {
  const results = await searchDocumentation(query);
  return { content: [{ type: "text", text: JSON.stringify(results) }] };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```
