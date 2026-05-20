# 04 — Governance

> How to define AI-safe change boundaries, runbooks, and guardrails. Without governance, AI rollouts are a coin flip — sometimes brilliant, sometimes catastrophic.

## The three governance questions

Every organization must answer these, in order:

1. **What can the agent change autonomously vs. with human review?**
2. **What context must the agent be given before making any change?**
3. **What happens when something goes wrong?**

These aren't independent. The answer to #1 determines what context is required for #2, and #2 determines the failure modes you'll see in #3. Treat them as a system, not a checklist.

Three more questions extend the original three at enterprise scale, addressed later in this chapter:

4. **How do you know the agent is getting better or worse over time?** — evaluation and observability.
5. **What rules from outside engineering apply?** — regulatory, license, and data-residency compliance.
6. **What is the cost envelope, and what happens when something runs away?** — cost controls.

Security threats — prompt injection, tool poisoning, secret exfiltration — are governance-adjacent but distinct, and live in their own chapter (`06-security.md`).

---

## Question 1 — What can the agent change autonomously?

The honest starting answer: *less than you think for "autonomously," more than you think for "with review."*

### The autonomy spectrum

Four levels, not three. Most discussions collapse them:

- **Level 0 — Suggestion only.** Agent proposes diffs in the editor; human accepts each one (Cursor autocomplete). Low risk, modest velocity gain.
- **Level 1 — Drafted PR, full human review.** Agent makes multi-file changes, opens a PR. Human reviews like any other PR. Low to medium risk, significant velocity gain.
- **Level 2 — Drafted PR, lightweight review.** Agent makes changes in well-understood patterns (test additions, documentation, internal refactors). Reviewer skims for sanity. Medium risk, large velocity gain.
- **Level 3 — Auto-merge after CI.** Agent makes changes, CI passes, PR auto-merges with no human review. Used for things like Dependabot bumps, formatting, README typo corrections. Risk depends entirely on what CI catches.

The mistake orgs make is jumping straight to Level 3 for things that should be Level 1, or staying at Level 0 for things that could be Level 2.

### The tiering system

Three tiers of code, mapped to autonomy levels:

**Tier 1 — Green zone (agent operates freely, Level 2-3 typical)**
- Tests (new tests, expanding coverage)
- Documentation, comments, README updates
- Internal tooling, scripts, dev utilities
- Local refactors within a single file
- Linting fixes, formatting

**Tier 2 — Yellow zone (agent assists, human drives, Level 1-2)**
- Application code within one service
- Internal APIs not consumed by other services
- Database queries (not schema)
- Configuration that doesn't affect security or networking

**Tier 3 — Red zone (Level 0-1, human designs)**
- Public API contracts (REST, gRPC, GraphQL)
- Event schemas (Kafka, MQTT, queues)
- Database schemas, migrations
- Authentication, authorization, security boundaries
- Anything in PCI/HIPAA/regulated scope
- Infrastructure as code
- CI/CD pipeline configuration
- Cross-service refactors

### Encoding tiers as file patterns

The pattern that works: **autonomy is a property of the file path, not the change**. This makes it enforceable in CI and clear to both agents and humans.

Example in `CLAUDE.md`:

```
Autonomy levels by path:

Level 3 (auto-merge after CI):
  - **/*.md, **/CHANGELOG.md
  - **/test/**/*.test.ts (new tests only)

Level 2 (lightweight review):
  - src/**/*.test.ts
  - scripts/**
  - internal/utils/**

Level 1 (full review):
  - src/**/*.ts (application code)
  - migrations/**

Level 0 (suggestion only):
  - src/api/contracts/**
  - src/auth/**, src/payment/**
  - infrastructure/**, terraform/**
  - *.proto
```

### Who decides the tiers

Hybrid model works best:
- Platform team publishes defaults
- Service teams can override with justification, visible in PR review
- Tier escalations (making Red zone Yellow) require platform team review

### The tests trap

