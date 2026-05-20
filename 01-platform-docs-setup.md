# Platform docs setup — strategy

> Goal: Give AI coding agents (Claude Code, Cursor) reliable context about all services in the enterprise, without manual prompt overhead.

## The model

`platform-docs` is the editable source of truth. Every service repo gets a read-only synced copy of `SERVICES.md` via CI.

```
                  ┌─────────────────────────┐
                  │   platform-docs repo    │
                  │   ─────────────────     │
                  │   SERVICES.md  ← edit here
                  │   .github/workflows/    │
                  │     sync.yml            │
                  └──────────┬──────────────┘
                             │ on merge to main,
                             │ CI pushes file to each
                             ▼
        ┌──────────┬──────────┬──────────┬──────────┐
        │          │          │          │          │
   ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
   │order-  │ │payment-│ │user-   │ │catalog-│ │  ...   │
   │service │ │service │ │service │ │service │ │ 30+    │
   │        │ │        │ │        │ │        │ │ repos  │
   │SVCS.md │ │SVCS.md │ │SVCS.md │ │SVCS.md │ │SVCS.md │
   │(synced)│ │(synced)│ │(synced)│ │(synced)│ │(synced)│
   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

## The three moving parts

1. **One central repo** (`platform-docs`) — humans edit `SERVICES.md` via PR
2. **A CI job** in that repo — on every merge, pushes the file to every service repo
3. **A guard in each service repo** — prevents local edits from sneaking in

## Why push to every repo (vs. central only)

The whole point of `SERVICES.md` is that the agent reads it *without being told*. Most agentic IDEs and CLIs scan the current repo's root files by default. If the file only lives in `platform-docs`, the agent working in `order-service` won't see it unless someone manually adds the central repo as a workspace folder or references it in every prompt.

If it's at `order-service/SERVICES.md`, it's just there. No setup, no prompt discipline required. Same reason `README.md` lives in every repo even though most of its content could live elsewhere — discoverability beats DRY.

## Day-to-day workflow

**When someone adds a new event or service:**

1. Open PR in `platform-docs` editing `SERVICES.md`
2. CODEOWNERS auto-requests review from affected service teams
3. Merge → CI fires → file appears in all service repos within ~2 minutes
4. Next time anyone (human or AI) opens any service repo, they see the update

**When someone tries to edit it locally in `order-service`:**

- Their PR fails CI with a clear message: *"SERVICES.md is synced from platform-docs. Edit it there instead: github.com/acme/platform-docs"*
- They go open a PR in the right place

**When the AI agent works on a task:**

- It opens `order-service`, scans the root, finds `SERVICES.md`
- Reads it before making cross-service changes
- It doesn't need to know about `platform-docs` at all — the file is just there

## Build order (one day of work)

1. **Hour 1:** Create `platform-docs` repo. Put `SERVICES.md` in it (start with 3-4 services, expand from there).
2. **Hour 2:** Write the sync GitHub Action. Test it on 2 repos.
3. **Hour 3:** Add CODEOWNERS in `platform-docs`. Add the local-edit guard in 2 service repos.
4. **Hour 4:** Roll out the guard to all repos. Announce in engineering chat.
5. **Day 2 onwards:** Add an agent instruction file to each repo telling the agent to read `SERVICES.md` first.

## What you avoid

- **Drift across copies** — only one editable location, CI handles propagation
- **Stale info** — CODEOWNERS forces review by people who know the system
- **Manual prompt overhead** — the agent finds the file automatically
- **Ongoing toil** — sync is automated, edits are gated by PR review

## Pragmatic shortcut (if CI sync feels heavy at first)

- Keep `SERVICES.md` only in `platform-docs`
- In each service repo, put a one-line `SERVICES.md` pointing to it
- Rely on the agent instruction file to tell the agent to fetch the remote version

Less robust but zero infrastructure. Upgrade to full CI sync once the format stabilizes.

## When you might skip this entirely

- **Monorepo** — one copy at the root is enough.
- **Tool server exposing platform docs** — if the agent can reliably query the service map via a tool integration, you don't strictly need the file copied. A markdown copy is still cheap insurance and survives tool outages.

## Artifacts your platform team will build

These are intentionally not provided as drop-in copy-paste; the right shape depends on your CI system, branch-protection rules, and agent stack. Sketches:

**1. The sync job.** A workflow in `platform-docs` that, on merge to `main`, opens a PR (or pushes directly to a protected branch) in every service repo with the updated `SERVICES.md`. Use a bot account with a scoped token. Fail loudly if any target repo is unreachable so drift is visible.

**2. The local-edit guard.** A CI check in every service repo that fails the build if `SERVICES.md` is modified outside of a sync commit. Detect via committer identity (the sync-bot account) or a sentinel header in the file. The failure message points contributors to `platform-docs`.

**3. The agent instruction file.** A short file at the repo root that tells whatever agent runs there: read `SERVICES.md` first, check the change-impact cheatsheet before touching anything cross-service, and follow the tier rules in `04-governance.md`. The exact filename depends on the agent — some look for `AGENTS.md`, some for `CLAUDE.md`, some for `.cursorrules`. Keep one canonical file and symlink or sync to the others your org uses.

Build all three in a day. Iterate from there.
