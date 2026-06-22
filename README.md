# Eddytor Skills

LLM skills and MCP tool guidance for the [Eddytor](https://eddytor.com) master data management platform — plain-markdown [Agent Skills](https://agentskills.io) that work with Claude Code, Cursor, Codex, Gemini CLI, and any other Agent Skills-compatible tool.

## Quick start

```bash
# 1. Point your agent at your Eddytor server's MCP endpoint (the tools).
#    Self-hosted: https://<your-eddytor-host>/mcp   ·   Eddytor Cloud: https://mcp.eddytor.com/mcp
claude mcp add eddytor --transport streamable-http https://<your-eddytor-host>/mcp

# 2. Install the skills (the guidance) into whatever agent you use.
npx skills add eddytor-labs/eddytor-skills
```

That's it — your agent now has Eddytor's 45 MCP tools plus 29 auto-discovered skills. The
rest of this page expands on each step, lists the skills, and documents the tools.

## Installation

### Step 1: Connect to your Eddytor MCP endpoint

This gives your agent access to Eddytor's 45 MCP tools. Eddytor serves MCP at
**`/mcp` on your own server**, so use the URL where your Eddytor is reachable:

- **Self-hosted:** `http[s]://[<your_domain>|localhost]/mcp` (your server's public URL — e.g.
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

Pick whichever fits your setup — all four leave you with auto-discoverable `SKILL.md` files.

**Option A: `npx skills add`** (recommended — one command, any agent)

The open [`skills` CLI](https://github.com/vercel-labs/skills) (GitHub is the registry)
detects which agents you have — Claude Code, Cursor, Codex, Gemini CLI, Goose, OpenCode,
and 60+ more — and installs the `SKILL.md`s into each one's skills directory for you:

```bash
npx skills add eddytor-labs/eddytor-skills                      # interactive: pick skills, agents, scope
npx skills add eddytor-labs/eddytor-skills --skill eddytor-querying   # just one
npx skills add eddytor-labs/eddytor-skills -g -y                # all skills, global, non-interactive
```

It copies (or symlinks) into the detected agent dir (e.g. `.claude/skills/`) and writes a
`skills-lock.json`. Note: it installs the skills only — for the `guides/` use Option B.

**Option B: `eddytor install skills`** (no Node; first-party; pulls the guides too)

The [`eddytor` CLI](https://eddytor.com/docs/cli) downloads straight from this repo into
`./skills/` (each as `<skill-id>/SKILL.md`, plus `guides/`). No clone, no auth:

```bash
eddytor get skills                              # list all available skills
eddytor install skills                          # install all of them into ./skills/
eddytor install skills --skill eddytor-querying # or just one
eddytor install skills --path .agent/skills     # custom target directory
```

**Option C: Git submodule** (stays in sync via `git submodule update --remote`)

```bash
git submodule add https://github.com/eddytor-labs/eddytor-skills.git eddytor-skills
```

**Option D: Clone or copy** (grab everything, or just the skills you need)

```bash
git clone https://github.com/eddytor-labs/eddytor-skills.git
# or copy individual skills:
cp -r eddytor-skills/skills/eddytor-querying skills/
```

## How skills work

Each skill is a folder with a `SKILL.md` — YAML frontmatter (`name` + `description`)
followed by markdown instructions. At startup your agent scans the project for `SKILL.md`
files, reads only the `name` and `description` from each, and loads the full instructions
when a task matches the description. No manual invocation — the agent picks the right skill
automatically. This works across 30+ tools including Claude Code, Cursor, VS Code / Copilot,
Gemini CLI, Goose, and OpenHands. See [agentskills.io](https://agentskills.io) for the list.

**Chat assistants (ChatGPT, Gemini app, Mistral Le Chat, claude.ai):** these don't scan a
project folder, so the installers above don't apply. Connect the MCP endpoint (Step 1) for
the tools, and supply a skill as context the product's own way — a Custom GPT's
instructions, a Gemini Gem, a Project's files, claude.ai's Skills, or the system prompt.

## Skills

29 skills, grouped by area. Browse them under [`skills/`](skills/); install with Step 2.

**Data & MDM**

- `eddytor-getting-started` — onboarding: connect, auth, first table
- `eddytor-zero-to-table` — end-to-end: storage → table → domains → validate
- `eddytor-table-management` — create tables, schema design, column types
- `eddytor-table-lifecycle` — rename, move, drop, share tables
- `eddytor-data-import` — CSV import, schema inference
- `eddytor-data-quality` — constraints, validation, profiling
- `eddytor-domain-hierarchies` — fixed, hierarchical, reference domains
- `eddytor-querying` — `query_rows`, `execute_sql`, aggregation
- `eddytor-bulk-operations` — merge, batch insert/update/delete
- `eddytor-version-control` — history, diff, rollback, restore
- `eddytor-optimization` — compact, vacuum, performance
- `eddytor-migration` — from Excel, MDS, SQL databases
- `eddytor-storage-registration` — connect S3 / Azure / GCS storage
- `eddytor-object-store` — list, upload, download, move objects

**Deployment**

- `eddytor-deploy-helm` — any Kubernetes via the OCI Helm chart
- `eddytor-deploy-aks` — Azure AKS
- `eddytor-deploy-gke` — Google GKE
- `eddytor-deploy-eks` — AWS EKS
- `eddytor-deploy-hetzner` — Hetzner k3s (budget)
- `eddytor-deploy-compose` — single-host Docker Compose
- `eddytor-deploy-object-store` — choose + wire the backing object store

**Admin & ops**

- `eddytor-access-control` — roles, API keys, OAuth clients, audit
- `eddytor-provider-oauth` — per-org Azure/Google delegated-storage apps
- `eddytor-observability` — OTLP traces + metrics, log levels
- `eddytor-backup-key-rotation` — backups + secret / key rotation

**Interfaces**

- `eddytor-cli` — `eddytor` CLI best practices
- `eddytor-flight-sql` — bulk reads over Arrow Flight SQL
- `eddytor-rest-api` — REST: auth, OpenAPI, errors, rate limits
- `eddytor-python-sdk` — `eddytor-sdk`: query → pandas / Arrow

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

Full parameter details: [guides/mcp-tool-reference.md](guides/mcp-tool-reference.md).

## Repository layout

```
eddytor-skills/
├── skills/      # one folder per skill, each a SKILL.md (Agent Skills format)
├── guides/      # cross-cutting references: mcp-tool-reference.md + provider-setup/
├── examples/    # worked end-to-end workflows
└── schemas/     # skill-manifest.json — machine-readable registry of all skills
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding or modifying skills.

## License

This project is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) — free to use, share, and adapt with attribution, but not for commercial purposes. See [LICENSE](LICENSE) for details.