A naive policy says "tests are Tier 3, agent can write whatever it wants." Then the agent writes 50 tests that assert what the code currently does (which may be buggy), giving you green CI and false confidence.

Better policy:
- *New* tests can be Tier 3 (additive)
- *Modifications* to existing tests are Tier 2 (could remove real coverage)
- Deletions of tests are always Tier 1

General principle: **autonomy is granted for adding things, not for changing or removing things**.

---

## Question 2 — What context must the agent be given?

This is where most orgs underinvest. Highest ROI area.

### Graduated context

Different changes need different context. Force-feeding everything wastes tokens and dilutes the agent's attention.

**Always (every session, automatic):**
- The repo's `CLAUDE.md` / `.cursorrules`
- Repo `README.md`

**For any file change:**
- The file itself, plus directly imported files
- Test files for the module being changed

**For Tier 2 changes:**
- Above + service's own contract files
- `SERVICES.md`

**For Tier 1 (red zone):**
- All of the above
- Relevant contracts from `platform-docs/contracts/`
- Event topology table from `SERVICES.md`
- Change-impact cheatsheet
- Recent ADRs related to the area

### Making context verifiable

The PR description must declare what context the agent consulted:

```
## Context consulted
- [ ] SERVICES.md sections: ...
- [ ] Contract files: ...
- [ ] Related ADRs: ...
- [ ] Other services that may be affected: ...
```

Reviewers can now see what the agent *claims* to have considered. A payment-touching PR with `SERVICES.md sections: none` is an immediate flag.

### Just-in-time vs just-in-case context

Counterintuitively, too much context is a problem:
1. The agent ignores most of it (attention dilution)
2. You burn tokens on context that isn't relevant

The fix: **just-in-time context via MCP tools**, not just-in-case context in every prompt.

- Default context: `CLAUDE.md` + repo README (small, always loaded)
- The agent has MCP tools like `get_service_info(name)` and `find_consumers(event)` to pull context when needed

### Data sovereignty considerations

For regulated environments, where context goes matters as much as what it contains. Vendor postures vary across products and pricing tiers — check current terms rather than relying on this document. Things to verify for every agent your org uses:

- Is prompt content used for vendor model training? (Look for an explicit opt-out or zero-retention enterprise tier.)
- Where geographically are prompts processed and stored?
- What is the retention period for prompts, tool calls, and responses?
- Is there an audit-log export your compliance team can consume?

Red-zone changes touching regulated data may need self-hosted inference even if your normal agent stack is fine elsewhere. Document this explicitly in your AI usage policy.

---

## Question 3 — What happens when things go wrong?

Three things must exist:

### Attribution

Every PR declares AI involvement — not as judgment, just as fact:

```
This PR was authored with significant AI assistance: yes / no
```

When something breaks, you want to know whether to update your prompt patterns or your code review process.

### The failure modes you'll actually see

After teams adopt AI tooling, failures cluster into five patterns:

| Pattern | Description | Best defense |
|---------|-------------|--------------|
| Confident hallucination | Agent invents an API method or library that doesn't exist | Strict CI (compile, lint, type-check) |
| Locally-correct, globally-wrong refactor | Updates direct callers, breaks transitive ones | `SERVICES.md`, cross-repo search, contract tests |
| Subtle semantic drift | Refactor changes behavior; tests updated to match | Behavior-focused tests, mutation testing, careful review |
| Security regression | Removes "redundant" auth check, weakens validation | SAST, security-focused PR templates, gated reviews |
| Performance regression | Idiomatic O(n²) where original was O(n) | Load tests, perf budgets in CI |

You can't catch all of them with one mechanism. Defense in depth matters.

### Adjusted postmortem template

Standard postmortems ask: what happened, what was the impact, how do we prevent recurrence. For AI-related incidents, add three questions:

1. What context did the agent have?
2. What context did the agent miss?
3. Is the fix to the agent's instructions, the human review process, or the technical guardrails?

