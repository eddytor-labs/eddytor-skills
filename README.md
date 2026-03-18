# Eddytor Skills

A library of LLM skills and MCP tool guidance for working with the [Eddytor](https://eddytor.com) master data management platform.

**Provider-agnostic** — skills are plain markdown that works with Claude, OpenAI, Gemini, Cursor, and any other LLM.

## What are skills?

Skills follow the open [Agent Skills](https://agentskills.io) format — a standard supported by Claude Code, Cursor, VS Code, Gemini CLI, and 30+ other agent products. Each skill is a folder containing a `SKILL.md` file with YAML frontmatter and markdown instructions that agents discover and activate automatically.

## Repository structure

```
eddytor-skills/
├── skills/                                    # Individual skill definitions (Agent Skills format)
│   ├── eddytor-getting-started/SKILL.md       # Onboarding: connect, auth, first table
│   ├── eddytor-table-management/SKILL.md      # Create, schema design, column types
│   ├── eddytor-data-import/SKILL.md           # CSV import, schema inference
│   ├── eddytor-data-quality/SKILL.md          # Constraints, validation, profiling
│   ├── eddytor-querying/SKILL.md              # query_rows, execute_sql, aggregation
│   ├── eddytor-domain-hierarchies/SKILL.md    # Fixed, hierarchical, reference domains
│   ├── eddytor-version-control/SKILL.md       # History, diff, rollback, restore
│   ├── eddytor-bulk-operations/SKILL.md       # Merge, batch insert/update/delete
│   ├── eddytor-migration/SKILL.md             # From Excel, MDS, SQL databases
│   └── eddytor-optimization/SKILL.md          # Compact, vacuum, performance
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

Each skill follows the [Agent Skills specification](https://agentskills.io/specification) — a folder with a `SKILL.md` file containing `---` YAML frontmatter. You can:

1. **Reference directly** — point your LLM at a specific skill file when working on that topic
2. **Include in system prompts** — paste skill content into your LLM's context for domain expertise
3. **Use with MCP** — skills teach the LLM which MCP tools to call and in what order
4. **Install as Claude Code commands** — see below

Skills include a YAML frontmatter block with trigger conditions — use these to build automatic skill selection in your toolchain.

## Claude Code slash commands

This repo includes pre-built slash commands in `.claude/commands/`. After cloning, you get commands like `/eddytor-getting-started`, `/eddytor-querying`, etc.

**Install into your project:**

```bash
# Clone the repo
git clone https://github.com/eddytor-labs/eddytor-skills.git

# Symlink commands into your project
mkdir -p .claude/commands
for f in eddytor-skills/.claude/commands/*.md; do
  ln -s "$(pwd)/$f" .claude/commands/
done
```

**Or install globally** (available in all projects):

```bash
mkdir -p ~/.claude/commands
for f in eddytor-skills/.claude/commands/*.md; do
  ln -s "$(pwd)/$f" ~/.claude/commands/
done
```

Available commands:

| Command | Description |
| --- | --- |
| `/eddytor-getting-started` | Onboarding, MCP setup, first table |
| `/eddytor-table-management` | Create tables, schema design, column types |
| `/eddytor-data-import` | CSV import, schema inference |
| `/eddytor-data-quality` | Constraints, validation, profiling |
| `/eddytor-querying` | query_rows, aggregate, execute_sql |
| `/eddytor-domain-hierarchies` | Fixed, hierarchical, reference domains |
| `/eddytor-version-control` | History, diff, rollback, restore |
| `/eddytor-bulk-operations` | Merge, batch insert/update/delete |
| `/eddytor-migration` | From Excel, MDS, SQL databases |
| `/eddytor-optimization` | Compact, vacuum, performance |

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

This project is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) — free to use, share, and adapt with attribution, but not for commercial purposes. See [LICENSE](LICENSE) for details.
