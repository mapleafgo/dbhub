**Fork notice:** this repository is a fork of [bytebase/dbhub](https://github.com/bytebase/dbhub). It tracks upstream `main` and adds **project config discovery** — when `--config` is omitted, DBHub loads `dbhub.toml` from the working directory, so the project you open picks the database. See [Project Config Discovery](#project-config-discovery).

> [!NOTE]  
> If you need an enterprise-level database MCP server with built-in guardrails like approval flow, access control, data masking, and audit logging beyond what DBHub offers, check out [Bytebase](https://www.bytebase.com/).

<p align="center">
 <a href="https://www.star-history.com/bytebase/dbhub">
  <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/badge?repo=bytebase/dbhub&type=trending&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/badge?repo=bytebase/dbhub&type=trending" />
   <img alt="GitHub Trending Repository of the Day" src="https://api.star-history.com/badge?repo=bytebase/dbhub&type=trending" />
  </picture>
 </a>
</p>

<p align="center">
<a href="https://dbhub.ai/" target="_blank">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/bytebase/dbhub/main/docs/images/logo/full-dark.svg" width="75%">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/bytebase/dbhub/main/docs/images/logo/full-light.svg" width="75%">
  <img src="https://raw.githubusercontent.com/bytebase/dbhub/main/docs/images/logo/full-light.svg" width="75%" alt="DBHub Logo">
</picture>
</a>
</p>

```bash
            +------------------+    +--------------+    +------------------+
            |                  |    |              |    |                  |
            |  Claude Desktop  +--->+              +--->+    PostgreSQL    |
            |                  |    |              |    |                  |
            |  Claude Code     +--->+              +--->+    SQL Server    |
            |                  |    |              |    |                  |
            |  Cursor          +--->+    DBHub     +--->+    Oracle        |
            |                  |    |              |    |                  |
            |  VS Code         +--->+              +--->+    SQLite        |
            |                  |    |              |    |                  |
            |  Copilot CLI     +--->+              +--->+    MySQL         |
            |                  |    |              |    |                  |
            |                  |    |              +--->+    MariaDB       |
            |                  |    |              |    |                  |
            +------------------+    +--------------+    +------------------+
                 MCP Clients           MCP Server             Databases
```

DBHub is a minimal MCP server: token-efficient, zero-dependency, and just two tools by default with opt-in extras. This lightweight gateway allows MCP-compatible clients to connect to and explore different databases:

- **Minimal**: Zero dependency, token efficient with a minimal set of MCP tools to maximize context window
- **Multi-Database**: PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, and SQLite through a single interface
- **Multi-Connection**: Connect to multiple databases simultaneously with TOML configuration
- **Guardrails**: Read-only mode, row limiting, and query timeout to prevent runaway operations
- **Secure Access**: SSH tunneling and SSL/TLS encryption

> DBHub is the official example in the [Claude Code docs](https://code.claude.com/docs/en/mcp#example-query-your-postgresql-database) for connecting to PostgreSQL via MCP.

## Token Efficiency

DBHub loads just 2 tools by default at **1.4k tokens** — 13-14x fewer than alternatives — keeping the context window open for your actual work.

| MCP Server | Default Config | Default Tools |
|------------|---------------|--------------|
| **DBHub** | **1.4k** | 2 (`execute_sql`, `search_objects`) |
| MCP Toolbox | 19.0k | 28 |
| Supabase MCP | 19.3k | all |

## Use Cases

- **Local Development**: Schema exploration, query validation, and data debugging with Claude Code, VS Code, Cursor, etc.
- **Non-Technical Access**: Expose curated, read-only views to non-technical staff via Claude Desktop, VS Code, Cursor, etc.
- **Multi-Database Consolidation**: Replace separate MCP servers for each database with a single DBHub process
- **Production Troubleshooting**: Read-only diagnostics with guardrails against runaway queries

## Supported Databases

PostgreSQL, MySQL, SQL Server, MariaDB, Oracle, and SQLite.

## MCP Tools

DBHub implements MCP tools for database operations:

- **[execute_sql](https://dbhub.ai/tools/execute-sql)**: Execute SQL queries with transaction support and safety controls
- **[search_objects](https://dbhub.ai/tools/search-objects)**: Search and explore database schemas, tables, columns, indexes, and procedures with progressive disclosure
- **[explain_sql](https://dbhub.ai/tools/explain-sql)** (opt-in): Show a query's execution plan without running it
- **[health_check](https://dbhub.ai/tools/health-check)** (opt-in): Report connection pool state and buffer cache hit ratio
- **[Custom Tools](https://dbhub.ai/tools/custom-tools)**: Define reusable, parameterized SQL operations in your `dbhub.toml` configuration file

## Workbench

DBHub includes a [built-in web interface](https://dbhub.ai/workbench/overview) for interacting with your database tools. It provides a visual way to execute queries, run custom tools, and view request traces without requiring an MCP client.

![workbench](https://raw.githubusercontent.com/bytebase/dbhub/main/docs/images/workbench/workbench.webp)

## Installation

```bash
npx @bytebase/dbhub@latest --transport http --port 8080 --dsn "postgres://user:password@localhost:5432/dbname?sslmode=disable"
```

Also available as:

- [Docker image](https://dbhub.ai/installation#docker)
- [MCP Bundle](https://dbhub.ai/mcpb) (one-click install, read-only)
- [Claude Code plugin](https://dbhub.ai/claude-code-plugin)

See the [Installation Guide](https://dbhub.ai/installation) for all options, [Command-Line Options](https://dbhub.ai/config/command-line) for parameters, and [Multi-Database Configuration](https://dbhub.ai/config/toml) for connecting several databases at once.

## Project Config Discovery

This fork adds one behavior on top of upstream: **when `--config` is omitted, DBHub loads `./dbhub.toml` from the working directory if one exists.**

MCP clients (Claude Code, Codex, Cursor) launch DBHub with the working directory set to the project you opened, so a `dbhub.toml` at that project's root is picked up automatically — the project selects the database, with no per-project MCP configuration and no registration step.

```bash
# In a project whose root has dbhub.toml: uses it
npx @bytebase/dbhub@latest

# Names the file explicitly; wins over the project config
npx @bytebase/dbhub@latest --config ./dbhub.toml
```

Resolution order is `--config` → `<cwd>/dbhub.toml` → `DSN` / `DB_*` / `.env`. Only the working directory is checked — parent directories are not searched, so a session started in a subdirectory does not silently bind to a config further up. The auto-discovered path is logged at startup.

Because a TOML file defines its own sources, a TOML config and `--dsn` cannot be combined; when both are present DBHub fails with an error naming the actual remedy instead of silently picking one. This is a breaking change relative to upstream: a `dbhub.toml` in the working directory is now loaded without being named.

## Development

Requires Node.js >= 22.5.0 (DBHub uses the built-in `node:sqlite` module).

```bash
# Install dependencies
pnpm install

# Run in development mode
pnpm dev

# Build and run for production
pnpm build && pnpm start --transport stdio --dsn "postgres://user:password@localhost:5432/dbname"
```

See [Testing](.claude/skills/testing/SKILL.md) and [Debug](https://dbhub.ai/config/debug).

## Contributors

<a href="https://github.com/bytebase/dbhub/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=bytebase/dbhub" />
</a>

## Star History

<a href="https://www.star-history.com/?repos=bytebase%2Fdbhub&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=bytebase/dbhub&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=bytebase/dbhub&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=bytebase/dbhub&type=date&legend=top-left" />
 </picture>
</a>