The third question matters most. Was this preventable by:
- (a) Better `CLAUDE.md` instructions? → Update `CLAUDE.md`
- (b) More careful human review? → Update PR templates, reviewer guide
- (c) Better CI/testing? → Add tests, contract tests, linting

Most incidents have a mix. The output should be specific changes to one or more of these layers.

### Reversibility as a design principle

The most important guardrail isn't catching mistakes — it's making them cheap to fix:

- **Small PRs.** Easier to revert, easier to review carefully
- **Feature flags for behavior changes.** Flip the flag, don't revert the deploy
- **Two-step database migrations.** Add column → populate → switch reads → drop old
- **Atomic deploys with health checks.** Rollback in seconds, not hours

If you can roll back any AI-assisted change in under 5 minutes with no data loss, the cost of AI mistakes drops dramatically — and so does the cost of governance.

### Escalation path

Every team should know: *"who do I tell when an AI tool does something concerning?"*

Simple pattern:
- Slack channel: `#ai-tooling-feedback` (low ceremony)
- Platform team triages weekly
- Quarterly review: top 5 issues, top 5 wins, what changed

The biggest risk in AI governance is people not reporting weirdness because it didn't seem big enough.

---

## The three artifacts every org needs

1. **AI usage policy** — one page, lives in `platform-docs`. What tools are approved, what data can be sent where, what's prohibited, escalation contacts.

2. **Agent instruction file template** — lives in each repo via sync, under whatever filename your agent expects. How the agent should operate, tier rules, preflight checklist, project-specific conventions.

3. **Reviewer's guide for AI-assisted PRs** — one page, in `platform-docs`. What to look for, how to verify the agent consulted `SERVICES.md`, common failure modes, when to require a re-do.

Three documents. Most orgs over-engineer this — they want elaborate governance frameworks, AI review boards, mandatory training. None of that scales. Three living documents in git, reviewed via PR, do.

## Two patterns that fail

- **Total ban.** "No AI for production code." Lasts about three months. Engineers use it anyway, just covertly, now you have zero traceability. Worst of both worlds.
- **Total free-for-all.** Works until your first AI-caused incident. Knee-jerk policies appear, pendulum swings to total ban, nobody trusts the tools.

The middle path: explicit tiers, required context, attribution, runbooks. Boring. Effective.

## Make AI usage observable

You should be able to answer:
- How many PRs this month involved AI assistance?
- What's the merge rate of AI-assisted vs. baseline?
- What's the revert rate?
- Which services have the highest AI-assisted PR volume?

Not to police usage. Because once you have data, you can make decisions. "AI-assisted PRs in `user-service` revert 3x more often than the org average — why?" leads to actionable conversations. Without data, governance is just opinion.

Lightweight tag in PR titles or labels (`[ai-assisted]`), plus a dashboard. Two days of work, ongoing value.

---

## Question 4 — How do you know the agent is getting better or worse?

Most orgs ship AI tooling and never measure whether it's improving, regressing, or quietly causing harm. Postmortems are reactive; evals are proactive. Both are needed.

### Three evaluation layers

**Pre-deployment evals (regression tests for the agent).**
- Curate a small **golden set** of representative tasks for each major workflow (refactor, add endpoint, write test, fix bug from stack trace).
- For each task, record what "good" looks like — at minimum a passing test, ideally a reference diff to compare against.
- Run the golden set whenever you change instruction files, tier rules, tool integrations, or models. Catch regressions before they ship to developers.
- Use LLM-as-judge sparingly and skeptically — for subjective criteria (clarity, idiomatic style) it's useful; for correctness, prefer deterministic checks.

**In-session evals (the agent grading itself).**
- Plan adherence: did the agent do what it said it would?
- Tool correctness: did it call the right tool with the right arguments?
- Context usage: did it read the files / contracts / ADRs it should have?
- These can run as lightweight asserts inside the agent loop and surface in the PR description.

