# `specs/research/reference/` — vendored platform reference (frozen)

This directory holds the **23 research documents `rebus` inherits from the
`skipper` platform** (`github.com/monamaret/skipper`) — every numbered
research doc *except* the three rebus-core docs (`006`, `013`, `014`), which
are `rebus`-owned and live one level up at `specs/research/`.

## Source of truth

**The authoritative, maintained version of every file here is in `skipper`:
`github.com/monamaret/skipper/specs/research/`.**

The copies in this directory are **frozen snapshots**, vendored into the
`rebus` repo so `rebus`'s own docs can cite them locally. They are **not
updated following this pass** — do not edit them here expecting the change to
propagate. When you need the current decision, the current rationale, or the
current cross-references, **check `skipper`**, not these files. Drift between
these snapshots and `skipper` is expected and acceptable; `skipper` wins.

The numbering (`R-NNN`) is `skipper`'s platform-wide scheme and is preserved
verbatim here so an `R-006` citation means the same thing in both repos.

## Link integrity caveat

Because these are frozen snapshots, their *internal* markdown links are not
maintained. Links from one reference doc to another (e.g. `009` → `002`) still
resolve since they all moved into this directory together; but links from a
reference doc up to the three rebus-core docs (`006`/`013`/`014`, now at
`../`) or out to `../features/` may be stale. The rebus-owned docs one level
up *do* keep their links into this directory current. For the authoritative
link graph, consult `skipper`.

## What's here

Grouped by how directly each bears on `rebus`. Citations use `R-NNN` for a
research doc and `F-NNN` for a feature item under `specs/features/`.

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
