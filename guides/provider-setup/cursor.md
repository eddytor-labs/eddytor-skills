# Eddytor setup for Cursor

## Configuration

Add to `.cursor/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "eddytor": {
      "url": "https://mcp.eddytor.com/mcp"
    }
  }
}
```

Restart Cursor or reload the window. Authentication is handled automatically via OAuth 2.1.

## Verifying the connection

Open Cursor's AI chat and ask it to call `list_tables`. If it returns your workspace tables, the connection is working.

## Using skills with Cursor

Add skill files to your project's `.cursorrules` or reference them directly:

```
@skills/getting-started/SKILL.md Help me create a new table for product data.
```

Cursor will use the skill content as context for its responses.
