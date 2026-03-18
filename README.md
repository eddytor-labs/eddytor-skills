# Eddytor Skills

A library of LLM skills and MCP tool guidance for working with the [Eddytor](https://eddytor.com) master data management platform.

**Provider-agnostic** — skills are plain markdown that works with Claude, OpenAI, Gemini, Cursor, and any other LLM.

## What are skills?

Skills are prompt-based instruction sets that teach any LLM how to use Eddytor effectively via its MCP endpoint. They encode best practices, workflows, and patterns — the knowledge layer that makes LLMs productive with Eddytor.

## Repository structure

```
eddytor-skills/
├── skills/                          # Individual skill definitions
│   ├── getting-started/SKILL.md     # Onboarding: connect, auth, first table
│   ├── table-management/SKILL.md    # Create, schema design, column types
│   ├── data-import/SKILL.md         # CSV import, schema inference
│   ├── data-quality/SKILL.md        # Constraints, validation, profiling
│   ├── querying/SKILL.md            # query_rows, execute_sql, aggregation
│   ├── domain-hierarchies/SKILL.md  # Fixed, hierarchical, reference domains
│   ├── version-control/SKILL.md     # History, diff, rollback, restore
│   ├── bulk-operations/SKILL.md     # Merge, batch insert/update/delete
│   ├── migration/SKILL.md           # From Excel, MDS, SQL databases
│   └── optimization/SKILL.md        # Compact, vacuum, performance
│
├── guides/                          # Cross-cutting references
│   ├── mcp-tool-reference.md        # Complete MCP tool catalog
│   └── provider-setup/              # Per-provider setup instructions
│       ├── claude.md
│       ├── cursor.md
│       └── generic.md
│
├── examples/                        # Real-world workflow examples
│   ├── product-catalog-setup.md
│   └── data-cleanup-workflow.md
│
└── schemas/                         # Machine-readable metadata
    └── skill-manifest.json          # Registry of all skills
```

## Quick start

### Claude Desktop / claude.ai

Add Eddytor as an MCP server in your Claude Desktop config:

```json
{
  "mcpServers": {
    "eddytor": {
      "url": "https://mcp.eddytor.com/mcp"
    }
  }
}
```

### Cursor

Add to `.cursor/mcp.json` in your project:

```json
{
  "mcpServers": {
    "eddytor": {
      "url": "https://mcp.eddytor.com/mcp"
    }
  }
}
```

### Other providers

See [guides/provider-setup/generic.md](guides/provider-setup/generic.md) for setup instructions with any MCP-compatible client.

## How to use skills

Each skill is a standalone markdown file in `skills/<topic>/SKILL.md`. You can:

1. **Reference directly** — point your LLM at a specific skill file when working on that topic
2. **Include in system prompts** — paste skill content into your LLM's context for domain expertise
3. **Use with MCP** — skills teach the LLM which MCP tools to call and in what order

Skills include a YAML frontmatter block with trigger conditions — use these to build automatic skill selection in your toolchain.

## MCP tools overview

Eddytor exposes 28 MCP tools across these categories:

| Category | Tools |
| --- | --- |
| Discovery | `list_tables`, `get_table_schema`, `get_table_metadata`, `profile_table` |
| Creation | `create_table`, `add_column`, `update_column_settings`, `rename_table`, `move_table` |
| Reading | `query_rows`, `aggregate`, `execute_sql` |
| Writing | `insert_rows`, `merge_rows`, `delete_rows` |
| Import | `infer_schema`, `import_csv` |
| Domains | `set_column_domain`, `get_allowed_values`, `delete_column_domain` |
| Validation | `validate_constraints`, `validate_domain_values` |
| History | `get_table_history`, `diff_table_versions`, `rollback_table`, `restore_table`, `get_audit_log` |
| Maintenance | `optimize_table`, `vacuum_table` |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding or modifying skills.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
