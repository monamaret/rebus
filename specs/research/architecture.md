# Architecture: `skipper`

**Status:** Layer shape confirmed; first release's internal design confirmed for the TUI shell/backend (002–024, plus the 025 catalog-document and 026 client-package cross-repo contracts) and for the web application layer (001, 005, 008, 017) — no open layer-shape questions remain (see [roadmap.md §0](roadmap.md#0-outstanding-design-questions)). Per-feature specs under `specs/features/` are **not yet created** (see the numbering note below and [roadmap.md](roadmap.md)).

This document holds the reference diagrams and call-flows for `skipper`,
kept in sync with confirmed decisions in `specs/research/` and the roadmap
in `specs/research/roadmap.md`, per the "Architecture sync" step in the
feature development process (see `AGENTS.md` and
`.specify/memory/constitution.md`). Update this *before* implementation
starts for any change that affects the diagrams/call-flows below — it
should never drift out of sync with reality.

## System overview

The platform `skipper` scaffolds and operates is a single developer's
personal platform, composed of three layers with distinct deployment
targets (see constitution Principle VIII):

```
                    ┌────────────────────────┐
                    │   skipper (this repo)   │
                    │  scaffold + deploy/     │
                    │  upgrade/maintain CLI    │
                    └───────────┬─────────────┘
                                │ generates / operates
        ┌───────────────────────┼───────────────────────────┐
        ▼                       ▼                           ▼
┌───────────────┐     ┌─────────────────────────┐  ┌──────────────────┐
│   Backend      │     │   Web application layer  │  │  Hybrid TUI       │
│  custom apps + │     │   — two categories:       │  │     layer         │
│  GCP managed   │     │                           │  │  client-side,     │
│  services      │     │  (a) standalone content   │  │  lazy data sync   │
│  on GKE        │     │      sites → Cloudflare    │  │  to backend on    │
│                │     │      Workers (biblio)      │  │  certain pages    │
│                │     │  (b) backend-application   │  │                   │
│                │     │      UIs → Firebase        │  │                   │
│                │     │      Hosting + Cloud Run    │  │                   │
└───────┬────────┘     └────────────┬──────────────┘  └─────────┬─────────┘
        │                            │                            │
        │ own repo, scaffolded by skipper, maintained separately
        ▼                            ▼                            ▼
  (backend infra lives    (web app repo — biblio       (TUI client repo —
   in/managed by this      pattern for (a), or          kingfish, pattern
   repo directly)          Firebase Hosting/Cloud       from bbb-le; see
                           Run for (b))                 022)
```

- **Backend** — custom applications plus supporting GCP managed services
  (e.g. Firebase), deployed on GKE. This is the one layer whose
  infrastructure/deployment configuration lives directly in this repo.
- **Web application layer** — generated as its own repo per project; not
  vendored into `skipper`. Two categories, different deployment targets
  per [005-generic-app-scaffold.md](005-generic-app-scaffold.md):
  standalone content sites (Cloudflare Workers, reference
  `github.com/monamaret/biblio`) and backend-application UIs (Firebase
  Hosting + Cloud Run, GCP-native, governed by constitution Principle V).
- **Hybrid TUI layer** — generated as its own repo per project; not
  vendored into `skipper`. Reference pattern: `github.com/monamaret/bbb-le`.
  First concrete instance: **`kingfish`** (`github.com/monamaret/kingfish`,
  not yet created — see
  [022-tui-shell-repo-boundary.md](022-tui-shell-repo-boundary.md)).

