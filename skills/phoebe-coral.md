---
name: Withcoral
description: Use when querying multiple data sources with SQL, setting up MCP connections for agents, writing custom source specs, or managing data access workflows. Agents should reach for Coral when they need to join data across APIs, avoid repeated tool calls, or provide a unified query interface to multiple sources.
metadata:
    mintlify-proj: withcoral
    version: "1.0"
---

# Coral Skill

## Product summary

Coral is a local SQL interface for APIs, files, and other data sources. It translates SQL queries into API calls or file reads, returning results as a single query result set. Agents use Coral to query multiple sources with one SQL statement instead of making separate tool calls to each provider. Coral runs locally on your machine—your data, credentials, and usage history never leave your system.

**Key files and commands:**
- `config.toml`: Stores workspace and source metadata in the platform-specific config directory
- `coral sql "<SQL>"`: Execute read-only SQL queries
- `coral source add <NAME>`: Install a bundled or custom source
- `coral mcp-stdio`: Expose Coral as an MCP server for agents
- Source specs: YAML files that define how to connect to APIs or read local datasets

**Primary docs:** https://withcoral.com/docs

## When to use

Reach for Coral when:

- An agent needs to query multiple data sources in one operation (e.g., "Find GitHub issues assigned to me and compare them with related Slack messages")
- You want to reduce token traffic and tool calls by using SQL instead of sequential API calls
- You're setting up MCP for Claude Code, Cursor, VS Code, or other agents to access data sources
- You need to write a custom source spec to connect an API or dataset that isn't bundled
- You're managing source credentials and want them stored locally with OS keychain support
- You need to inspect available tables, columns, filters, or table functions before writing queries

Do not use Coral for:
- Write operations (Coral is read-only)
- Untrusted agents or scripts (Coral does not sandbox; it trusts the user and their chosen clients)
- Multi-user or remote access (Coral is designed for single-user local use)

## Quick reference

### Essential CLI commands

| Command | Purpose |
| --- | --- |
| `coral source discover` | List bundled sources available in your build |
| `coral source add --interactive <NAME>` | Install a bundled source with interactive prompts |
| `coral source add --file ./spec.yaml` | Import a custom source spec |
| `coral source list` | List installed sources and their secret storage route |
| `coral source info <NAME>` | Show metadata, inputs, and description for a source |
| `coral source lint ./spec.yaml` | Validate a source spec YAML before installing |
| `coral source test <NAME>` | Run post-install validation and test queries |
| `coral source remove <NAME>` | Remove an installed source |
| `coral sql "<SQL>"` | Execute a read-only SQL query |
| `coral mcp-stdio` | Start the MCP stdio server |
| `coral ui` | Open the local Coral UI in your browser |
| `coral workspace list` | List configured workspaces |
| `coral workspace create <NAME>` | Create a new workspace |

### Bundled sources

Coral ships with 25+ bundled sources including: GitHub, GitLab, Slack, Datadog, Linear, Sentry, Jira, Stripe, PagerDuty, Notion, Grafana, Confluence, Google Calendar, and more. Run `coral source discover` to see what's available in your build.

### SQL catalog tables

Query these system tables to inspect your sources:

| Table | Purpose |
| --- | --- |
| `coral.tables` | List all queryable tables and their schemas |
| `coral.columns` | List columns for each table with types and descriptions |
| `coral.filters` | List required and optional filters for tables |
| `coral.table_functions` | List source-scoped table functions and their arguments |
| `coral.inputs` | List configured source variables and secrets (secrets show `NULL` for values) |

Example: `SELECT schema_name, table_name FROM coral.tables ORDER BY 1, 2`

### MCP setup for agents

Use `npx add-mcp` to add Coral to all your agents at once:

```bash
npx add-mcp -n coral -g "$(which coral) mcp-stdio"
```

Or configure manually for specific clients:

**Cursor:** Add to `.cursor/mcp.json` or `~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "coral": {
      "type": "stdio",
      "command": "coral",
      "args": ["mcp-stdio"]
    }
  }
}
```

**VS Code:** Add to `.vscode/mcp.json`:
```json
{
  "servers": {
    "coral": {
      "type": "stdio",
      "command": "coral",
      "args": ["mcp-stdio"]
    }
  }
}
```

**Claude Desktop:** Add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "coral": {
      "command": "coral",
      "args": ["mcp-stdio"]
    }
  }
}
```

### Source spec essentials

Every source spec starts with:
```yaml
name: my_source
version: 0.1.0
dsl_version: 3
backend: http  # or "file" or "mcp"
```

**Column types:** `Utf8`, `Int64`, `Float64`, `Boolean`, `Timestamp`, `Json`

**File-backed tables:** Use `backend: file` with `format: jsonl`, `json`, `csv`, or `parquet`

**HTTP-backed tables:** Declare `base_url`, `auth`, `request`, `response`, `pagination`, and `columns`

**Inputs:** Declare variables and secrets that Coral collects at install time:
```yaml
inputs:
  API_TOKEN:
    kind: secret
    hint: Bearer token for the API
  API_BASE:
    kind: variable
    default: https://api.example.com
