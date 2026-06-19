# Eddytor Skills

A library of LLM skills and MCP tool guidance for working with the [Eddytor](https://eddytor.com) master data management platform.

**Provider-agnostic** — skills are plain markdown that works with Claude, OpenAI, Gemini, Cursor, and any other LLM.

## What are skills?

Skills follow the open [Agent Skills](https://agentskills.io) format — a standard supported by Claude Code, Cursor, VS Code, Gemini CLI, and 30+ other agent products. Each skill is a folder containing a `SKILL.md` file with YAML frontmatter and markdown instructions that agents discover and activate automatically.

## Repository structure

```
eddytor-skills/
├── skills/                                 # 29 skills (Agent Skills format), grouped by area:
│   │ ── Data & MDM ──
│   ├── eddytor-getting-started/            # Onboarding: connect, auth, first table
│   ├── eddytor-zero-to-table/              # End-to-end: storage → table → domains → validate
│   ├── eddytor-table-management/           # Create, schema design, column types
│   ├── eddytor-table-lifecycle/            # Rename, move, drop, share tables
│   ├── eddytor-data-import/                # CSV import, schema inference
│   ├── eddytor-data-quality/               # Constraints, validation, profiling
│   ├── eddytor-domain-hierarchies/         # Fixed, hierarchical, reference domains
│   ├── eddytor-querying/                   # query_rows, execute_sql, aggregation
│   ├── eddytor-bulk-operations/            # Merge, batch insert/update/delete
│   ├── eddytor-version-control/            # History, diff, rollback, restore
│   ├── eddytor-optimization/               # Compact, vacuum, performance
│   ├── eddytor-migration/                  # From Excel, MDS, SQL databases
│   ├── eddytor-storage-registration/       # Connect S3 / Azure / GCS storage
│   ├── eddytor-object-store/               # List, upload, download, move objects
│   │ ── Deployment ──
│   ├── eddytor-deploy-helm/                # Any Kubernetes via the OCI Helm chart
│   ├── eddytor-deploy-aks/                 # Azure AKS
│   ├── eddytor-deploy-gke/                 # Google GKE
│   ├── eddytor-deploy-eks/                 # AWS EKS
│   ├── eddytor-deploy-hetzner/             # Hetzner k3s (budget)
│   ├── eddytor-deploy-compose/             # Single-host Docker Compose
│   ├── eddytor-deploy-object-store/        # Choose + wire the backing object store
│   │ ── Admin & ops ──
│   ├── eddytor-access-control/             # Roles, API keys, OAuth clients, audit
│   ├── eddytor-provider-oauth/             # Per-org Azure/Google delegated-storage apps
│   ├── eddytor-observability/              # OTLP traces + metrics, log levels
│   ├── eddytor-backup-key-rotation/        # Backups + secret / key rotation
│   │ ── Interfaces ──
│   ├── eddytor-cli/                        # eddytor CLI best practices
│   ├── eddytor-flight-sql/                 # Bulk reads over Arrow Flight SQL
│   ├── eddytor-rest-api/                   # REST: auth, OpenAPI, errors, rate limits
│   └── eddytor-python-sdk/                 # eddytor-sdk: query → pandas / Arrow
│   (each is a folder with a SKILL.md)
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

### Step 1: Connect to your Eddytor MCP endpoint

This gives your agent access to Eddytor's 45 MCP tools. Eddytor serves MCP at
**`/mcp` on your own server**, so use the URL where your Eddytor is reachable:

- **Self-hosted:** `https://<your-eddytor-host>/mcp` (your server's public URL — e.g.
  `https://eddytor.example.com/mcp`, or `http://localhost:8080/mcp` when port-forwarding).
- **Eddytor Cloud:** `https://mcp.eddytor.com/mcp`.

Substitute that URL below (`https://<your-eddytor-host>/mcp`):

**Claude Desktop / claude.ai** — add to your MCP config:

```json
{ "mcpServers": { "eddytor": { "url": "https://<your-eddytor-host>/mcp" } } }
```

**Cursor** — add to `.cursor/mcp.json`:

```json
{ "mcpServers": { "eddytor": { "url": "https://<your-eddytor-host>/mcp" } } }
```

**Claude Code:**

```bash
claude mcp add eddytor --transport streamable-http https://<your-eddytor-host>/mcp
```

The endpoint authenticates against your server's own OAuth (it's the IdP), so your MCP
client runs the normal browser sign-in on first connect. See
[guides/provider-setup/](guides/provider-setup/) for other providers.

### Step 2: Add skills to your project

Skills are auto-discovered by any [Agent Skills-compatible tool](https://agentskills.io) — drop the skill folders into your project and agents find them automatically.

**Option A: `eddytor install skills`** (recommended — one command, no git)

The [`eddytor` CLI](https://eddytor.com/docs/cli) installs skills straight from this repo
into `./skills/` (each as `<skill-id>/SKILL.md`, plus the guides). No clone, no auth — it
just downloads the published markdown:

```bash
eddytor get skills                              # list all available skills
eddytor install skills                          # install all of them into ./skills/
eddytor install skills --skill eddytor-querying # or just one
eddytor install skills --path .agent/skills     # custom target directory
```

Re-run `eddytor install skills` to pull the latest versions.

**Option B: Git submodule** (stays in sync via `git submodule update --remote`)

```bash
git submodule add https://github.com/eddytor-labs/eddytor-skills.git eddytor-skills
```

**Option C: Clone or copy** (grab everything, or just the skills you need)

```bash
git clone https://github.com/eddytor-labs/eddytor-skills.git
# or copy individual skills:
cp -r eddytor-skills/skills/eddytor-querying skills/
```

### How auto-discovery works

At startup, your agent scans the project for `SKILL.md` files and reads only the `name` and `description` from each. When a task matches a skill's description, the agent loads the full instructions. No manual invocation needed — the agent picks the right skill automatically.

This works across 30+ tools including Claude Code, Cursor, VS Code / Copilot, Gemini CLI, Goose, OpenHands, and more. See [agentskills.io](https://agentskills.io) for the full list.

## MCP tools overview

Eddytor exposes 45 MCP tools across these categories:

| Category | Tools |
| --- | --- |
| Discovery | `list_tables`, `resolve_table`, `get_table_schema`, `get_table_metadata`, `profile_table`, `get_share_link` |
| Creation & lifecycle | `create_table`, `add_column`, `update_column_settings`, `rename_table`, `move_table`, `drop_table` |
| Reading | `query_rows`, `aggregate`, `execute_sql` |
| Writing | `insert_rows`, `merge_rows`, `delete_rows` |
| Import | `infer_schema`, `import_csv` |
| Domains | `set_column_domain`, `get_allowed_values`, `delete_column_domain` |
| Validation | `validate_constraints`, `validate_domain_values` |
| History | `get_table_history`, `diff_table_versions`, `rollback_table`, `restore_table`, `get_audit_log` |
| Maintenance | `optimize_table`, `vacuum_table` |
| Storage configs | `register_s3_storage`, `register_az_storage`, `register_gcs_storage`, `list_storage_configs`, `update_storage_config`, `delete_storage_config` |
| Object store | `list_objects`, `download_object`, `upload_object`, `delete_object`, `create_folder`, `move_objects`, `create_demo_table` |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding or modifying skills.

## License

This project is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) — free to use, share, and adapt with attribution, but not for commercial purposes. See [LICENSE](LICENSE) for details.
