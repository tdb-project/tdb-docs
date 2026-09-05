---
hide:
  # Single-page tab: with navigation.tabs this page is already its own tab, so
  # the left sidebar renders one entry repeating the tab's label. Hiding it
  # drops the duplicate and gives the content the full width.
  - navigation
---

# Why TDB

## The problem

Your data already exists — scattered across PostgreSQL, MySQL, SQL Server, Snowflake,
and CSV files. Now every team wants AI agents and assistants to query it.

The hard part is doing that **safely**. Each source exposes data differently, and wiring
an AI tool straight into a production database — with no access control and no record of
what it asked — is how data leaks. Teams need governed, auditable, read-only access that
works across the databases they already run, without handing everything to a third party.

## The common approaches today — and where they leave a gap

Most teams reach for one of three existing options. Each is good at what it does. None
gives you governed, self-hosted, multi-source access to the data you *already have*.

**Warehouse-native AI assistants** — the AI layer built into a single cloud data
warehouse. Excellent for data that already lives in that one warehouse. But your data has
to *be* in that warehouse first; it doesn't govern your other databases; it isn't
self-hosted on your own infrastructure; and the audit trail belongs to the vendor, not to
you.

**SaaS-embedded copilots** — the assistant built into a productivity or SaaS suite. Great
for documents and email. But it's SaaS: your data and query context flow through the
vendor's cloud, it isn't a governed gateway over your production databases, and embedded
copilots have been shown to be coaxable — via prompt injection — into exfiltrating data
they could reach.

**Do-it-yourself MCP servers** — quick to stand up, but they typically ship with no
authentication, no audit log, and no access control. (The widely-used reference
open-source Postgres MCP server was archived after a critical security flaw — and is still
downloaded tens of thousands of times a week because nothing better exists at this tier.)

## What TDB does differently

TDB is a lightweight, self-hosted middleware that sits in front of your existing data
sources — **all of them** — on **your own infrastructure**:

- **Self-hosted** — data never leaves your perimeter; nothing is routed through a SaaS vendor.
- **Multi-source** — connects to the databases you already run; no migration, no consolidation.
- **REST + SQL + MCP, simultaneously** — one governed surface for dashboards, scripts, and any MCP-compatible AI tool.
- **Audited by default** — every query, result, and tool call is logged to a tamper-evident audit log that *you* own.
- **Governed** — OAuth 2.1, role-based access control, per-key rate limiting, and prompt-injection filtering.
- **Open-core** — start free on the community edition; upgrade for production and multi-source.

In short: the others answer *"AI on the data that already lives inside our platform."*
TDB answers **"governed AI access to the data you already have, wherever it already is —
without moving it, and without surrendering the audit trail."**

## Who it's for

Compliance-conscious mid-market teams — particularly in **healthcare-tech and fintech** —
that need to make existing data accessible to AI agents without sending it to a SaaS
vendor or consolidating everything into a single cloud warehouse.

---

Ready to try it? → [Quickstart](getting-started/quickstart.md) ·
[Community Edition (free)](getting-started/community.md) ·
[Compare editions](pricing.md)
