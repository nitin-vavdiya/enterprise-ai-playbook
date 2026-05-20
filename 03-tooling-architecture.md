# 03 — Tooling architecture

> How to set up AI coding agents with tool integrations, code indexes, and schema access so they actually have cross-repo context. Examples reference specific products only where a reader needs concrete starting points; the principles are vendor-neutral.

## The three layers of context

The agent needs three different kinds of access to your system. They're separate problems with separate solutions.

### Layer 1 — Static knowledge

Files in repos that the agent reads automatically. Zero infrastructure, highest ROI.

**What goes here:**
- `SERVICES.md` — system map (see `02a-services-map-template.md`)
- Per-agent instruction files at the repo root — most agents look for a conventional file (e.g. `AGENTS.md`, `CLAUDE.md`, `.cursorrules`) and load it automatically
- `contracts/` directory — OpenAPI specs, `.proto` files, Avro / JSON Schema, MQTT topic docs
- ADRs (Architectural Decision Records)

**Cost:** zero infrastructure, just files in git.
**Value:** highest-leverage starting point. In our experience this layer alone eliminates the bulk of "agent has no clue" problems at 15-50 service scale.

Build it first.

### Layer 2 — Code search across repos

When the agent asks "who calls `OrderService.createOrder()`?", it needs to grep across all your repos, not just the current one.

Three practical options, in order of effort:

**A. Multi-root workspaces (easiest, free)**

Most agentic IDEs support opening multiple repos in one workspace. Pattern:

```json
{
  "folders": [
    { "path": "../order-service" },
    { "path": "../payment-service" },
    { "path": "../catalog-service" },
    { "path": "../platform-docs" }
  ]
}
```

- Pros: works today, no infrastructure
- Cons: doesn't scale past ~10 repos, requires people to open the right workspace
- Best fit: domain-scoped workspaces (one per team)

Store workspace definitions in `platform-docs/workspaces/` so every engineer uses the same canonical groupings.

**B. Meta-repo with git submodules**

A `platform/` repo pulls in all services as submodules. Awkward for active development, good for read-only exploration.

**C. Code-search service exposed to the agent (best at 15-50 service scale)**

Run a service that indexes all your repos and exposes search through a tool integration the agent can call. Two delivery models:

- **Build a thin one yourself.** A small wrapper over a code-search engine (any of the open-source options below) plus a thin protocol layer to expose it to the agent. Refresh on a schedule.
- **Buy.** Commercial code-search platforms come with agent integrations. Faster to adopt, ongoing per-seat cost, lock-in risk.

This approach works from any repo without requiring a special workspace, scales well past the multi-root ceiling, and can expose structured queries (find callers, find consumers) beyond simple text search.

**Open-source code-search engines worth evaluating:**
- A high-performance trigram-indexed engine — fast and proven at scale
- A mature general-purpose engine — long track record, UI-focused, slower to integrate with agents
- Smaller agent-first projects — moving fast, smaller production footprint

Pick based on: index speed at your repo count, query latency target, and how easy it is to put a thin protocol layer in front of it.

### Layer 3 — Live system data

Most powerful, most work. MCP servers exposing:

- **Schema registry** — get current Avro schema, list schemas, find consumers
- **API gateway** — list registered routes, who hits which endpoint
- **Service mesh / observability** — runtime call graph from Istio, Linkerd, Jaeger
- **Event broker** — Kafka topics, consumer groups, lag

You probably won't build all of these. The schema registry has the highest ROI because contract changes cause the most production breaks.

**The internal tool registry pattern**

Enterprises increasingly run an internal registry of approved agent tool integrations. The pattern:

- Platform team builds a handful of custom tool servers (schema lookup, ticket lookup, deploy status, etc.)
- Tools are published to an internal registry with versioning and signatures
- Developer agents connect only to approved, signed tools from the registry

This is the governance layer on top of agent tooling. Both vendor-hosted registry products and self-hostable open-source registries exist. See `06-security.md` for why an allowlist + signature workflow is non-negotiable.

## Build order

See `07-roadmap.md` for the single canonical adoption roadmap that reconciles this chapter with `05-workflow-patterns.md`. Summarized:

1. **Foundation** — `platform-docs` repo, `SERVICES.md`, CI sync, agent instruction files.
2. **Contracts** — centralize OpenAPI / proto / Avro / MQTT specs; adopt contract-first PR workflow.
3. **Cross-repo code search** — index all repos, expose to the agent as a tool.
4. **Live data integrations (optional)** — schema registry, gateway, observability; pick highest-ROI one first.

## What surprises people

Layer 1 (just files) carries the bulk of the load. Teams jump to building elaborate tool servers and skip writing `SERVICES.md`, then discover the tooling doesn't help because the agent still doesn't know what services exist or who owns what.

**Files first, then plumbing.** Always.

## RAG over code: be skeptical as a first step

You might be tempted to throw all your code into a vector database and let the agent retrieve via similarity. As a first step, don't. Directed code search (grep / AST / structured queries) is more reliable for finding callers and references than "here are ten chunks that might be relevant." RAG over docs (ADRs, runbooks, design docs) is useful; RAG over code is overhyped.

The better framing is **progressive context disclosure**: load a small, always-relevant default (instruction file, README), and give the agent tools it can call to pull more context on demand. This avoids both context starvation and attention dilution.

## Agent-tooling notes

Tool integration support varies across agentic IDEs and CLIs. Common ground:

- Most popular agents now support some form of pluggable tool integration that the agent invokes during a session.
- Configuration usually lives in a per-repo or per-user file.
- Allowlisting, signature verification, and sandboxing are the responsibility of the platform team — see `06-security.md`.

When evaluating an agent for enterprise use, ask: how do tools get approved, how are tool responses sanitized before they re-enter the prompt, and what is the audit trail of tool calls?

## Reference architecture

```
                    ┌─────────────────────────┐
                    │   platform-docs repo    │
                    │ SERVICES.md, contracts/ │
                    └──────────┬──────────────┘
                               │ CI sync
                               ▼
        ┌──────────┬──────────┬──────────┬──────────┐
   ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
   │  svc 1 │ │  svc 2 │ │  svc 3 │ │  svc 4 │ │  ...   │
   │SERVICES│ │SERVICES│ │SERVICES│ │SERVICES│ │ 50+    │
   │CLAUDE  │ │CLAUDE  │ │CLAUDE  │ │CLAUDE  │ │ repos  │
   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
                               │
                               │ git clones
                               ▼
                  ┌───────────────────────────┐
                  │   Code-search indexer     │
                  └─────────────┬─────────────┘
                                │
                                ▼
                  ┌───────────────────────────┐
                  │  Tool server              │
                  │  search_code,             │
                  │  find_callers,            │
                  │  find_consumers           │
                  └─────────────┬─────────────┘
                                │ (signed, allowlisted)
                                ▼
                  ┌───────────────────────────┐
                  │  AI coding agent          │
                  │  (developers' machines)   │
                  └───────────────────────────┘
```
