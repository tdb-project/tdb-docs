# Connect an IDE or AI Tool (MCP)

This page covers how to point popular IDEs and AI clients at TDB's MCP endpoint
(`POST /v1/mcp`) and how to run your first query once connected. It assumes you
already have TDB running and at least one source registered — see the
[Community quickstart](community.md) or [Enterprise quickstart](quickstart.md) if not.

!!! info "Community vs Enterprise"
    Community exposes one MCP tool, `query_source`, against the single registered
    CSV (table name always `data`). Enterprise exposes seven tools — including
    schema/preview/filter/aggregate helpers that don't require the model to write
    SQL — against real table names. See [MCP API reference →](../api/mcp.md) for
    the full tool list.

Every client below speaks the same protocol, so the underlying connection details
never change:

| Setting | Value |
|---|---|
| URL | `http://localhost:8000/v1/mcp` (swap for your deployed host) |
| Transport | Streamable HTTP |
| Auth header | `Authorization: Bearer <YOUR_KEY>` |

What differs between clients is the **config file format and key names** — several
clients use subtly different field names for the same thing, which is the most
common source of a silent "server not connecting" failure. The table below is a
quick reference; details and JSON for each client follow.

| Client | Config location | Top-level key | URL field | Native header/auth support |
|---|---|---|---|---|
| [Claude Desktop](#claude-desktop) | The TDB extension (`tdb.mcpb`); Settings → Extensions | — | Entered in the extension's settings | Yes — the extension adds the key |
| [Claude Code](#claude-code) | The TDB plugin, or `claude mcp add` (stored in `~/.claude.json`; `--scope project` writes `.mcp.json`) | `mcpServers` | `url` | Yes — `--header` |
| [VS Code](#vs-code) | The TDB extension, or `.vscode/mcp.json` / user profile | `servers` (not `mcpServers`) | `url` | Yes — `headers`, plus `inputs` for secret prompting |
| [Cursor](#cursor) | `.cursor/mcp.json` | `mcpServers` | `url` | Yes — `headers` |
| [Windsurf (Cascade)](#windsurf-cascade) | `~/.codeium/windsurf/mcp_config.json` | `mcpServers` | `serverUrl` (not `url`) | Yes — `headers` |
| [JetBrains IDEs](#jetbrains-ides) | Settings UI (AI Assistant) | `mcpServers` | `url` | Not documented for HTTP servers — use the stdio bridge below |
| [Cline](#cline) | Cline sidebar → Configure MCP Servers | `mcpServers` | `url` | Yes — `headers`, but requires explicit `"type": "streamableHttp"` |

!!! warning "Don't commit raw API keys into project config files"
    `.vscode/mcp.json`, `.cursor/mcp.json`, and Windsurf's config can live inside a
    project folder that gets checked into git. Prefer the environment-variable
    interpolation each client offers (shown below) over pasting the raw key, and
    add the file to `.gitignore` if you do paste it.

---

## Claude Desktop

Use the TDB extension:

1. Download `tdb.mcpb` from the
   [latest tdb-connectors release](https://github.com/tdb-project/tdb-connectors/releases/latest)
   and open it to install.
2. In Settings → Extensions → **The Data-Bridge (TDB)**, enter the server URL
   (`http://localhost:8000`) and your API key. Claude Desktop keeps the key in your
   system's credential store.

`query_source` (Community) or the Enterprise tools your key may use (all seven unless it
has a [tool allow-list](../api/mcp.md#tool-allow-lists-per-api-key)) then appear in the
tools list. The extension runs a small local bridge that forwards to TDB over HTTP, so it
works with TDB on your own machine or private network.

TDB's audit log names the client as
`local-agent-mode-The Data-Bridge (TDB)/1.0.0 (via tdb-bridge/0.1.0)`: Claude Desktop
reports the extension's name, not its own. Tested on Windows against community 0.12.0.

!!! note "Not `claude_desktop_config.json`"
    That file is for local servers that Claude Desktop starts itself; a `url` entry there
    does not connect to TDB. Remote servers are added as custom connectors, which Anthropic's
    cloud reaches, so they need TDB at a public HTTPS URL — see
    [Connecting Claude Desktop →](../auth/oauth.md#connecting-claude-desktop).

---

## Claude Code

### With the TDB plugin (recommended)

In a terminal, with Claude Code 2.1.292 or later:

```bash
claude plugin install tdb --marketplace tdb-project/tdb-connectors
```

On earlier versions, run `claude plugin marketplace add tdb-project/tdb-connectors`
first, then `claude plugin install tdb@tdb-connectors`.

Then, in Claude Code, run `/plugin configure tdb@tdb-connectors` to enter the server URL
and your API key, and `/reload-plugins` to connect. `/mcp` should show `tdb` as
connected. The key goes into your system's credential store, not into settings. The
plugin also teaches Claude how to query TDB: read-only SQL, the table name, and what
`truncated` means.

Tested with Claude Code 2.1.310 against community 0.12.0; the audit log names the
client `claude-code/2.1.310`.

### By hand

```bash
claude mcp add --transport http tdb http://localhost:8000/v1/mcp --header "Authorization: Bearer YOUR_API_KEY"
```

Then run `claude mcp list`; `tdb` should show as connected. The command uses the
default `local` scope, which keeps the key in your user configuration
(`~/.claude.json`), not in the project. `--scope project` writes it to `.mcp.json`
in the project folder instead, so don't use that with a raw key in a repository.

Claude Code connects with MCP `2026-07-28`, TDB's stateless revision: it was tested
with Claude Code 2.1.300 against community 0.12.0.

---

## VS Code

### With the TDB extension (recommended)

Install **The Data-Bridge (TDB)** from the
[VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=tdb-project.tdb-vscode)
or [Open VSX](https://open-vsx.org/extension/tdb-project/tdb-vscode). Then run
**TDB: Connect to Server** from the Command Palette and enter the server URL and
your API key. The key goes into VS Code's encrypted secret storage, not into a file.
TDB's tools then appear in agent mode, and the status bar shows the server's version.

The extension needs desktop VS Code 1.101 or later. It also sends the
`MCP-Protocol-Version` header that VS Code itself leaves out, so your agent gets tool
titles, read-only hints and structured results. Without the header, TDB answers with
the oldest protocol version. **TDB: Disconnect** removes the key and the server.

### By hand

Add a `.vscode/mcp.json` in your workspace (or run **MCP: Add Server** from the
Command Palette to create one in your user profile):

```json
{
  "servers": {
    "tdb": {
      "type": "http",
      "url": "http://localhost:8000/v1/mcp",
      "headers": {
        "Authorization": "Bearer ${input:tdb-api-key}",
        "MCP-Protocol-Version": "2025-11-25"
      }
    }
  },
  "inputs": [
    {
      "type": "promptString",
      "id": "tdb-api-key",
      "description": "TDB API key",
      "password": true
    }
  ]
}
```

!!! warning "VS Code uses `servers`, not `mcpServers`"
    This is the single most common mistake when copying a Claude Desktop or Cursor
    config into VS Code — the file is silently ignored if the top-level key is wrong.

VS Code agrees `2025-11-25` with the server when it connects but never repeats the
`MCP-Protocol-Version` header, so TDB would answer every later request as
`2024-11-05`. The extra header gets your agent tool titles, read-only hints and
structured results. Servers older than community 0.11.0 / enterprise 0.17.0 ignore it.

The `inputs` block above prompts you for the key on first connect and masks it in
the UI, instead of storing it in plaintext in a file that might be committed. If
you'd rather hardcode it for local testing, replace `${input:tdb-api-key}` with the
raw `Bearer YOUR_API_KEY` string and drop the `inputs` array.

Save, then open the Command Palette and run **MCP: List Servers** to confirm `tdb`
shows as connected. This requires GitHub Copilot Chat with agent mode enabled.

---

## Cursor

Open **Cursor Settings → MCP → Add Server**, or edit `.cursor/mcp.json`
(`~/.cursor/mcp.json` for a global server, `.cursor/mcp.json` in a project root for
a project-scoped one):

```json
{
  "mcpServers": {
    "tdb": {
      "url": "http://localhost:8000/v1/mcp",
      "headers": { "Authorization": "Bearer ${env:TDB_API_KEY}" }
    }
  }
}
```

Cursor resolves `${env:VAR_NAME}` against your shell environment at connect time, so
`export TDB_API_KEY=...` before launching Cursor instead of pasting the raw key.

!!! tip "Enterprise: OAuth instead of a static key"
    Same as Claude Desktop — see [Connecting Cursor →](../auth/oauth.md#connecting-cursor).

---

## Windsurf (Cascade)

Edit `~/.codeium/windsurf/mcp_config.json`:

```json
{
  "mcpServers": {
    "tdb": {
      "serverUrl": "http://localhost:8000/v1/mcp",
      "headers": { "Authorization": "Bearer ${env:TDB_API_KEY}" }
    }
  }
}
```

!!! warning "Windsurf uses `serverUrl`, not `url`"
    Every other client in this list uses `url`. Pasting a Claude Desktop or Cursor
    config as-is into Windsurf produces a server entry with no endpoint and it will
    fail to connect without an obvious error.

Reload Cascade's MCP panel from the Windsurf sidebar to pick up the change.

---

## JetBrains IDEs

Applies to IntelliJ IDEA, PyCharm, WebStorm, GoLand, Rider, and other JetBrains
IDEs with the **AI Assistant** plugin (2025.2+), via
**Settings → Tools → AI Assistant → Model Context Protocol (MCP) → Add**.

JetBrains' documented HTTP configuration is URL-only:

```json
{ "mcpServers": { "tdb": { "url": "http://localhost:8000/v1/mcp" } } }
```

— there is no documented field for custom headers, so there's no way to attach the
required `Authorization: Bearer` token directly. If your IDE version's Add dialog
doesn't expose a headers field, use the **stdio bridge** workaround instead: add the
server as a **command**, not a URL, using the community
[`mcp-remote`](https://github.com/geelen/mcp-remote) tool to inject the header:

| Field | Value |
|---|---|
| Command | `npx` |
| Arguments | `-y mcp-remote http://localhost:8000/v1/mcp --header "Authorization: Bearer YOUR_API_KEY"` |

This runs `mcp-remote` as a local subprocess that relays stdio to TDB's HTTP
endpoint with the header attached — a general escape hatch for any MCP client that
doesn't yet support auth headers on remote servers natively (requires Node.js).

---

## Cline

In VS Code, open the Cline sidebar → **MCP Servers → Configure MCP Servers**, which
opens a JSON file:

```json
{
  "mcpServers": {
    "tdb": {
      "type": "streamableHttp",
      "url": "http://localhost:8000/v1/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

!!! warning "Set `\"type\": \"streamableHttp\"` explicitly"
    Omitting `type` causes Cline to fall back to the deprecated SSE transport
    instead of Streamable HTTP, which TDB does not speak — the server will appear
    added but never connect.

---

## Running your first query

Once a client shows `tdb` as connected, don't write SQL yourself — describe what
you want and let the assistant call the tool.

### Community (one CSV, table name `data`)

- *"Using the tdb `query_source` tool, how many rows are in my data?"*
- *"Show me the first 5 rows."*
- *"What columns does the dataset have, and what does a typical row look like?"*
- *"List the top 10 customers by revenue, highest first."*

TDB caps every response at 1,000 rows and enforces read-only SQL, so the assistant
can explore freely without being able to modify anything. If a result comes back
`"truncated": true`, ask it to narrow the query — a `WHERE` clause or an aggregate
instead of `SELECT *`.

### Enterprise (multiple sources, real table names)

With more than one source registered, name it explicitly so the assistant doesn't
default to the first one:

- *"Using the `production_db` source, show me the schema."* → calls `schema_source`
- *"Preview 5 rows from `production_db`."* → calls `preview_source`
- *"In `production_db`, how many customers are there per country?"* → calls
  `aggregate_source` or `query_source` depending on the model
- *"Join customers and orders in `production_db` and show total spend per customer,
  highest first."* → calls `query_source` with a hand-written JOIN

Steering the assistant to the no-SQL tools (`schema_source`, `preview_source`,
`filter_source`, `aggregate_source`) first is useful when you want it to explore a
source without risking a malformed query — see the
[full tool reference →](../api/mcp.md#available-tools) for what each one accepts.

---

### Smaller results for the assistant

From community 0.13.0 / enterprise 0.21.0, TDB can send results in a smaller form
([Lean Mode](../api/mcp.md#saving-tokens-lean-mode)): rows as arrays, and optionally
long text cut with a marker. Nothing in the client changes. Ask for it in the prompt
(*"use lean: true"*), or have the operator turn it on for everyone with
`TDB_LEAN_MODE=on` or, in enterprise, for one API key.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Server shows disconnected, no error | Wrong top-level key (`servers` vs `mcpServers`) or wrong URL field (`serverUrl` vs `url`) — see the per-client warnings above |
| `401` / "Unauthorized" from the assistant | Header not actually sent — check the client supports `headers` natively (JetBrains does not; use the `mcp-remote` bridge) |
| Assistant answers without calling the tool | Name the tool explicitly in your prompt: *"use the tdb server's `query_source` tool"* |
| `ENOENT` or immediate disconnect | A `command`/stdio entry pointed at something that isn't a persistent process (e.g. `curl`) — TDB is HTTP-only; use a `url`/`type: "http"` entry, or `mcp-remote` if the client needs stdio |

For the full JSON-RPC method and error-code reference, see [MCP API →](../api/mcp.md).