```

## Decision guidance

### When to use bundled sources vs custom specs

| Scenario | Use bundled | Use custom |
| --- | --- | --- |
| GitHub, Slack, Datadog, Linear, Jira, Stripe, etc. | ✓ | — |
| API not in bundled or community list | — | ✓ |
| Local files (JSONL, CSV, Parquet, JSON) | — | ✓ |
| Want to extend a bundled source | — | ✓ |

### When to use file vs HTTP backend

| Scenario | File | HTTP |
| --- | --- | --- |
| Local JSONL, CSV, Parquet, JSON files | ✓ | — |
| S3 objects | ✓ | — |
| REST API with pagination | — | ✓ |
| GraphQL or MCP tools | — | ✓ |

### When to use keychain vs file secret storage

| Scenario | Keychain | File |
| --- | --- | --- |
| macOS, Windows, or Linux with Secret Service | ✓ | — |
| Keychain unavailable or disabled | — | ✓ |
| Plaintext acceptable for local dev | — | ✓ |

## Workflow

### Query multiple sources with SQL

1. **Discover available sources:** `coral source discover`
2. **Install sources:** `coral source add --interactive github` (repeat for each source)
3. **Inspect tables:** `coral sql "SELECT schema_name, table_name FROM coral.tables ORDER BY 1, 2"`
4. **Write SQL:** `coral sql "SELECT ... FROM github.issues JOIN linear.issues ON ... WHERE ..."`
5. **Verify results:** Check row count and column names match expectations

### Set up MCP for an agent

1. **Ensure sources are installed:** `coral source list` (should not be empty)
2. **Choose your client:** Cursor, VS Code, Claude Desktop, or other MCP-capable tool
3. **Add Coral to the client:** Use `npx add-mcp` or manual config (see Quick reference)
4. **Verify connection:** Ask the agent to "Use Coral to show what sources and tables you can query"
5. **Test a query:** Ask the agent to run a small Coral query against `coral.tables`

### Write a custom source spec

1. **Validate the API:** Check that it has HTTP credentials, structured docs, and real data to test against
2. **Create the spec:** Start with a minimal YAML file with `name`, `version`, `dsl_version`, `backend`, and one table
3. **Lint before installing:** `coral source lint ./my-source.yaml` (catches YAML and schema errors)
4. **Install the source:** `coral source add --file ./my-source.yaml`
5. **Run strict validation:** `coral source test my_source` (executes test_queries if declared)
6. **Inspect the schema:** Query `coral.tables`, `coral.columns`, `coral.filters` to verify the shape
7. **Refine and repeat:** Update the spec, lint, reinstall, test, and inspect until tables look right

### Manage source credentials

1. **Check storage route:** `coral source list` (shows `keychain` or `file (plaintext)`)
2. **Update credentials:** `coral source add --interactive <NAME>` (replaces stored values)
3. **Inspect inputs:** `coral sql "SELECT schema_name, key, is_set FROM coral.inputs WHERE schema_name = '<NAME>'"`
4. **Remove a source:** `coral source remove <NAME>` (deletes stored credentials)

## Common gotchas

- **Nested field naming:** Use double underscores (`__`) for flattened nested fields. Query `assignee__name`, not `assignee.name`. Inspect `coral.columns` to see the exact names.
- **JSON columns:** Use `json_get_str()`, `json_get_int()`, etc. to query JSON columns. Do not use JSONPath syntax like `$.key`; use plain keys like `json_get_str(properties, 'key')`.
- **Read-only SQL:** Coral blocks DDL, DML, and multiple statements. Only `SELECT` queries work.
- **Required filters:** Some tables require filters in the `WHERE` clause. Query `coral.filters` to see which ones. If a filter is required and missing, the query fails.
- **Pagination:** HTTP sources handle pagination automatically. You do not need to paginate manually; Coral fetches all pages up to any declared limits.
- **OAuth refresh:** If a source uses OAuth and the access token expires, Coral refreshes it transparently if a refresh token was issued. Otherwise, users reconnect with `coral source add --interactive <NAME>`.
- **Workspace isolation:** Each workspace has its own installed sources and credentials. Use `--workspace <NAME>` or `CORAL_WORKSPACE=<NAME>` to switch workspaces.
- **Secret visibility:** Secret values are never returned through `coral.inputs` or MCP discovery. Only `is_set` shows whether a secret is configured.
- **Source spec validation:** `coral source lint` validates syntax and schema but does not execute test queries. Use `coral source test` for strict pass/fail validation.
- **MCP client trust:** A connected MCP client can query any source installed in that workspace. Only connect Coral to agents you trust.

## Verification checklist

Before submitting work with Coral:

- [ ] Run `coral source list` to confirm all required sources are installed
- [ ] Run `coral sql "SELECT * FROM coral.tables"` to verify tables are discoverable
- [ ] Test a simple query: `coral sql "SELECT * FROM <schema>.<table> LIMIT 1"`
- [ ] For custom sources: run `coral source lint ./spec.yaml` and `coral source test <name>`
- [ ] For MCP setup: ask the agent to list Coral tables or run a test query
- [ ] Check secret storage: `coral source list` shows `keychain` or `file (plaintext)` for each source
- [ ] Verify filters: query `coral.filters` if your query uses `WHERE` predicates
- [ ] Inspect columns: query `coral.columns` to confirm column names and types match expectations

## Resources

**Comprehensive page navigation:** https://withcoral.com/docs/llms.txt

**Critical documentation:**
- [Bundled sources](https://withcoral.com/docs/reference/bundled-sources) — Full list of 25+ built-in sources and their configuration
- [Use Coral over MCP](https://withcoral.com/docs/guides/use-coral-over-mcp) — Setup instructions for Claude Code, Cursor, VS Code, and other agents
- [Write a custom source spec](https://withcoral.com/docs/guides/write-a-custom-source) — Complete guide with examples for HTTP APIs and file-backed sources
- [Source spec reference](https://withcoral.com/docs/reference/source-spec-reference) — Full YAML field reference for HTTP, file, and MCP backends
- [CLI reference](https://withcoral.com/docs/reference/cli-reference) — All commands and options
- [Security model](https://withcoral.com/docs/project/security) — Threat model, credential storage, and practical guidance

---

> For additional documentation and navigation, see: https://withcoral.com/docs/llms.txt