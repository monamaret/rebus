# Research: Platform repo inventory — what every named repo is and does

**Status:** Complete — survey only, no new design decisions; the one
naming gap originally flagged here (the generic-scaffold dashboard's
backend) is now resolved — see
[024-dashboard-backend-repo-naming-and-boundary.md](024-dashboard-backend-repo-naming-and-boundary.md)
**Date:** 2026-06-24
**Owner:** mona

## Problem Statement

Nine separate `github.com/monamaret/*` repos (plus `skipper` itself) are
named across `specs/research/` so far (the ninth, `saratoga`, was named after
this doc's first draft — see the resolved Open Question below), each
introduced in a different doc
for a different reason — naming decisions, reference-pattern citations,
and lifecycle-tool wrapping decisions are all scattered. There is no
single place that says, at a glance, what each repo *is* and *does*. This
doc is that single place: a high-level inventory, not new design.

- **Goal:** One inventory listing every named repo, what it does, what
  category it falls into (real not-yet-built platform component vs.
  already-existing tool `skipper` wraps vs. reference pattern only), and
  which doc(s) are authoritative for its details.
- **In scope:** Every `github.com/monamaret/*` repo named in
  `specs/research/` or the constitution as of this doc's date; `skipper`
  itself, for contrast.
- **Out of scope:** Re-deciding any repo's name, role, or design — this
  doc points to the docs that already decided those, it doesn't relitigate
  them. Repos that don't exist yet anywhere in the research corpus (e.g. a
  hypothetical future app) are not speculatively added.
- **Success criteria:** A reader unfamiliar with the research corpus can
  read this one doc and know what every named repo is for, without
  reading every other doc in the research corpus.
- **Non-goals:** Designing the name for the generic-scaffold dashboard's
  backend (flagged in Open Questions when this doc was drafted, since
  resolved separately as `saratoga` in `024`) — follows the same pattern as
  `018`/`006` (each a dedicated naming doc), not done inline here.

## Background & Context

- **Relevant prior work:** every doc cited in the table below decided the
  specific repo's name and/or role; this doc only aggregates.
- **Domain assumption:** "Named repo" means a repo with an actual
  `github.com/monamaret/<name>` identifier already chosen, whether or not
  it's been created yet. Generic categories without a chosen name (e.g.
  "a future content site," or the dashboard backend below) are not listed
  as repos in the table — they're flagged separately in Open Questions.
- **Stakeholders:** mona (sole owner/operator of every repo below).

## Libraries & Stack

Not applicable — this is a repo-role survey, not a technology evaluation.

## Architecture Patterns

### Three categories

1. **Real, not-yet-created components of `skipper`'s own deployed
   platform** — `skipper` will scaffold and/or deploy these; they exist
   only as named, designed-but-unbuilt repos today.
2. **Already-existing tools `skipper` wraps or invokes directly** — not
   platform components `skipper` deploys; external (from `skipper`'s
   perspective) lifecycle tools it orchestrates per Principle II.
3. **Reference patterns only** — real, existing repos `skipper`'s design
   draws on, never vendored, wrapped, deployed, or invoked by `skipper`
   or anything it scaffolds.

### Inventory

