# `specs/research/` — the research corpus

This directory holds the research corpus for **`rebus`**
(`github.com/monamaret/rebus`), the private 1:1 chat app's stateless
backend. It is a mix of (a) a **vendored platform corpus copied from
`skipper`** — the scaffold/deploy tool that builds and operates `rebus` —
and (b) `rebus`'s own navigation and (eventually) `rebus`-specific research.
Of the 26 numbered docs, **23 are read-only vendored context** and **3 are
`rebus`-owned** — the rebus-core docs `006`/`013`/`014`, whose subject *is*
`rebus`/`bateau`. The split matters for the edit rule below.

## How to read this corpus

The numbered documents `001-*.md` through `026-*.md` are the **vendored
platform corpus**, copied from `skipper` verbatim and kept with `skipper`'s
platform-wide numbering (so `R-006` means "chat design" everywhere across
the platform's repos, not just here). They are the **read-only historical
record** of platform-level decisions `rebus` *inherits* — per the
constitution (`.specify/memory/constitution.md`):

> Platform-level decisions `rebus` inherits (rook/sight stateless pattern,
> Firestore, SSH-key identity, transport+at-rest confidentiality, asymmetric
> admin/participant roles, per-app namespace/IAM isolation) are recorded in
> the vendored `specs/research/` corpus copied from `skipper` and are cited
> per principle below rather than re-decided here.

**Do not edit the 23 platform-context docs in place.** They are cited as
inherited context by the constitution, AGENTS.md, and each other via
`R-NNN` references; editing them here would diverge this repo's record
from `skipper`'s and silently break those citations.

**The three rebus-core docs — `006`, `013`, `014` — are the exception: they
are `rebus`'s own design records**, written during `skipper`'s research
process but owned and maintained here (their subject *is* `rebus`/`bateau`,
not platform machinery). They are edited in place to stay accurate — e.g.
applying the stale-noun corrections `R-022` prescribed (see PR #1 for `006`).
Their substantive, dated Decision Log entries are still preserved as
history; only stale nouns/framing get corrected, never the decisions
themselves.

`rebus`'s own *new* research extends the corpus with **new** numbered docs
(copy `research-template.md`), never by rewriting the 23 inherited platform
docs.

Navigation docs — `architecture.md`, `roadmap.md`, and this `README.md` —
are **repo-owned**: they are re-scoped to `rebus` and edited freely, unlike
the numbered corpus.

## Relevance index (what to read first for `rebus`)

Every numbered doc, classified by how directly it concerns `rebus`. Citations
like `R-006` follow the convention in AGENTS.md: **`R-NNN`** = a research
doc, **`F-NNN`** = a feature item under `specs/features/`.

### Rebus-core — the subject IS `rebus` or `bateau`

| Doc | Topic |
| --- | --- |
| [`006-private-chat-application.md`](006-private-chat-application.md) | **The `rebus` design doc** — confidentiality model, Firestore schema, sync, roles, conversation scope |
| [`013-cli-command-mapping-and-privilege-model.md`](013-cli-command-mapping-and-privilege-model.md) | `rebus`'s asymmetric admin/participant role model (also contains `skipper`'s command table as context) |
| [`014-guest-chat-client-scope-and-auth-ux.md`](014-guest-chat-client-scope-and-auth-ux.md) | `bateau` — the guest's standalone client (commands, distribution, SSH-agent auth) |

### Inherited context — `rebus` directly depends on these platform decisions

| Doc | Why `rebus` depends on it |
| --- | --- |
| [`002-hybrid-tui-layer.md`](002-hybrid-tui-layer.md) | Establishes the stateless rook/sight RPC pattern and SSH-key, IdP-free identity `rebus` is built on |
| [`011-shared-operation-definitions-and-mcp-sdk.md`](011-shared-operation-definitions-and-mcp-sdk.md) | The `func(ctx, Input) (Output, error)` operation shape `rebus`'s RPC surface uses |
| [`019-gke-namespace-strategy.md`](019-gke-namespace-strategy.md) | `rebus`'s own namespace `skipper-rebus` (per-app isolation, default-deny `NetworkPolicy`) |
| [`021-iam-service-account-scoping-strategy.md`](021-iam-service-account-scoping-strategy.md) | `rebus`'s dedicated SA `skipper-rebus` via Workload Identity (no static keys) |
| [`023-platform-repo-inventory.md`](023-platform-repo-inventory.md) | Where `rebus` sits among the platform's repos |
| [`026-public-client-package-contract.md`](026-public-client-package-contract.md) | The cross-repo Go API contract `rebus/client` publishes for `kingfish`/`bateau` to import |

### Tangential — touch `rebus` but are not core

| Doc | Relation to `rebus` |
| --- | --- |
| [`003-backend-application-deployment.md`](003-backend-application-deployment.md) | `skipper app add --github` — how `rebus` gets built and deployed |
| [`010-gke-manifest-generation-and-upgrade.md`](010-gke-manifest-generation-and-upgrade.md) | The Kustomize/persistence/update-vs-upgrade mechanism managing `rebus`'s manifests (which live in `skipper`'s `deployments/rebus/`, not here) |
| [`015-shell-notification-and-sync-freshness-model.md`](015-shell-notification-and-sync-freshness-model.md) | How `rebus` notifications surface on `kingfish`'s status bar |
| [`016-notification-and-theming-customization.md`](016-notification-and-theming-customization.md) | Per-app notification theming `rebus` uses |
| [`020-webauthn-implementation-mechanics.md`](020-webauthn-implementation-mechanics.md) | Reserved for a *future* `rebus` web client only — out of scope for the TUI-only v1 |

