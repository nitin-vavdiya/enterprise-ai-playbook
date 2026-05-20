# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A documentation playbook for making AI coding agents work effectively across enterprise multi-repo environments (15-50 microservices). No code, no build system — pure Markdown.

## Document map

| File | Contents |
|------|----------|
| `00-overview.md` | The four core problems AI agents face at multi-repo scale |
| `01-platform-docs-setup.md` | Strategy for a central `platform-docs` repo synced to all service repos |
| `02a-services-map-template.md` | Blank `SERVICES.md` scaffold (start here when adopting) |
| `02b-services-map-example.md` | Fully filled-in worked example using fictional Acme org |
| `03-tooling-architecture.md` | Three-layer context stack: static files → code search → live data |
| `04-governance.md` | Tier system, context rules, failure modes, evals, regulatory, cost, fleets |
| `05-workflow-patterns.md` | Contract-first, consumer-driven contract tests, event catalogues, ADRs |
| `06-security.md` | Threat model for agent tooling: injection, tool poisoning, secrets, audit |
| `07-roadmap.md` | Single canonical adoption roadmap reconciling 03 and 05 |

## Editing principles

**1. Vendor-neutral by default.** This is a high-level guideline document. Strip product names from body text; keep them only where a reader needs concrete starting points (and even then, prefer category descriptions over specific products).

**2. No unsupported numbers.** Quantitative claims ("70-80%", "30-50 ADRs", "$X/user/month", "~200 lines of code") rot. Use qualitative framing or mark explicitly as anecdotal.

**3. Time-stamped claims need an owner.** Any "as of <date>" statement implies a refresh obligation. The repo footer in `README.md` commits to a six-month refresh cadence.

**4. The example is fictional and stays in 02b.** Acme, `order-service`, `payment-service`, etc. live only in `02b-services-map-example.md`. Don't seed example services into other chapters — readers will think they're recommended names.

**5. Keep ASCII diagrams.** They survive renders that strip Mermaid. The 2D matrix and roadmap diagrams in `04` and `07` are load-bearing.

**6. Cross-reference, don't duplicate.** Build orders live canonically in `07-roadmap.md`; the four problems live canonically in `00-overview.md`. Other chapters link, not copy.

## Key concepts

**Three context layers** (`03-tooling-architecture.md`):
- Layer 1 — static files in repos (highest leverage, zero infrastructure)
- Layer 2 — cross-repo code search exposed as a tool
- Layer 3 — live system data (schema registry, gateway, observability)

**Six governance questions** (`04-governance.md`): autonomy, context, failure response, evaluation, regulatory/compliance, cost. Security threats live separately in `06-security.md`.

**Autonomy tier system** (`04`): Green / Yellow / Red zones × Levels 0-3 of automation. Encoded as file-path patterns, enforced in CI.

**Four workflow patterns** (`05`): ADRs → contract-first → event catalogue → CDC testing.

**The central thesis:** the model isn't the bottleneck — the environment is. Files first, then tooling, then governance, then workflow patterns. Roadmap in `07`.

## When editing

- When adding a service to `02b-services-map-example.md`: also update its section, the event topology table, the change-impact cheatsheet, and the system overview diagram.
- When introducing a new claim that depends on time or vendor: tag it explicitly or push to an appendix.
- When adding a new chapter: update this file, `README.md` doc map, and the roadmap in `07`.