**Production evals (what's actually happening).**
- Sample real agent traces; review weekly.
- Track tool-call patterns over time — sudden shifts often indicate prompt-injection attempts or model behavior changes.
- Correlate with PR outcomes: revert rate, time-to-merge, CI pass rate.

### The observability ground truth

For every AI-assisted change you should be able to reconstruct, after the fact:
- Which agent and model version made the change.
- Which instructions and which context were in the prompt.
- Which tools were called, with what arguments, returning what.
- Which human reviewed it, and what they checked.

Without this, postmortems are guesswork. With it, "why did this incident happen" becomes a tractable question.

---

## Question 5 — What rules from outside engineering apply?

The non-engineering layers of governance — legal, compliance, procurement, privacy — touch AI tooling in ways most platform teams underestimate.

### Three buckets

**AI-specific regulation.** Multiple jurisdictions have AI-specific rules either active or rolling out on staggered timelines. The EU AI Act is the most visible example as of writing; others exist or are coming. Code generation is generally lower-risk than uses like credit scoring, but logging, transparency, and human-oversight requirements typically apply regardless. Track the deadlines that apply to your org. Have a named owner.

**Sector and data-residency regimes.** PCI, HIPAA, SOX, financial regulators, and analogues constrain where prompt data may flow, who may see it, and how long it must be retained. The audit-log requirements in Question 4 also satisfy most of these.

**License and provenance for AI-generated code.** AI output can echo training data, including copyleft-licensed code. Three habits to build now:
- Label AI-assisted PRs so future license audits don't require archaeology.
- Maintain an SBOM that includes AI tooling, not only application dependencies.
- For high-risk code paths, use code-signing / provenance attestation so you can prove what produced a given change.

### What this means for the platform team

Don't try to be the legal team. Do build three things they'll need:
- A documented data flow per agent: vendor, region, retention, training-use posture.
- An audit-log pipeline that compliance can read.
- A kill switch — a single config change that disables a tool, a vendor, or all AI tooling org-wide. Test it once a quarter.

---

## Question 6 — Cost controls

Agentic workflows are token-hungry by design. A loop that retries on failure, expands context on demand, and calls many tools can spend orders of magnitude more than a single completion. Without controls, monthly bills surprise people.

### Three controls

**1. Budgets at the right granularity.**
- Per-developer caps for IDE use.
- Per-repo caps for CI agents.
- Per-org cap as a final backstop.
- Cap the *budget*, not individual sessions — a hard session limit makes the agent abandon work mid-task; a budget limit signals when the rate is unsustainable.

**2. Visibility.**
- Dashboards showing spend per repo, per agent, per workflow.
- Anomaly detection on per-developer or per-repo spikes.
- Tie spend back to outcomes — cost per merged PR, cost per reverted PR — so the conversation is value, not just expense.

**3. Runaway-agent kill switch.**
- Detect loops: same tool called N times with similar arguments, or session exceeding a hard time budget.
- Auto-terminate, with a clear notification to the developer and an entry in the audit log.
- Kill switches at the gateway (`06-security.md`) are the enforcement point.

### Cost as a quality signal

Sudden cost increases are rarely a pricing change; they're usually a regression in agent behavior (a new tool that returns large responses, an instruction file that triggers excessive re-reading, a model that loops). Treat cost spikes as you would error-rate spikes — page someone.

---

## Heterogeneous fleets

Most orgs run more than one agent. An IDE assistant for developers, a CI bot for review, a scheduled job for routine maintenance, a chat-based agent for support queries. Each has different context, different authority, different failure modes.

Don't try to unify them. Do unify the things underneath:

- **One services map, one set of contracts, one set of ADRs.** Every agent reads the same source of truth.
- **One tool gateway, one allowlist, one audit log.** Where credentials and tool calls flow is the same regardless of which agent initiated.
- **One tier definition.** A red-zone path is red-zone for every agent.
- **Per-agent instruction files where behavior must differ.** A CI review bot follows different rules from an interactive IDE assistant.

Treat the fleet as a system. The failure mode to watch for is *one agent does what it's supposed to, the other doesn't, and nobody notices because the dashboards only cover one*.
