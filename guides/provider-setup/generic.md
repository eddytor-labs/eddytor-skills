# Eddytor setup for any MCP-compatible client

## MCP endpoint

```
URL: https://mcp.eddytor.com/mcp
Auth: OAuth 2.1 (automatic for MCP-compatible clients)
```

## For clients with native MCP support

Configure the MCP server URL in your client's settings. The exact location varies by client — look for "MCP servers", "tool servers", or "integrations" in settings.

## For clients without native MCP support (function calling)

If your LLM provider uses function calling (OpenAI, Gemini, Mistral), you need an MCP-to-function-calling adapter:

1. Run an MCP client that connects to `https://mcp.eddytor.com/mcp`
2. Expose the MCP tools as function definitions in your provider's format
3. Route function calls through the MCP client

The `schemas/skill-manifest.json` file provides metadata for all skills. Tool definitions can be introspected from the MCP endpoint itself.

## Using skills without MCP

Skills are plain markdown. Even without MCP tool calling, you can:

1. Copy skill content into your LLM's system prompt or context
2. The LLM will understand Eddytor concepts and can guide you through workflows
3. You execute the tool calls manually via the MCP endpoint or API

## Verifying the connection

Ask your LLM to call `list_tables`. If it returns your workspace tables, the connection is working.