| Repo | Category | What it is/does | Status | Authoritative doc(s) |
| --- | --- | --- | --- | --- |
| **`skipper`** (this repo) | — (the platform's own scaffold/deploy tool) | The CLI-first scaffold/deploy/upgrade/maintain tool for the whole platform (Principle II). Holds only: persisted Kustomize bases and Config Connector IAM/namespace config for backend apps (`deployments/<app>/`), the app-catalog *write* path, and this research/spec pipeline. Holds no app source, no TUI shell code (see `022`). | Exists (this repo); no implementation code yet | constitution Principles I–VIII; `architecture.md` |
| **`kingfish`** | 1. Real platform component | The TUI shell: launcher, app-catalog *client* (rook-style local cache with fallback), three dispatch mechanisms (SSH-passthrough via a shared Wishlist gateway, in-process embedded-view, CLI-exec) sharing one suspend-and-exec driver core, the Lip Gloss theme system, the Glow-derived UX convention (`Esc`/`q` back, `?` help drawer, status bar), and the cross-app notification model. Also hosts the owner's compiled-in embedded-view adapters for stash and chat (importing `pocket`'s and `rebus`'s public client packages), since `kingfish`'s dispatch resolves `viewID`s against a compile-time registry with no dynamic plugin loading. | Not yet created | `002`, `007`, `015`, `016`, `022`, `025` (catalog it reads), `026` (client packages it imports) |
| **`pocket`** | 1. Real platform component | The Markdown-stash app's backend: Cloud Storage for file content, Firestore for metadata (soft delete with a recovery window, folder/tag organization via `path`/`tags` fields), `updatedAt`-incremental sync (sight's pattern). Deployed to GKE via `skipper app add --github`. Publishes a public (not `internal/`) Go client package that `kingfish`'s embedded stash view imports. | Not yet created | `009`, `018`, `026` (client-package contract) |
| **`rebus`** | 1. Real platform component | The private 1:1 chat app's backend: stateless RPC/HTTPS (rook/sight pattern), Firestore-backed (`conversations/{id}/messages/{id}` subcollections), `updatedAt`-incremental sync, supporting multiple separate 1:1 conversations. Transport+at-rest encryption only (not E2EE); indefinite retention; asymmetric admin (hard-delete/edit/invite) vs. participant (hide-only) permissions; async-only delivery, no live/real-time mode. Deployed to GKE via `skipper app add --github`. Publishes a public Go client package (wire types + RPC calls) imported by both `kingfish`'s owner-side embedded chat view and `bateau`. Admin operations surface only inside `kingfish`'s embedded chat view, never as a `skipper`-level command, never available to `bateau`. | Not yet created | `006`, `013`, `026` (client-package contract) |
| **`bateau`** | 1. Real platform component | The outside-domain guest's standalone chat client — a lighter binary with no shell/catalog awareness (not part of `kingfish`), importing `rebus`'s public client package directly. Send/read/hide only, no admin capability. Auth resolution: SSH agent first, falling back to an encrypted private-key file with a passphrase prompt. Interactive TUI plus non-interactive `history`/`send` commands (bbb-le's CLI shape); XDG config storage (rook-cli's convention); distributed via GitHub Releases (`goreleaser`) + `go install` (biblio's distribution precedent). | Not yet created | `006`, `013`, `014`, `026` (client-package contract) |
| **`saratoga`** | 1. Real platform component | The generic CLI+web+MCP scaffold's v1 instance: a deployment/admin dashboard. Stateless RPC/HTTPS backend on GKE with a co-located MCP server (not Cloud Run); shared `func(ctx, Input) (Output, error)` operations wrapped by both an MCP adapter and a Cobra CLI adapter; reads app-catalog data (GCS/Firestore) and live GKE health (Kubernetes API) over a private VPC path, never required to be public. Static web client on Firebase Hosting calling the same MCP-exposed operations client-side; no live mode, no WebAuthn needed for this instance. Backend, CLI, and web client all live in this one repo. The CLI client is shelled out to from `kingfish`'s catalog (CLI-exec driver). | Not yet created | `005`, `011`, `024`, `025` (catalog it reads) |
| **`biblio`** | 2. Existing tool, wrapped | `biblio-cli` — the real, already-built lifecycle tool for standalone content sites (React Router v7 + Vite, deployed to Cloudflare Workers). `skipper site new --type=<type> <repo-name>` wraps `biblio-cli new` directly; every subsequent lifecycle step (`index`/`update`/`build`/`deploy`) is `biblio-cli`'s own job, including its own generated CI workflow. Three new site types (`broadcast`/`changelog`/`api-docs`) are designed in `008`, pending `biblio-cli` itself adding `--type` support for them — a `biblio-cli`-side dependency, not `skipper`'s. Each individual site `skipper site new` creates is its *own* separate repo, named per-site by the owner at creation time — not a single fixed repo the way `pocket`/`rebus` are. | Exists, real, in active use as the reference/wrapped tool | `001`, `008`, `017` |
| **`bbb-le`** | 3. Reference pattern only | SSH-session-served TUI reference: Wish multiplexes a server-side Bubble Tea app (chat/profile TUI) directly over the SSH session itself; Cobra CLI; GKE via Kustomize base+overlays; Charm KV/BadgerDB session-resident state. Source of the category-4 (SSH-only-reachable apps, e.g. Soft Serve) pattern, and of the "backend-only admin role, separate from app access" convention (`bbb admin invite`/`admin revoke-key`) reused for `rebus`'s and the platform's general admin posture. Not forked, vendored, or deployed by `skipper`/`kingfish` — pattern only. | Exists, external reference only | constitution Principle VIII; `002`; `architecture.md` |
| **`rook-reference`** | 3. Reference pattern only | Stateless client + RPC/HTTPS server reference: `rook-cli` (local-first Bubble Tea TUI, signs an SSH auth challenge locally, never transmits the key) + `rook-server` (Cloud Run microservices) + `rook-server-cli` (separate admin binary). Source of the category-1/3 stateless-client pattern and of `kingfish`'s catalog-client cache-with-fallback design (`cache/spaces.json`'s "app ACL cache" shape, directly reused). Not forked or vendored. | Exists, external reference only | constitution Principle VIII; `002` |
| **`sight`** | 3. Reference pattern only | Stateless client + RPC-over-SSH-auth reference: `sightd` (server; as of v0.8.2, `interactiveRejectionMiddleware` actively rejects interactive SSH sessions — no server-side TUI rendering) + `sight-cli` (client). Source of the `updatedAt`-style incremental-sync pattern (`NoteService.SyncNotes(ctx, customerID, since)`) reused by both `pocket` and `rebus`, and of the bounded-wire-type-package import-boundary pattern adapted (as a *public*, not `internal/`, package) for `rebus`'s client package. Not forked or vendored. | Exists, external reference only | constitution Principle VIII; `002`, `006` |

**Tradeoffs:** None — this is a survey, not a new design choice.
**Integration points:** The table itself is the integration point — it's
the one place that should be checked before introducing a *tenth* named
repo, to confirm whether an existing one already covers the need.
**Data model implications:** None new.

