# SERVICES.md — template

> A blank scaffold for your own services map. Copy this file into `platform-docs/SERVICES.md`, replace placeholders, and let CI sync it to every service repo.
>
> A worked example using a fictional org lives in `02b-services-map-example.md`.

---

```
# <Your Org> Platform — Services Map

> Canonical source: `github.com/<your-org>/platform-docs`. Do not edit copies in service repos.
> Last updated: YYYY-MM-DD · Owner: <team> (<contact channel>)

## How to read this file

Every service has the same sections:

- **Purpose** — one sentence, what it owns
- **Repo** — where the code lives
- **Owns** — domain entities/data this service is the source of truth for
- **Sync calls out** — services this one calls (HTTP/gRPC)
- **Async publishes** — events/messages this service emits
- **Consumes** — events/messages this service subscribes to
- **Called by** — known upstream consumers (sync)
- **Contracts** — paths to API specs / schemas

If you are an AI agent: **before making any change that touches an API, an event payload, or a shared entity, read this file end-to-end and identify every service in the blast radius.**

---

## System overview

<ASCII or Mermaid diagram of high-level service topology>

---

## Services

### <service-name>

- **Purpose:**
- **Repo:**
- **Owns:**
- **Sync calls out:**
- **Async publishes:**
- **Consumes:**
- **Called by:**
- **Contracts:**
- **Notes:** (security scope, on-call, regulatory tags)

<repeat per service>

---

## Event topology

| Event | Producer | Consumers | Schema version |
|-------|----------|-----------|----------------|
|       |          |           |                |

---

## Change-impact cheatsheet

Before changing any of the following, check who's affected:

| If you change… | Re-check these services |
|----------------|--------------------------|
|                |                          |

---

## Conventions

- **REST APIs:** versioning policy, deprecation window
- **gRPC:** version rules, field-number policy
- **Events:** schema format, compatibility mode, breaking-change policy
- **MQTT / queues:** topic/queue naming convention, coordination requirements

---

## Updating this file

1. Open a PR in `platform-docs` editing `SERVICES.md`. CODEOWNERS auto-requests review from affected service teams.
2. On merge, CI syncs the file to every service repo's root.
3. If you add a service: add a section here AND update the event topology table AND the change-impact cheatsheet AND the system overview diagram.
4. If you delete a service: leave a tombstone entry (`### <name> [REMOVED YYYY-MM]`) for at least six months so the agent doesn't try to call it.
```

## Conventions worth keeping

- **Same section shape for every service.** Predictability matters more than expressive flexibility — the agent learns the shape once.
- **Cross-references are paths, not URLs.** Repo-relative `contracts/...` paths survive renames; URLs rot.
- **Tombstone removed services.** A six-month tombstone period prevents the agent from "rediscovering" deleted services in old call sites.
- **Diagram is part of the file.** Don't link out to a wiki diagram — the agent reads the file, not the link.
