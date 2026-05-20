# 05 — Workflow patterns

> How to structure your development process so AI tools work dramatically better. The framing that matters: your workflow is the agent's environment. A messy workflow gives the agent ambiguous signals; a structured workflow gives it rails to run on.

## Why workflow matters more with AI

The same model performs dramatically differently in well-structured vs. poorly-structured codebases — not because the model changed, but because the agent has fewer ways to be wrong.

These four patterns share a common shape: **they push decisions out of the implementation phase into earlier, more structured phases**. The agent is fast at implementation but bad at decision-making with incomplete context. By the time the agent writes code, all the load-bearing decisions should already be made and encoded somewhere the agent can read.

---

## Pattern 1 — Contract-first development

Before writing implementation code, define the contract (OpenAPI spec, `.proto` file, Avro schema). The contract is the source of truth; implementations follow from it.

### Why it matters with AI

Without contract-first: agent jumps into the controller, writes a handler, invents request/response shapes, writes tests that pass. Weeks later, downstream teams discover field-name mismatches and unexpected response codes.

With contract-first: agent edits the contract first as a separate PR, reviewed by service team + downstream consumers. Only then does implementation happen — and generated client SDKs propagate type-safe updates automatically.

The agent works dramatically better in the second flow because **the contract is a constraint it can reason about**.

### What it looks like

Three-step PR sequence:
1. **Contract PR.** Edit spec in `platform-docs/contracts/`. Reviewed by service team + downstream consumer reps via CODEOWNERS.
2. **Provider PR.** Implement contract in the service. CI checks implementation matches spec.
3. **Consumer PRs.** Each consumer updates to use the new contract. Generated SDK already has new types.

The agent can drive most of steps 2 and 3 autonomously because the constraint is locked in.

### The trap

Some teams adopt OpenAPI but treat the spec as documentation generated *from* the code. That's worse than nothing — the agent can't trust the spec as a source of truth.

Discipline: **spec changes before implementation changes, always**. If CI doesn't enforce this, your team will drift.

---

## Pattern 2 — Consumer-driven contract testing

Instead of testing your service in isolation, test that it satisfies the *expectations of every consumer*. Each consumer publishes a "contract" describing what it needs; your CI runs all consumer contracts against your implementation.

### Why dramatically more important with AI

The dominant failure mode of AI-assisted multi-service work is exactly this: agent updates direct callers, transitive callers break silently.

CDC testing solves this at CI level. Agent can refactor your service all day. If a consumer's contract no longer passes, CI fails. The agent doesn't need to be smart enough to find every consumer — your tests find them.

### How it works

CDC tooling is a mature ecosystem. Pact is the most widely used; spec-driven alternatives (such as Schemathesis for OpenAPI-fuzz testing, or Specmatic for contract-as-code) are lighter-weight on-ramps for orgs not ready for full CDC.

The shape of the workflow is consistent across tools:

1. **Consumer writes a test:** "I expect `GET /users/{id}` to return JSON with at least `id`, `email`, `name` as strings." Generates a contract artifact.
2. **A broker stores the contract.** Central service collects contracts from all consumers.
3. **Provider CI runs all consumer contracts** against the implementation. Any failure fails the build.

Key insight: **the provider doesn't need to know who its consumers are**. The broker tells it. A new consumer publishes its contract; the provider's CI picks it up automatically.

This is the technical solution to the social problem of "who consumes my API?" — exactly the problem the agent struggles with.

### Where it fits AI workflows

- **Before agent-led refactors:** pull all current consumer contracts, run locally. Safety net for cross-service work.
- **As a CI gate for AI-assisted PRs:** any PR touching a public API must pass all consumer contracts. Agent proposes whatever; CDC tests decide.

### Reality check

CDC testing requires significant cultural shift — every consumer team must publish and maintain contracts, and the provider team must treat broken contracts as build failures rather than consumer problems. This is why it appears late in `07-roadmap.md` even though it directly attacks the dominant AI failure mode. Adopt for the most-depended-on services first; expand as ROI proves out.

---

## Pattern 3 — Event catalogues (for async)

REST has OpenAPI. gRPC has `.proto`. Events have... whatever your team agreed on, often nothing structured.

An event catalogue is a centralized, schema-versioned, human-readable registry of every event. Events become first-class artifacts with owners, schemas, consumers, and lifecycle policies.

### Why this matters

Async breakage is the worst kind. A REST 500 fires alarms; a malformed event silently piles up in a dead-letter queue or — worse — gets parsed by a forgiving consumer that does the wrong thing. There's no synchronous handshake forcing the system to notice.

