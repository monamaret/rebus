# Research: Dashboard backend repo naming and the skipper/implementation boundary

**Status:** Complete — repo named (`saratoga`), boundary resolved; ready
to update `specs/research/023-platform-repo-inventory.md`'s Open Questions
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

[005-generic-app-scaffold.md](005-generic-app-scaffold.md) designed the
generic CLI+web+MCP scaffold and its concrete v1 instance — a deployment/
admin dashboard reading app-catalog data and live GKE health — but never
named the repo that backend lives in.
[023-platform-repo-inventory.md](023-platform-repo-inventory.md) surfaced
this as the one real gap in an otherwise-complete set of repo-naming
decisions: `pocket` (`018`), `rebus`/`bateau` (`006`), and `kingfish`
(`022`) each got this exact treatment; the dashboard backend never did.
This doc closes that gap.

- **Goal:** Name the dashboard backend's repo and confirm the
  `skipper`/implementation boundary, mirroring `018`'s treatment of
  `pocket` exactly.
- **In scope:** The repo name; the `skipper`/implementation boundary for
  this specific app; whether it needs more than one new repo (the CLI
  client, the web client, and the backend could in principle be three —
  resolved below).
- **Out of scope:** Re-opening `005`'s or `011`'s settled scaffold design
  (CLI+web+MCP shape, shared-operation mechanism, static-web-+-MCP-on v1
  defaults) — not reconsidered here.
- **Success criteria:** A named repo and a confirmed boundary, recorded in
  this doc's Decision Log.
- **Non-goals:** Designing the dashboard's actual internal code structure
  beyond what `005`/`011` already specified.

## Background & Context

- **Relevant prior work:**
  - [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) —
    the direct structural precedent this doc applies to the dashboard.
  - [005-generic-app-scaffold.md](005-generic-app-scaffold.md) — the
    scaffold design (CLI via Cobra, web via static Firebase Hosting +
    optional Cloud Run live mode, optional MCP co-located on the backend)
    this doc's named repo implements.
  - [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) —
    the shared `func(ctx, Input) (Output, error)` operation pattern,
    wrapped separately by an MCP adapter and a Cobra CLI adapter, both
    living in this same repo.
  - [022-tui-shell-repo-boundary.md](022-tui-shell-repo-boundary.md) —
    confirms the CLI-exec driver in `kingfish` shells out to a local
    binary; that binary is this doc's named repo's CLI client.
  - [.specify/memory/constitution.md Principle VIII](../../.specify/memory/constitution.md) —
    repo separation, directly decisive again here.
- **Domain assumption:** Like `pocket` (and unlike chat), the dashboard
  has no second-client/guest equivalent needing its own repo — its CLI
  and web clients are both first-party (owner-only), not a
  multi-party trust split the way `rebus`/`bateau` is.
- **Stakeholders:** mona (sole owner/operator, and sole dashboard user).

## Libraries & Stack

Not applicable — naming/boundary decision, not a technology comparison.

## Architecture Patterns

- **The dashboard backend is named `saratoga`, at
  `github.com/monamaret/saratoga` (not yet created).** Owner's own
  creative call, the same way `pocket`/`rebus`/`bateau`/`kingfish` were.
- **One repo, not three, despite three surfaces (CLI, web, backend).**
  Per `005`'s already-confirmed shape, the CLI (Cobra) and the MCP server
  live together with the backend in one process/repo (MCP is
  "co-located... not on Cloud Run," per `architecture.md`'s dashboard
  diagram) — there's no separate CLI-only or MCP-only repo to name. The
  web client (Firebase Hosting static shell) is the one piece that could
  in principle be a separate repo, but `005`'s scaffold treats CLI+web as
  "two thin clients" generated together as part of one scaffold instance,
  not two independently-versioned repos — so `saratoga` holds all three:
  backend, CLI, and the static web client's source.