## Existing Codebase Evaluation

None of the five "real platform component" repos (`kingfish`, `pocket`,
`rebus`, `bateau`, `saratoga`) have been created yet — all five are fully
named and designed (per their cited docs) but zero-code.
`biblio` is real, existing, and already in active use as a wrapped tool.
`bbb-le`/`rook-reference`/`sight` are real, existing repos consulted only
as design references — no code from them is reused, forked, or vendored
anywhere in this platform.

## Security & Compliance

No new concerns — this doc aggregates already-decided security postures
(SSH-key/WebAuthn identity model per `002`/`020`, per-app GCP IAM scoping
per `021`, per-app GKE namespace isolation per `019`) rather than
introducing any.

## Performance, Reliability & Operations

Not applicable — this is a survey doc, not an operational design.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation | Owner |
| --- | --- | --- | --- | --- |
| A sixth repo gets introduced informally (named in passing in some future doc) without ever getting the naming/boundary treatment `006`/`018`/`022`/`024` gave the others | Medium | Low–Medium | Check this inventory first; if a new repo is named, add it here and give it its own boundary doc if its role isn't already obvious from an existing pattern | mona |
| `biblio-cli` is conflated with the five "real platform component" repos in casual conversation, leading someone to expect `skipper`-style ongoing lifecycle commands for it | Low | Low | This table's explicit three-category split keeps the distinction visible | mona |

## Open Questions

- [x] ~~The generic-scaffold dashboard's backend repo has never been
  named.~~ — **Resolved 2026-06-24 by
  [024-dashboard-backend-repo-naming-and-boundary.md](024-dashboard-backend-repo-naming-and-boundary.md):
  named `saratoga`** (`github.com/monamaret/saratoga`, not yet created),
  one repo holding the backend, CLI, and static web client together, no
  second/guest-client repo needed.
- [ ] **No naming *convention* exists for content sites created via
  `skipper site new`**, unlike the fixed one-name-per-role pattern used
  for `pocket`/`rebus`/`bateau`/`kingfish`. This is expected, not a gap —
  per `017`, each site is named per-instance by the owner at creation
  time (`skipper site new --type=<type> <repo-name>`), since there can be
  many content sites, not one fixed repo per category. Noted here only
  so it isn't mistaken for an oversight when this table is read
  alongside the fixed-name rows above.

## Decision Log

- **2026-06-24** — **This inventory's three-category split (real platform
  component / wrapped existing tool / reference pattern only) is the
  organizing structure for tracking every named repo going forward** —
  any newly named repo should be slotted into one of these three
  categories, not treated as a fourth kind. — Rationale: the corpus
  already implicitly uses exactly these three relationships (`kingfish`/
  `pocket`/`rebus`/`bateau`/dashboard-backend are designed-but-unbuilt
  components; `biblio` is wrapped via `017`'s explicit "orchestrate, don't
  reimplement" decision; `bbb-le`/`rook-reference`/`sight` are repeatedly
  described as "reference... not vendored" — making the split explicit
  here just names a pattern that was already consistently followed.
  — Decided by: mona.

## Internal References

- [.specify/memory/constitution.md](../../.specify/memory/constitution.md) — Principle
  II (orchestrate vs. wrap, basis for `biblio`'s category) and Principle
  VIII (repo separation, basis for the "real platform component" category)
- [002-hybrid-tui-layer.md](002-hybrid-tui-layer.md) — names `bbb-le`,
  `rook-reference`, `sight` as the TUI layer's reference patterns
- [006-private-chat-application.md](006-private-chat-application.md) and
  [013-cli-command-mapping-and-privilege-model.md](013-cli-command-mapping-and-privilege-model.md) —
  name and design `rebus`/`bateau`
- [009-markdown-stash-application.md](009-markdown-stash-application.md)
  and [018-stash-backend-repo-naming-and-boundary.md](018-stash-backend-repo-naming-and-boundary.md) —
  design and name `pocket`
- [022-tui-shell-repo-boundary.md](022-tui-shell-repo-boundary.md) — names
  and bounds `kingfish`
- [005-generic-app-scaffold.md](005-generic-app-scaffold.md),
  [011-shared-operation-definitions-and-mcp-sdk.md](011-shared-operation-definitions-and-mcp-sdk.md),
  and [024-dashboard-backend-repo-naming-and-boundary.md](024-dashboard-backend-repo-naming-and-boundary.md) —
  design and name `saratoga`, the dashboard backend
- [001-web-layer-templates.md](001-web-layer-templates.md),
  [008-new-content-site-types.md](008-new-content-site-types.md), and
  [017-content-site-scaffold-and-deployment-boundary.md](017-content-site-scaffold-and-deployment-boundary.md) —
  design `biblio`'s wrapped role and the per-site repo-naming convention
- [architecture.md](architecture.md) — the diagrams this inventory's
  repos all appear in

## Web Citations

No external web research was needed — this doc aggregates already-cited
internal decisions, not new technical research.

## Appendix

None yet.
