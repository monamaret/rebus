# Roadmap: `rebus` implementation plan

**Status:** `rebus`'s architecture and data model are confirmed in research
(R-006, R-013, R-014, R-026) and the constitution, but the Go module does
not exist yet — `specs/features/` currently holds only `TEMPLATE.md`, and
**no `rebus` feature item has been created or implemented**. The next
concrete step is **F-005** (rebus's own operation/method set). The platform
that builds and operates `rebus` (`skipper`) has its own roadmap; it is
cited here as dependency context, not owned here.

This roadmap is the single, current view of "what's next" and "where we
are" for `rebus` (`github.com/monamaret/rebus`) — the private 1:1 chat
app's stateless backend. Each section below corresponds to a feature item
under `specs/features/`, in dependency order, and is only added once the
research behind it has actually settled a buildable shape — nothing here
is speculative scope.

## 0. Outstanding design questions

The single tracked location for open research/design items that affect
`rebus` itself. Platform-level questions (content sites, the TUI shell,
stash/dashboard design) are `skipper`'s concern and are **not** tracked
here — see the vendored corpus ([README.md](README.md)) for their status.

- [ ] **`rebus`'s concrete operation/method set (F-005)** — the shared
  client-package *shape* is fixed in
  [R-026](reference/026-public-client-package-contract.md) (`New(cfg)`, ctx-first
  `func(ctx, In) (Out, error)`, a `sight`-shaped `Sync(ctx, since)`,
  injected `Authenticator`, typed error sentinels), but the concrete method
  roster (`Send`, `Edit`, `Delete`, `Hide`, `Invite`, `ListConversations`,
  `History`, etc. — names and exact inputs/outputs) is not yet enumerated.
  This is finalized in feature item **F-005**, which is the gate before
  Spec Kit runs per-feature. — owner: mona
- [ ] **In-app unread-state UI design** — explicitly deferred during R-006's
  discovery interview (Q13): it is an in-app design question for the chat
  client's own screens, several levels below the backend surface. Belongs
  to `kingfish`/`bateau`'s design, not `rebus`'s; tracked here only because
  the backend may need to expose a cheap "unread count" query to support
  it. Revisit once the clients' screens are being built. — owner: mona
- [ ] **Local export/backup format** — R-006 confirmed the owner can export
  the full message history locally (Markdown/JSON, stash-app-style, off by
  default), but the exact format and whether it is a backend operation or
  a client-side rendering of synced data is undecided. — owner: mona

No platform-level design question blocks `rebus`'s own build: the
stateless pattern (R-002), Firestore schema (R-006), role model (R-013),
client-package contract (R-026), and deployment/IAM posture (R-019/R-021)
are all settled in the vendored corpus and cited by the constitution.

## 1. First release — the `rebus` backend

Deliverable: a working `rebus` backend deployed to GKE by `skipper`,
publishing the `rebus/client` package that `kingfish` and `bateau` can
import, exercising the full chat data model end to end. The design is
confirmed and scoped directly from [R-006](006-private-chat-application.md),
[R-013](013-cli-command-mapping-and-privilege-model.md),
[R-014](014-guest-chat-client-scope-and-auth-ux.md), and
[R-026](reference/026-public-client-package-contract.md).

**None of the feature items below exist on disk yet** — `specs/features/`
holds only `TEMPLATE.md`. The planned filenames resolve once the items are
created from that template.

In dependency order:

- [ ] **`rebus` Go module + stateless RPC skeleton** — establish the module,
  the `func(ctx, Input) (Output, error)` operation convention (R-011), the
  SSH-key-signed-challenge auth, the backend-RPC-over-HTTPS transport, and
  the Firestore client wired to the backend service account. Foundation
  for every operation below.
  **Planned feature spec (not yet created):** `specs/features/005-private-chat-backend.md`
- [ ] **Operation/method set (the public client package surface)** —
  finalize and publish `github.com/monamaret/rebus/client` (the cross-repo
  contract, R-026): the `Client`, `New(cfg)`, wire types, error sentinels,
  and the concrete method roster (this is **F-005** — the gate before the
  importers' specs can pin against a known surface). Send/read/hide for
  participants; hard-delete/edit/invite for the admin, gated server-side
  (R-013).
  **Planned feature spec (not yet created):** `specs/features/005-private-chat-backend.md`
  (F-005's method set is finalized within it)
- [ ] **Firestore data model + incremental sync** —
  `conversations/{conversationId}/messages/{messageId}` subcollections,
  the per-participant `role` field, the `updatedAt`/`hiddenFor`/`sentAt`
  message fields, and the `updatedAt`-incremental sync query (extending
  `sight`'s `SyncNotes(ctx, since)`), manual-check-only for v1 (R-006).
- [ ] **GKE deployment via `skipper`** — build-from-source CI (WIF) into
  Artifact Registry; the persisted Kustomize base + Config Connector
  IAM/namespace config land in **`skipper`'s** `deployments/rebus/` (not
  here — repo-separation, R-023); namespace `skipper-rebus`, SA
  `skipper-rebus` via Workload Identity, default-deny `NetworkPolicy`
  (R-019/R-021, Principle VIII).

The two clients — `kingfish`'s embedded owner view and `bateau` — are
**separate repos** with their own roadmaps; `rebus`'s first release is
done when the backend is deployed and the `client` package is importable
and tested against, not when either client is feature-complete.

## Dependency context (inherited from `skipper`)

`rebus` ships as **feature 005** of `skipper`'s platform first release
("Private 1:1 chat (TUI, v1 — web deferred)") — see the vendored corpus.
`skipper`'s first release defines six platform features in dependency
order; `rebus` is one of them, scaffolded and deployed by `skipper app add
--github`. `skipper`'s roadmap (the TUI shell, the app catalog, the
deployment pipeline, the other apps) lives in `skipper`'s repo; the
status of `skipper`'s own first-release items is **not** tracked here —
consult `github.com/monamaret/skipper` for that view. What `rebus` depends
on from `skipper` (the `app add`/`update`/`upgrade` deploy path, the
namespace/IAM convention) is settled in the vendored corpus and cited, not
re-litigated.
