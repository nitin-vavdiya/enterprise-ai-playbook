# 00 — Overview: the four problems

> When AI coding agents work in single-repo setups, they perform well. In enterprise environments with many services communicating over multiple protocols, they fail in specific, predictable ways. Understanding these failures is the foundation for everything else in this documentation.

## The setting

A typical enterprise development environment looks like this:

- 15-50+ microservices across separate repositories
- Communication via multiple protocols: HTTP REST, gRPC, Kafka events, MQTT, message queues
- Multiple teams, each owning a subset of services
- Decisions made years ago that nobody remembers the reason for
- Tribal knowledge spread across Slack, Confluence, ADRs, senior engineers

In this setting, AI coding agents face four core problems.

## Problem 1 — Context fragmentation

The agent sees what's in its context window. Your system isn't in that window.

When you ask the agent to rename a field on a User model, it confidently makes the change in the current repo. What it doesn't see:

- The 7 other microservices consuming that field
- The Kafka event schema in your registry
- The Confluence page documenting why the field was named that way
- The ADR from 18 months ago explaining the constraint

Smaller context window ≠ smaller blast radius. The agent's view is local; your system is distributed.

**This is the dominant failure mode at multi-repo scale.**

## Problem 2 — Cross-repo impact

Service A → calls Service B → calls Service C. The agent modifies C. It updates direct callers (B), ships. Two days later: Service A breaks in production, the mobile app crashes, the reporting pipeline silently produces wrong numbers.

Direct callers are easy to find with grep. Transitive callers are where bugs hide. Multiply across 50 services and dozens of teams, and "blast radius" becomes a question no single repo can answer.

## Problem 3 — Protocol diversity

Most AI tooling is REST-centric. Your services aren't.

Every communication protocol has a contract:

- REST → OpenAPI / Swagger
- gRPC → `.proto` files
- Events → Avro / Protobuf schemas
- MQTT → topic structure + payload

Each is a contract. Breaking any of them breaks a consumer. Most agents only check the REST one — and even then, only the spec, not the consumers.

## Problem 4 — Change propagation

"Who consumes my API?" has no single answer. The truth lives in pieces:

- API gateway logs
- Service mesh routing tables
- Event broker consumer groups
- Tribal knowledge

Until that graph exists in one place, every change is a guess.

## The mental model

These four problems share a structure: **the agent has bounded context, but your system has unbounded dependencies**. The agent isn't broken — it's working with incomplete information.

Three layers of context bring it closer to working well:

```
┌─────────────────────────────────────────────────────┐
│ Layer 1 — Static knowledge                          │
│ SERVICES.md, CLAUDE.md, contracts in git            │
│ (files in repos, synced via CI)                     │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Layer 2 — Cross-repo code search                    │
│ Find usages, callers, similar patterns              │
│ (multi-root workspaces, Zoekt + MCP, Sourcegraph)   │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Layer 3 — Live system data                          │
│ Schema registry, API gateway logs, service mesh     │
│ (custom MCP servers wrapping your platform)         │
└─────────────────────────────────────────────────────┘
```

Build top to bottom. Layer 1 is the highest-leverage starting point — in our experience it eliminates the bulk of "agent has no clue" problems at 15-50 service scale. Layer 2 helps when grep across repos isn't enough. Layer 3 is the most powerful and the most work.

## The fix isn't a smarter model

The temptation is to wait for better AI. Don't. Even with a perfect model, the four problems persist because they're about *information your model doesn't have access to*. The investment that pays off is in plumbing:

- Make the information legible (SERVICES.md, contracts, ADRs)
- Make it reachable (files in every repo, MCP servers)
- Make it enforced (CI checks, CODEOWNERS, schema compatibility rules)

Once that's in place, the agent gets dramatically better — and so do your humans.

## What follows

The rest of this documentation set walks through how to actually build this:

- `**01-platform-docs-setup.md*`* — A central repo to hold the canonical context, synced to every service repo
- `**02-services-map.md**` — The single highest-leverage artifact: a map of your services
- `**03-tooling-architecture.md**` — MCP servers, code search, the three-layer stack
- `**04-governance.md**` — Three questions every org must answer before scaling AI usage
- `**05-workflow-patterns.md**` — Contract-first, CDC tests, event catalogues, ADRs

