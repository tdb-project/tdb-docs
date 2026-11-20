# Claude in a private network

TDB can run inside a network zone with **no internet access, in either direction**, and
still serve Claude. What decides it is where the Claude app connects *from*:

- **From the user's own computer** (Claude Desktop with the TDB extension, Claude Code): the
  computer reaches TDB over your internal network. TDB needs no inbound or outbound
  internet access.
- **From Anthropic's cloud** (claude.ai on the web and mobile, and *custom connectors* in
  Claude Desktop): Anthropic's servers call TDB, so TDB needs a public HTTPS address.

So a private TDB works with Claude Desktop and Claude Code, and not with claude.ai on the
web or mobile.

## Which route to use

| Claude app | Connects from | TDB needs | Sign-in | Set up |
|---|---|---|---|---|
| **Claude Desktop** with the TDB extension | The user's computer (a small local bridge) | Reachable from the user's computer only | An API key per person | [Claude Desktop](../getting-started/ide-setup.md#claude-desktop) |
| **Claude Code** with the TDB plugin | The user's computer | Reachable from the user's computer only | An API key per person | [Claude Code](../getting-started/ide-setup.md#claude-code) |
| **Claude Code**, by hand with sign-in | The user's computer | Reachable from the user's computer only; HTTPS (an internal certificate is fine) | [Single sign-on](../auth/sso.md) in the user's browser | [Claude Code](../getting-started/ide-setup.md#by-hand) |
| **claude.ai** (web, mobile) or a Desktop **custom connector** | Anthropic's cloud | A public HTTPS address | Sign-in page (single sign-on, or the admin login) | [Connecting Claude Desktop](../auth/oauth.md#connecting-claude-desktop) |

## The network

```
  ┌──────────── your secure zone ─────────────┐
  │                                           │
  │  databases ◀── TDB ◀────────────┐         │       ┌──────────────────┐
  │                (no internet)    │         │       │ Anthropic        │
  │                                 │ :443    │       │ (the model)      │
  └─────────────────────────────────┼─────────┘       └────────▲─────────┘
                                    │                          │ HTTPS
                          ┌─────────┴──────────────────────────┴──┐
                          │ user's computer:                      │
                          │ Claude Desktop + TDB extension,       │
                          │ or Claude Code                        │
                          └───────────────────────────────────────┘
```

**Firewall rules this needs:**

- **User computers → TDB:** the port your TLS proxy serves (or 8000 on a trusted network).
- **TDB → your data sources:** as for any deployment ([network and ports](it-admin-guide.md#13-network-ports)).
- **TDB → the internet: nothing.** TDB never calls home, and its licence is checked offline.
  Only features you turn on reach out: Splunk or S3 export, an approval webhook, and client
  metadata documents (`TDB_OAUTH_CIMD_HOSTS`), each to the one host you configure.
- **The internet → TDB: nothing.**

The user's computer already reaches both your internal network and Claude, as it does for
any internal tool.

!!! warning "What still leaves the zone"
    The question and the rows a tool returns go to Anthropic's model as part of the
    conversation, whichever route you use. TDB decides what may be read (roles, tool
    allow-lists, [column and row policies](../security/policies.md), [approvals](../security/approvals.md))
    and records every query in the [audit log](../security/audit.md). It does not change
    where the model runs. Use policies to keep columns that must not leave out of reach.

## Rolling it out to many people

**Give each person their own API key**, never a shared one. The audit log records the first
six characters of the key with every query (`key_hint`), so a key per person is what lets you
say who asked; a [policy rule](../security/policies.md) can match the key by name. With a
shared key, every query looks the same.

```bash
curl -X POST https://tdb.internal/v1/auth/keys \
  -H "Authorization: Bearer <ADMIN_KEY>" -H "Content-Type: application/json" \
  -d '{"name": "alice.chen", "role": "read"}'
```

The response holds `raw_key` once; hand it to that person. Narrow what a key may do with a
[tool allow-list](../api/mcp.md#tool-allow-lists-per-api-key), and revoke it when they leave.

Each person installs the extension (`tdb.mcpb`, one file from the
[tdb-connectors releases](https://github.com/tdb-project/tdb-connectors/releases/latest)) and
enters TDB's internal URL and their key. Claude Desktop keeps the key in the computer's
credential store. In an environment without access to GitHub, download the file once and
share it internally.

**With single sign-on** (Mid-Market), Claude Code users can sign in as themselves instead of
holding a key, and the audit log names the person (`user`): add TDB by hand with `claude mcp add --transport http tdb
https://tdb.internal/v1/mcp` and sign in when Claude Code opens the browser. The sign-in page
is reached from the user's browser on your network, so it needs no public address.
`TDB_SERVER_URL` must be HTTPS; a certificate from your internal CA is enough. The Claude
Desktop extension uses an API key today.

## What does not work in a closed zone

- **claude.ai on the web and mobile, and custom connectors**: Anthropic's servers would have
  to reach TDB. If you need them, put TDB (or a proxy in front of it) at a public HTTPS
  address and read [What TDB does not protect you from](../architecture.md#what-tdb-does-not-protect-you-from)
  first.
- **Client metadata documents** (`TDB_OAUTH_CIMD_HOSTS`): TDB would fetch each client's
  document from the internet. Leave it off; registration still works.
