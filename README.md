# Enterprise AI Playbook

> A vendor-neutral playbook for running AI coding agents across enterprise multi-repo environments. Covers context architecture, governance, security, workflow patterns, and adoption roadmap.

## What this is

When AI coding agents operate inside a single repo, they work well. When the same agents operate across an enterprise with many services, communication protocols, and teams, they fail in specific, predictable ways.

This documentation set captures patterns that address those failures — built around four problems, six governance questions, and four workflow patterns.

## How to read this

Start with `00-overview.md` for the conceptual framing. Then read in order, or jump to whichever piece you need.

```
docs/
├── 00-overview.md                — The 4 problems, the mental model
├── 01-platform-docs-setup.md     — Central source-of-truth repo strategy
├── 02a-services-map-template.md  — Blank SERVICES.md scaffold (start here)
├── 02b-services-map-example.md   — Worked example using fictional Acme org
├── 03-tooling-architecture.md    — Code indexes, tool integrations, three layers
├── 04-governance.md              — Boundaries, context, failures, evals, compliance, cost
├── 05-workflow-patterns.md       — Contract-first, CDC tests, event catalogues, ADRs
├── 06-security.md                — Threat model for agent tooling and prompts
└── 07-roadmap.md                 — Single canonical adoption roadmap
```

## The one-page summary

See `00-overview.md` for the canonical statement of the four problems, three governance questions, and four workflow patterns.

**The unifying insight:** the model isn't the bottleneck. The environment around it is. Files first (service maps, agent instructions, contracts), then tooling (cross-repo search, tool integrations), then governance (tiers, context rules, evals, compliance), then workflow patterns (contract-first, CDC). The full sequenced roadmap lives in `07-roadmap.md`.

## Who this is for

- Platform engineers rolling out AI tooling at organizations with 15-50 services
- Engineering managers thinking about AI governance
- Developers wanting to make their AI agents more effective

## Where to start

If you're just starting:
1. Read `00-overview.md`
2. Adopt the `SERVICES.md` pattern from `02a-services-map-template.md`
3. Set up the platform-docs repo per `01-platform-docs-setup.md`
4. Layer on governance from `04-governance.md`

If you already have AI tools deployed and want to improve them:
1. Read `04-governance.md` to assess your current boundaries
2. Read `06-security.md` to check your tool-integration threat model
3. Read `05-workflow-patterns.md` for what to invest in next
4. Skim `03-tooling-architecture.md` for the cross-repo search layer

---

> **Refresh cadence.** This document set discusses fast-moving practice. Review every six months; flag time-stamped claims and vendor specifics for re-verification. Last full review: 2026-05.