### Platform-only — other apps / `skipper`'s own surface (not `rebus`)

Kept as inherited context for cross-references; not load-bearing for
`rebus`'s own build.

| Doc | What it is |
| --- | --- |
| [`001-web-layer-templates.md`](001-web-layer-templates.md) | Content-site web templates (biblio category) |
| [`004-mobile-device-layer.md`](004-mobile-device-layer.md) | Mobile layer — on hold, stale premise |
| [`005-generic-app-scaffold.md`](005-generic-app-scaffold.md) | Generic CLI+web scaffold (dashboard) |
| [`007-tui-shell-dispatch-and-catalog.md`](007-tui-shell-dispatch-and-catalog.md) | `kingfish` shell dispatch + catalog schema |
| [`008-new-content-site-types.md`](008-new-content-site-types.md) | Content-site types (changelog/api-docs/broadcast) |
| [`009-markdown-stash-application.md`](009-markdown-stash-application.md) | `pocket` stash backend design |
| [`012-firebase-hosting-cloud-run-deployment.md`](012-firebase-hosting-cloud-run-deployment.md) | Web-layer (Firebase Hosting + Cloud Run) deployment |
| [`017-content-site-scaffold-and-deployment-boundary.md`](017-content-site-scaffold-and-deployment-boundary.md) | Content-site scaffolding (`biblio-cli` wrapping) |
| [`018-stash-backend-repo-naming-and-boundary.md`](018-stash-backend-repo-naming-and-boundary.md) | `pocket` repo naming/boundary |
| [`022-tui-shell-repo-boundary.md`](022-tui-shell-repo-boundary.md) | `kingfish` repo boundary |
| [`024-dashboard-backend-repo-naming-and-boundary.md`](024-dashboard-backend-repo-naming-and-boundary.md) | `saratoga` dashboard repo naming/boundary |
| [`025-app-catalog-document-contract.md`](025-app-catalog-document-contract.md) | Platform app-catalog document contract |

## Navigation docs (repo-owned, edit freely)

- [`architecture.md`](architecture.md) — `rebus`'s own architecture: the chat
  backend, Firestore schema, public client package, auth, and GKE deployment.
  Cites the `skipper` platform architecture as inherited context, not as the
  subject.
- [`roadmap.md`](roadmap.md) — `rebus`'s own implementation roadmap;
  `F-005` (rebus's operation/method set) is the next concrete step, with
  `skipper`'s first release as dependency context.
- [`research-template.md`](research-template.md) — the template for new
  research docs (copy it; new `rebus` research extends the corpus with new
  numbering).