With AI: agent is asked to "add a field to the OrderCreated event." Without a catalogue:
- Agent edits producer code
- Maybe finds a couple of consumers via grep
- Ships
- A third consumer that uses dynamic deserialization keeps working but silently ignores the new field

With a catalogue + schema registry:
- Schema change is a PR against the registry, not code
- Compatibility rules enforced at registration time
- All consumers listed; agent knows the blast radius
- Consumer teams notified when subscribed events change

### What an event catalogue contains

For each event:
- **Name and version** (e.g., `order.created.v2`)
- **Owner team and contact**
- **Schema** (Avro, Protobuf, JSON Schema)
- **Producers** — who emits this
- **Consumers** — who subscribes
- **Lifecycle status** — active, deprecated, retiring
- **Compatibility rules** — backward, forward, full, none
- **Sample payload**
- **Description** — what business event, when it fires

Tools: AsyncAPI (the async-message equivalent of OpenAPI; the 3.x line is current as of writing), a schema registry (commercial or open-source), and a catalogue/UI layer if your team needs browsable docs.

### Change-control pattern

- **Adding optional fields:** safe, automated, agent-friendly
- **Adding required fields:** breaking change, new event version required
- **Removing fields:** breaking change, requires deprecation period
- **Changing semantics of existing field:** never allowed; always a new event

These rules encoded in schema registry's compatibility mode. Agent can propose changes; registry refuses incompatible ones at CI time. **Guardrails enforced by infrastructure are worth ten times guardrails enforced by review.**

---

## Pattern 4 — ADRs (Architectural Decision Records)

Short documents capturing *why* an architectural decision was made. Dated, owned, includes context, alternatives considered, the decision.

### Why ADRs matter for AI

The agent has no memory of why things are the way they are. It looks at your codebase, sees patterns, has no idea that the weird-looking `pollIntervalMs = 31337` is that way because of a specific bug from 18 months ago. So it confidently "fixes" it.

ADRs are the institutional memory the agent can read. Tell the agent "before changing polling logic, read ADR-0042" → you give it the historical context that would otherwise live only in some senior engineer's head.

### Structure

Five short sections:
- **Context** — what was happening, what problem we faced
- **Decision** — what we chose to do
- **Alternatives** — what we considered and rejected
- **Consequences** — what we accepted as tradeoffs
- **Status** — proposed / accepted / superseded by ADR-XXX

Numbered sequentially: `adr/0001-use-kafka-for-events.md`.

### Where ADRs live

In `platform-docs/adr/` or each service repo's `docs/adr/`. Referenced from `SERVICES.md`. Referenced from `CLAUDE.md`:

```
Before changing core authentication logic, read ADRs 0012, 0017, 0033.
```

### Realistic adoption

You won't have ADRs for everything. Write them for:
- Decisions that took more than a day of discussion
- Decisions that look weird without context
- Decisions that have been re-litigated once already
- Anything load-bearing for compliance or security

The right ADR count is whatever covers the load-bearing weirdness in your system. Many orgs land between a few dozen and a couple hundred at steady state. Add new ones at the cadence of new significant decisions, not on a calendar.

---

## How these patterns reinforce each other

Each removes a class of agent failure mode:

| Pattern | What it removes |
|---------|-----------------|
| Contract-first | API shape guessing |
| Consumer-driven tests | Silent transitive breakage |
| Event catalogue | Async drift, hidden consumers |
| ADRs | Confident "fixes" of weird code |

Together: every change is constrained by structure, not vibes. The agent gets safer; humans get faster.

## Pragmatic adoption order

This chapter's adoption order and the tooling order in `03-tooling-architecture.md` are reconciled in `07-roadmap.md`. Within just the workflow patterns:

1. **ADRs first.** Zero infrastructure. Start with new decisions; backfill the most-asked "why is it like this" questions.
2. **Contract-first for new APIs.** Don't retrofit existing services — adopt the discipline that every *new* API starts with a contract PR.
3. **Event catalogue with schema registry.** Highest ROI if you use Kafka/MQTT/queues heavily.
4. **CDC testing.** Most cultural shift. Adopt for the most-depended-on services first.

If you only do one: **ADRs**. Cheapest, fastest, gives you a place to document why you adopted everything else.

## The deeper insight

The model isn't the bottleneck. The environment around it is. Teams with mature contract-first practices get dramatically more value from AI tools than teams without. Not because their model is smarter — because their environment is more legible.

Build the environment first. The agent gets better automatically.
