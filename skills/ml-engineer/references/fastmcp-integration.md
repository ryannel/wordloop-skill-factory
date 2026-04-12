# FastMCP Integrations

In 2026, the Model Context Protocol (MCP) dominates agent interaction bounds. All tools designed to be consumed by autonomous agents natively deploy through FastMCP server instances living logically inside the Entrypoint layer.

## Server Construction
Use the `FastMCP` class to construct standard tooling. Ensure transport occurs strictly over HTTP (SSE/Streamable bindings) instead of raw `stdio` arrays when actively orchestrating scalable backend production fleets.

## Capability Exposing
*   **@mcp.tool():** Expose specific execution routes directly mapping into backend `Service` endpoints. 
    *   **Rule:** You must maintain impeccable Python docstrings and extremely deep Type Hint validation boundaries on these functions. If the typing natively lacks precision, FastMCP fails to auto-generate the complex JSON Schema the LLM requires for tool calling.
*   **@mcp.resource():** Provide the LLM browsing context statically (e.g. referencing pre-processed file storage blocks, vector search endpoints, or configuration maps).
*   **@mcp.prompt():** Define explicitly pre-structured orchestration routing maps dictating precisely how the LLM should conduct interactions securely.

## Elicitation Pattern
If an agent initiates a `mcp.tool()` but misses required input formatting natively, orchestrate the "Elicitation Pattern". Do not terminate the session; intelligently prompt specifically asking the client natively for missing parameters over the active context pipeline seamlessly.
