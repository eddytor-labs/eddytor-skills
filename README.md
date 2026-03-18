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

## Installation

### Step 1: Connect to Eddytor's MCP endpoint

This gives your agent access to Eddytor's 28 MCP tools.

**Claude Desktop / claude.ai** — add to your MCP config:

```json
{ "mcpServers": { "eddytor": { "url": "https://mcp.eddytor.com/mcp" } } }
```

**Cursor** — add to `.cursor/mcp.json`:

```json
{ "mcpServers": { "eddytor": { "url": "https://mcp.eddytor.com/mcp" } } }
```

**Claude Code:**

```bash
claude mcp add eddytor --transport streamable-http https://mcp.eddytor.com/mcp
```

See [guides/provider-setup/](guides/provider-setup/) for other providers.

### Step 2: Add skills to your project

Skills are auto-discovered by any [Agent Skills-compatible tool](https://agentskills.io). Just add the skill folders to your project and agents find them automatically.

**Option A: Git submodule** (recommended — stays in sync with updates)

```bash
git submodule add https://github.com/eddytor-labs/eddytor-skills.git eddytor-skills
```

**Option B: Clone directly**

```bash
git clone https://github.com/eddytor-labs/eddytor-skills.git
```

**Option C: Copy specific skills** (if you only need a few)

```bash
# Example: copy just the querying and data-import skills
cp -r eddytor-skills/skills/eddytor-querying skills/
cp -r eddytor-skills/skills/eddytor-data-import skills/
```

### How auto-discovery works

At startup, your agent scans the project for `SKILL.md` files and reads only the `name` and `description` from each. When a task matches a skill's description, the agent loads the full instructions. No manual invocation needed — the agent picks the right skill automatically.

This works across 30+ tools including Claude Code, Cursor, VS Code / Copilot, Gemini CLI, Goose, OpenHands, and more. See [agentskills.io](https://agentskills.io) for the full list.

## Claude Code slash commands

This repo also includes pre-built slash commands in `.claude/commands/` for manual invocation. After cloning, you get `/eddytor-getting-started`, `/eddytor-querying`, etc.

**Install into your project:**

```bash
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
