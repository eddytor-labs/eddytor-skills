# Eddytor setup for Claude

## Claude Desktop

Add to your Claude Desktop MCP configuration (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "eddytor": {
      "url": "https://mcp.eddytor.com/mcp"
    }
  }
}
```

Restart Claude Desktop. Authentication is handled automatically via OAuth 2.1 — you'll be prompted to sign in on first use.

## claude.ai

MCP servers can be added in the claude.ai settings under the Integrations section. Add the Eddytor MCP endpoint URL: `https://mcp.eddytor.com/mcp`

## Claude Code

Claude Code supports MCP servers natively. Add the server to your project configuration:

```bash
claude mcp add eddytor --transport streamable-http https://mcp.eddytor.com/mcp
```

## Verifying the connection

After setup, ask Claude to run `list_tables`. If it returns your workspace tables, the connection is working.

## Using skills with Claude

Claude can read skill files directly. Reference them in your conversation:

> "Follow the getting-started skill in `skills/getting-started/SKILL.md` to help me set up my first table."

Or include skill content in your project knowledge (Claude Desktop) or system prompt for automatic context.