Exact framework/library choices within each layer (web framework specifics,
TUI stack, lazy-sync mechanism) are open research items — see
[roadmap.md §0](roadmap.md#0-outstanding-design-questions) — not yet
confirmed decisions.

## First major release — TUI shell, catalog, and app call flows

Confirmed shape for [roadmap.md §1](roadmap.md#1-first-major-release),
from [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) and
[003-backend-application-deployment.md](003-backend-application-deployment.md).

**Numbering note:** the `(001)`–`(006)` labels in the diagrams/prose below
are shorthand for [roadmap.md §1](roadmap.md#1-first-major-release)'s
dependency-ordered first-release feature list, not citations to existing
files — the corresponding `specs/features/NNN-*.md` files haven't been
created yet.

**Repo boundary note (added 2026-06-24, see
[022-tui-shell-repo-boundary.md](022-tui-shell-repo-boundary.md)):** every
reference to "the TUI shell" or "the owner's TUI view" across **all three**
diagram blocks below — the shell call-flow block, the chat/`bateau` block,
and the deployment-dashboard block — lives in `kingfish`
(`github.com/monamaret/kingfish`, not yet created), a separate repo from
`skipper`, matching the repo boundary already drawn for the
backend/web-app/TUI-client layers in the System overview diagram above.
`skipper`'s own repo holds none of it — only the Kustomize bases and
catalog-write path for the backend apps shown connecting to it (Soft
Serve, `pocket`, `rebus`).

```
┌─────────────────────────────────────────────────────────────────────┐
│  TUI shell (001) — Bubble Tea launcher, configurable styling        │
│  [lives in kingfish, not skipper — see repo boundary note above]    │
│                                                                      │
│  ┌────────────────┐        ┌──────────────────────────────────┐     │
│  │ App catalog     │        │ Local-only views (no catalog      │     │
│  │ client          │        │ entry needed):                    │     │
│  │ - fetch GCS/    │        │ - Markdown reader (002, Glow)     │     │
│  │   Firestore     │        └──────────────────────────────────┘     │
│  │ - local cache,  │                                                 │
│  │   fallback on   │        ┌──────────────────────────────────┐     │
│  │   fetch failure │───────▶│ Dispatch per catalog entry:       │     │
│  └────────────────┘        │                                    │     │
│                              │  SSH-passthrough driver           │     │
│                              │  (suspend terminal, hand off,     │     │
│                              │   restore on exit)                │     │
│                              │      │                            │     │
│                              │      ▼                            │     │
│                              │  ┌─────────────────────┐          │     │
│                              │  │ Wishlist gateway      │          │     │
│                              │  │ (one GKE LoadBalancer,│          │     │
│                              │  │ shared by all Wish-   │          │     │
│                              │  │ pattern SSH apps)     │          │     │
│                              │  └──────────┬──────────┘          │     │
│                              │             ▼                     │     │
│                              │  ┌─────────────────────┐          │     │
│                              │  │ Soft Serve (003)     │          │     │
│                              │  │ off-the-shelf image, │          │     │
│                              │  │ GKE, SSH-session-     │          │     │
│                              │  │ served, behind        │          │     │
│                              │  │ Wishlist              │          │     │
│                              │  └─────────────────────┘          │     │
│                              │                                    │     │
│                              │  In-process embedded-view driver   │     │
│                              │  (resolves viewID against the      │     │
│                              │   shell's compile-time registry;   │     │
│                              │   apps may own nested screens,     │     │
│                              │   honoring the shared Esc/q/?/      │     │
│                              │   status-bar UX convention)        │     │
│                              │      │                            │     │
│                              │      ▼                            │     │
│                              │  ┌─────────────────────┐          │     │
│                              │  │ Stash app (004) —     │          │     │
│                              │  │ backend = `pocket`    │          │     │
│                              │  │ (github.com/monamaret/│          │     │
│                              │  │ pocket, not yet       │          │     │
│                              │  │ created), GitHub-repo │          │     │
│                              │  │ build, GKE, Cloud     │          │     │
│                              │  │ Storage (content) +   │          │     │
│                              │  │ Firestore (metadata,  │          │     │
│                              │  │ soft delete), local   │          │     │
│                              │  │ cache + incr. sync    │          │     │
│                              │  ├─────────────────────┤          │     │
│                              │  │ Chat app (005) —      │          │     │
│                              │  │ backend = `rebus`     │          │     │
│                              │  │ (github.com/monamaret/│          │     │
│                              │  │ rebus, not yet created)│          │     │
│                              │  │ GitHub-repo build,    │          │     │
│                              │  │ GKE, Firestore-backed │          │     │
│                              │  │ (conversations/{id}/  │          │     │
│                              │  │ messages/{id}), owner │          │     │
│                              │  │ TUI view here; web    │          │     │
│                              │  │ client deferred       │          │     │
│                              │  └─────────────────────┘          │     │
│                              │                                    │     │
│                              │  CLI-exec driver — shares the      │     │
│                              │  same suspend-and-exec core as     │     │
│                              │  SSH-passthrough above (just a     │     │
│                              │  local binary instead of `ssh`);   │     │
│                              │  see the dashboard (006) diagram   │     │
│                              │  block below for its target app    │     │
│                              └──────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Chat app (005) — bateau, the guest's separate client                │
│                                                                       │
│  ┌────────────────────────────┐      ┌─────────────────────────┐    │
│  │ `bateau`                    │─────▶│ `rebus` backend (GKE) —  │    │
│  │ (github.com/monamaret/      │      │ same backend as the      │    │
│  │ bateau, not yet created;    │      │ owner's chat view —      │    │
│  │ lighter binary, no shell/   │      │ send/read/hide only, no  │    │
│  │ catalog awareness) —        │      │ admin hard-delete/edit   │    │
│  │ imports `rebus`'s public    │      └─────────────────────────┘    │
│  │ client package (adapted     │                                       │
│  │ sight import-boundary,      │                                       │
│  │ public not `internal/`,     │                                       │
│  │ since `rebus`/`bateau`/     │                                       │
│  │ `kingfish` are separate     │                                       │
│  │ repos)                      │                                       │
│  └────────────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Deployment/admin dashboard (006) — generic scaffold's v1 instance   │
│                                                                       │
│  ┌──────────────┐         ┌─────────────────────────────────────┐    │
│  │ CLI client    │────────▶│ Backend (GKE) — stateless RPC/HTTPS, │    │
│  │ (Cobra, Go)   │         │ MCP server co-located here (not on   │    │
│  │ renders MCP's │         │ Cloud Run) — shared operations read: │    │
│  │ plain-text/   │         │  - app catalog (GCS, per 025)        │    │
│  │ ASCII-chart   │         │  - live GKE health (Kubernetes API)  │    │
│  │ tier directly │         └───────────────┬─────────────────────┘    │
│  └──────┬───────┘                          │                          │
│         │ shelled out to                  │ private VPC path,        │
│         │ from the TUI shell's             │ backend never            │
│         │ catalog entry (001)              │ required to be public    │
│         ▼                                  ▼                          │
│  ┌──────────────┐         ┌─────────────────────────────────────┐    │
│  │ TUI shell     │         │ Web client — Firebase Hosting        │    │
│  │ (001) renders │         │ (static shell), calls the backend's   │    │
│  │ same ASCII    │         │ MCP-exposed operations client-side;   │    │
│  │ visualization │         │ no live mode, no WebAuthn needed for  │    │
│  │ inline        │         │ this instance                         │    │
│  └──────────────┘         └─────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

Key call-flow notes:

- The catalog client (001) is the single discovery path for every driver
  — no *custom-built* discovery gateway exists (confirmed in
  [002 → Decision Log](002-hybrid-tui-layer.md#decision-log)). The
  SSH-passthrough driver does route through the real
  `github.com/charmbracelet/wishlist` tool, but only as a shared
  *network*-multiplexing layer for Wish-pattern SSH apps (one
  LoadBalancer for all of them) — the catalog still owns discovery
  exclusively; Wishlist is never queried for "what apps exist." See the
  amendment in [002 → Decision Log](002-hybrid-tui-layer.md#decision-log)
  and [007-tui-shell-dispatch-and-catalog.md](007-tui-shell-dispatch-and-catalog.md#decision-log).
- The Markdown reader (002) and the stash/chat apps (004/005) all render
  through the same local-Markdown/incremental-sync shape conceptually, but
  only the reader has zero remote component — stash and chat both talk to
  their own GKE-deployed backends.
- Chat (005) has two terminal clients for v1 (owner's TUI view, guest's
  separate standalone client, sharing one Go client-library package) but
  no *web* client — that's deferred, with the dual-thin-client (CLI + web)
  idea generalized into its own reusable scaffold (006) instead of being
  chat-specific.
- The deployment/admin dashboard (006) is the one app in this release with
  both a CLI and a web client against the same backend — and the first
  app whose web layer is GCP-native (Firebase Hosting + Cloud Run) rather
  than Cloudflare, per
  [005-generic-app-scaffold.md → Decision Log](005-generic-app-scaffold.md#decision-log).
  Unlike every other app in this release, it reads its own data from two
  sources (the app catalog populated by feature 003, and live GKE health),
  rather than owning its own primary data store.
- MCP responses include a plain-text/terminal-renderable visualization
  tier (ASCII charts/sparklines), so the same stats render identically
  whether reached via the CLI (standalone or shelled-out-to from the TUI
  shell) or the static web dashboard — one set of shared operations, three
  rendering surfaces, no duplicated visualization logic.

## Deployment pipeline

For apps 003/004/005/006: `skipper app add` (source-specific — `--image`
for off-the-shelf apps like the v1 Soft Serve test case, `--github <repo>`
for the stash, chat, and dashboard backends, per
[003-backend-application-deployment.md](003-backend-application-deployment.md))
→ generates a **Kustomize base** under `deployments/<app-name>/` in this
repo — a dedicated `Namespace` (`skipper-<app-name>`, with a default-deny
`NetworkPolicy` and explicit allow rules, per
[019-gke-namespace-strategy.md](019-gke-namespace-strategy.md)), a
`Deployment`/`Service` per Principle IV defaults, and the app's Workload
Identity binding (a Kubernetes `ServiceAccount` plus Config Connector's
`IAMServiceAccount`/`IAMPolicyMember` CRDs, per
[021-iam-service-account-scoping-strategy.md](021-iam-service-account-scoping-strategy.md))
— **persisted and committed**, applied atomically via `kubectl apply -k`
(per [010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md))
→ optional `skipper app set-interface` → app catalog write (a single
`catalog.json` object in Cloud Storage, per
[025-app-catalog-document-contract.md](025-app-catalog-document-contract.md))
→ discoverable by the TUI shell (`kingfish`)'s catalog client on next
refresh. Shared infrastructure (the Wishlist gateway) follows the same
namespace convention with its own dedicated namespace
(`skipper-wishlist`, per [019](019-gke-namespace-strategy.md#decision-log)).

`skipper app update --image <new-ref>` patches an app's persisted base
with a new image reference and re-applies. `skipper app upgrade`
re-renders an app's base against `skipper`'s current manifest-generation
template (e.g. if Principle IV's defaults change in a future `skipper`
release) and re-applies — a distinct operation from `update`, per
[010 → Decision Log](010-gke-manifest-generation-and-upgrade.md#decision-log).

## Upgrade / maintenance flow

The general mechanism (`app update` vs. `app upgrade`, against the
persisted Kustomize base) is defined in
[010-gke-manifest-generation-and-upgrade.md](010-gke-manifest-generation-and-upgrade.md) —
see Deployment pipeline above. Applying that mechanism to each specific
v1 app (e.g. when/how `stash`'s or `chat`'s image actually gets bumped in
practice) is still out of scope per each feature item's "Out of scope"
section; revisit per-app once the first release's apps are running and a
real upgrade need arises.
