# 07 — Adoption roadmap

> The single canonical sequence for adopting the practices in this doc set. `03-tooling-architecture.md` and `05-workflow-patterns.md` each contain partial orderings (tooling layers, workflow patterns). This chapter reconciles them into one timeline and one decision tree.

## The principle behind the order

Every step earlier in the roadmap makes later steps cheaper and safer. Skipping ahead is a common failure mode: orgs that build elaborate tool integrations before writing a services map discover the integrations don't help, because the agent still doesn't know what services exist. Build the cheap, high-leverage layers first.

## The roadmap

```
Phase 0 ─ Foundation               (week 1)
│  - platform-docs repo
│  - SERVICES.md (template → fill in top 5-10 services)
│  - CI sync to all service repos
│  - Agent instruction file in each repo
│  - First ADRs for load-bearing weird decisions
│  - AI usage policy (1 page)
▼
Phase 1 ─ Contracts                (weeks 2-4)
│  - Centralize OpenAPI / proto / Avro / MQTT specs in platform-docs/contracts/
│  - Adopt contract-first for new APIs
│  - Schema-registry compatibility rules enforced in CI
▼
Phase 2 ─ Governance & security    (weeks 3-5, parallel with Phase 1)
│  - Tier rules in agent instruction files (paths × autonomy levels)
│  - PR template with "context consulted" checklist
│  - Attribution label for AI-assisted PRs
│  - Tool-integration allowlist + signature policy (see 06-security.md)
│  - Secret scanning in prompts and agent output
│  - Cost controls: per-repo token budget, runaway-agent kill switch
▼
Phase 3 ─ Cross-repo search        (weeks 4-8)
│  - Index all repos with a code-search engine
│  - Expose to the agent as a tool (find_callers, find_consumers, search_code)
│  - Domain-scoped multi-root workspaces as fallback for IDEs without tool support
▼
Phase 4 ─ Evals & observability    (weeks 6-10, parallel with Phase 3)
│  - Golden trace replay for top agent workflows
│  - Tool-correctness + plan-adherence metrics
│  - Production trace sampling, retention per data-residency rules
│  - Dashboard: AI-assisted PR volume, merge rate, revert rate per service
▼
Phase 5 ─ Live-system integrations (month 2-3, pick highest-ROI first)
│  - Schema registry tool (usually highest ROI)
│  - API gateway / service mesh tool (find runtime callers)
│  - Event broker tool (topics, consumer groups, lag)
▼
Phase 6 ─ Consumer-driven contract testing (month 3+)
│  - Adopt for top 3-5 most-depended-on services
│  - Provider CI runs all consumer contracts
│  - Expand as ROI proves out
▼
Phase 7 ─ Continuous (always on)
   - Refresh ADRs as decisions land
   - Refresh SERVICES.md with every new service
   - Quarterly review of incidents, evals, cost, governance
   - Re-verify time-stamped claims and vendor specifics every six months
```

## Why CDC testing is last (and why that's controversial)

CDC testing directly attacks the dominant AI-assisted failure mode — locally-correct, globally-wrong refactors that break transitive callers. By the logic of "build the thing that catches the worst failure first," it should be Phase 1.

It isn't, for three reasons:

1. **Cultural prerequisite.** CDC only works if every consumer team publishes and maintains contracts and the provider team treats broken contracts as build failures. That's a multi-team agreement, not a tool install.
2. **Contract-first prerequisite.** If your APIs aren't already spec-first, CDC tests have nothing to bind against.
3. **Coverage prerequisite.** Until `SERVICES.md` exists, you don't know who your top 3-5 most-depended-on services are — and you'll waste effort instrumenting the wrong ones.

If your org already clears these prerequisites, move CDC earlier. The order here is the default for a typical 15-50 service org starting from a low baseline.

## Decision tree: where to start

```
Have a services map (any form)?
├── No  →  Phase 0. This is the highest-ROI step at any scale.
└── Yes →
    Do you have a security threat model for agent tooling?
    ├── No  →  Phase 2 security subset first (allowlist + secrets).
    │          A single rogue tool can leak more than any refactor breaks.
    └── Yes →
        Are public-API contracts in version control and enforced?
        ├── No  →  Phase 1.
        └── Yes →
            Can the agent find cross-repo callers without a human?
            ├── No  →  Phase 3.
            └── Yes →
                Can you measure agent quality with anything beyond merge rate?
                ├── No  →  Phase 4.
                └── Yes →
                    Pick the live-system integration with the highest
                    blast-radius problem (usually schema registry).
                    Or start CDC for your top-dependence services.
```

## What "done" looks like

You're at steady state when, for any change an agent proposes:

- The agent reads `SERVICES.md` without being told.
- Public-contract changes go through a contract PR before implementation.
- Cross-repo blast radius is found by tools, not by humans grepping.
- Tier rules determine review depth automatically.
- Tool calls are allowlisted, signed, and logged.
- Evals catch regressions before merge; observability catches them after.
- Reverts take minutes, not hours.

Nothing in this list requires a smarter model. All of it is environment work.

## Anti-patterns

- **Tooling before files.** Building tool integrations before `SERVICES.md` exists.
- **Governance before usage.** Writing a 30-page AI policy before anyone is using the tools. You'll govern hypotheticals.
- **CDC as the first investment.** Cultural prerequisites unmet → CDC sits unused.
- **One-shot rollout.** "We'll do it all in Q3." Pick the next phase, ship it, learn, then pick the next.
- **Skipping evals.** Without evals you can't tell whether a change to instructions, tiers, or tooling made things better or worse.

## Refresh cadence

Re-read this roadmap quarterly. Each phase has a "done" definition above; track which phases are at steady state, which are in progress, and which haven't started. The output of the quarterly review is the next 1-2 phases to invest in.