- **The skipper/implementation boundary mirrors `018` exactly:**
  - **`skipper`'s own repo holds:** the persisted Kustomize base for the
    backend (`deployments/saratoga/`, per
    [010](010-gke-manifest-generation-and-upgrade.md#architecture-patterns)),
    the persisted Firebase Hosting/Cloud Run config for the web layer
    (`deployments/saratoga/web/`, per
    [012](012-firebase-hosting-cloud-run-deployment.md#decision-log)),
    and the app catalog entry (`cli-exec` for the CLI surface `kingfish`
    shells out to, plus `web` for the dashboard's static site, per
    [007](007-tui-shell-dispatch-and-catalog.md#decision-log)).
  - **`saratoga`'s own repo holds:** the backend service (stateless RPC/
    HTTPS, the MCP server, the shared operation functions and both their
    adapters), the Cobra CLI binary `kingfish`'s `cli-exec` driver
    invokes, and the static web client's source (React + Vite, per
    `005`).
- **No second repo is needed**, unlike chat's `rebus`+`bateau` split —
  there's no outside-domain guest equivalent for an admin/deployment
  dashboard; both clients are the owner's own.

**Tradeoffs:** None beyond what `018` already accepted for the analogous
case — a direct application of an already-justified pattern.
**Integration points:** Reuses `010`'s Kustomize mechanism (backend),
`012`'s Firebase Hosting/Cloud Run mechanism (web), and `007`'s catalog
schema (`cli-exec` + `web` entries) exactly as already designed — no new
deployment mechanism.
**Data model implications:** None new — `saratoga` reads, never owns, the
app-catalog (GCS/Firestore) and live GKE health (Kubernetes API); it has
no primary data store of its own, per `architecture.md`'s existing note
that this app is unlike every other v1 app in that respect.

## Existing Codebase Evaluation

No `saratoga` code exists yet, and the repo itself hasn't been created.
`pocket` remains the closest structural precedent (one repo, no second
client, public-but-thin boundary with `skipper`).

## Security & Compliance

No change from `005`'s already-confirmed posture (MCP defaults to
access-controlled not public; backend reachable only over a private VPC
path from Cloud Run; no WebAuthn needed since the v1 web client is static
with no live backend calls). This doc is purely a repo-boundary decision.

## Performance, Reliability & Operations

Not applicable — no new runtime characteristic; this doc only concerns
where code lives.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| `saratoga`'s implementation starts inside `skipper`'s own repo before `github.com/monamaret/saratoga` actually exists | Low–Medium | Low | Don't begin coding until the repo is created — `skipper app add --github github.com/monamaret/saratoga` has nothing to point at until then | mona |

## Open Questions

None — resolved below.

## Decision Log

- **2026-06-24** — **The dashboard backend is named `saratoga`, at
  `github.com/monamaret/saratoga` (not yet created).** Full
  implementation/code for the backend, CLI, and static web client lives
  in `saratoga`'s own repo — `skipper`'s own repo holds only the
  deployment config (Kustomize base + Firebase Hosting/Cloud Run config)
  and catalog entries that target it. — Decided by: mona.
- **2026-06-24** — **`saratoga` holds the backend, CLI, and web client
  together in one repo, not three** — `005`'s scaffold already treats
  CLI+web as two thin clients generated as part of one scaffold instance,
  not independently-versioned repos. — Decided by: mona.
- **2026-06-24** — **No second repo (no guest-client equivalent to
  `bateau`) is needed for the dashboard** — both its clients are
  owner-only, with no multi-party trust split. — Decided by: mona.

## Internal References

- [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) —
  the direct structural precedent applied here
- [005-generic-app-scaffold.md](005-generic-app-scaffold.md) and
  [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md) —
  the scaffold design this doc's named repo implements
- [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) and
  [012-firebase-hosting-cloud-run-deployment.md](012-firebase-hosting-cloud-run-deployment.md) —
  the deployment mechanisms `saratoga` deploys through
- [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md) —
  the catalog entry types (`cli-exec`, `web`) `saratoga` registers as
- [023-platform-repo-inventory.md](023-platform-repo-inventory.md) — the
  inventory this doc's naming decision updates
- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) —
  Principle VIII (repo separation), directly load-bearing above

## Web Citations

No external web research was needed — this doc applies an already-cited,
already-decided internal pattern (`018`) to a new case.

## Appendix

None yet.
