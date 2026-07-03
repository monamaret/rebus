# Architecture: `rebus`

**Status:** `rebus`'s architecture is confirmed in research (R-006, R-013,
R-014, R-026) and the constitution, but the Go module does not exist yet —
no per-feature `specs/features/` items have been created or implemented.
This document describes the backend `rebus` *is*; the platform that
scaffolds and operates it (`skipper`) is cited as inherited context, not
as the subject.

This document holds the reference diagrams and call-flows for `rebus`,
kept in sync with confirmed decisions in the vendored `specs/research/`
corpus and the roadmap in [roadmap.md](roadmap.md), per the "Architecture
sync" step in the feature development process (see `AGENTS.md` and
[`.specify/memory/constitution.md`](../../.specify/memory/constitution.md)).
Update this *before* implementation starts for any change that affects the
diagrams/call-flows below — it should never drift out of sync with reality.

`rebus` (`github.com/monamaret/rebus`) is **one application backend within
the platform that `skipper` scaffolds and operates** — specifically, the
private 1:1 chat app's stateless RPC-over-HTTPS backend. It is **not** the
platform tool, not a TUI shell, and not a client. The numbered research
docs under this directory are the vendored platform corpus (see
[README.md](README.md)); this architecture doc is `rebus`'s own, re-scoped
from the platform-level view it inherited.

## System overview

`rebus` is a stateless RPC-over-HTTPS backend in the rook/sight mold,
backed by Cloud Firestore, publishing one public Go client package
consumed by two separate client repos. It holds no open per-client session,
renders no server-side TUI, and delivers messages asynchronously (clients
pull via incremental sync — no live/real-time/push path).

```
   ┌─────────────────────────────────┐    ┌─────────────────────────────────┐
   │  kingfish (github.com/          │    │  bateau (github.com/            │
   │  monamaret/kingfish)            │    │  monamaret/bateau)              │
   │                                  │    │                                  │
   │  TUI shell + embedded           │    │  Lighter standalone client      │
   │  OWNER chat view (FULL admin:   │    │  for the outside-domain GUEST   │
   │  hard-delete, edit, invite)     │    │  (send / read / hide only —     │
   │                                  │    │  NO admin operations)           │
   └───────────────┬─────────────────┘    └────────────────┬────────────────┘
                   │                                        │
                   │   both import ONE public Go package:  │
                   │      github.com/monamaret/rebus/client │
                   │   (cross-repo contract — see R-026)    │
                   │                                        │
                   ▼                                        ▼
   ┌─────────────────────────────────────────────────────────────────────────┐
   │  rebus backend (this repo) — stateless RPC/HTTPS on GKE                    │
   │                                                                            │
   │   - holds NO open per-client session (rook/sight pattern, R-002)          │
   │   - every op is  func(ctx, Input) (Output, error)  (R-011)                 │
   │   - async-only delivery — clients pull; no live/push mode (Principle II)  │
   │   - asymmetric roles enforced HERE at the backend, never in clients       │
   │     (Principle VII): admin = hard-delete/edit/invite; participant = hide  │
   │   - auth: SSH-public-key-signed challenge, IdP-free (Principle VI);       │
   │     the private key never leaves the client                               │
   └───────────────────────────────────┬───────────────────────────────────────┘
                                       │  backend service account (full access);
                                       │  clients NEVER touch Firestore directly
                                       ▼
   ┌─────────────────────────────────────────────────────────────────────────┐
   │  Cloud Firestore                                                          │
   │                                                                            │
   │   conversations/{conversationId}            ← top-level collection        │
   │     ├─ sender/pubkey, participant/pubkey, role[], createdAt               │
   │     └─ messages/{messageId}                 ← subcollection               │
   │           └─ sender, content, sentAt, updatedAt, hiddenFor                │
   │                                                                            │
   │   sync = messages.where("updatedAt", ">", lastSyncTime) per conversation  │
   │   (extends sight's SyncNotes(ctx, since) — R-006)                         │
   └─────────────────────────────────────────────────────────────────────────┘
```

- **Backend (`rebus`, this repo)** — Go, stateless RPC over HTTPS. The unit
  of its surface is the `func(ctx, Input) (Output, error)` operation
  ([R-011](011-shared-operation-definitions-and-mcp-sdk.md)). All Firestore
  access is backend-mediated; Firestore security rules are **not** the
  access-control mechanism (the backend service account has full access;
  role checks live in application logic, [R-013](013-cli-command-mapping-and-privilege-model.md)).
- **Public client package** — exactly one, `github.com/monamaret/rebus/client`,
  **not** under `internal/` (the Go toolchain forbids cross-repo
  `internal/` imports; see [R-026](026-public-client-package-contract.md)).
  It exports the `Client`, its `New(cfg Config)` constructor, the
  request/response wire types, and the exported error sentinels. Any
  breaking change to this surface is a SemVer-major release coordinated
  with both importers (Principle III).
- **Two client repos (separate from `rebus`)** — `kingfish`
  (`github.com/monamaret/kingfish`) embeds the **owner** chat view with
  full admin capabilities; `bateau` (`github.com/monamaret/bateau`) is the
  **guest's** lighter standalone client, send/read/hide only. Both are
  thin wrappers over the same `rebus/client` package; neither imports any
  other `rebus` internal package. The clients keep a local rook-style
  flat-file cache of synced messages per conversation ([R-006](006-private-chat-application.md)).

## Data model

Cloud Firestore, `conversations/{conversationId}/messages/{messageId}`
subcollections ([R-006](006-private-chat-application.md), Principle IV).
Firebase's own documentation names this exact parent/child shape as the
standard pattern for a chat application (see R-006's Web Citations).

- **Conversation document** (`conversations/{conversationId}`) — holds the
  two participant identities keyed by their registered public key, an
  explicit per-participant `role` field ([R-013](013-cli-command-mapping-and-privilege-model.md),
  Principle VII — not "creator = admin"), and `createdAt`.
- **Message document** (`messages/{messageId}`) — `sender`, `content`
  (plaintext, backend-readable per Principle V), `sentAt`, `updatedAt`
  (distinct from `sentAt`, so an admin edit is distinguishable from a new
  message in sync queries), and `hiddenFor` (the per-participant
  soft-hide a participant sets on their own view — never removes the
  document; the admin's hard-delete instead physically removes it).

**Incremental sync** is `messages.where("updatedAt", ">", lastSyncTime)`
per conversation, directly extending `sight`'s
`SyncNotes(ctx, since time.Time)` pattern. Combining that with an equality
filter on `hiddenFor` needs a composite index (Firestore supports it; the
single-inequality-field limit restricts only inequality filters — see
R-006's Web Citations). Sync is **manual-check-only** for v1 — no
background polling ([R-006](006-private-chat-application.md)).

Relational stores (Cloud SQL/Postgres) and self-hosted databases are
explicitly rejected (no join need; self-hosting reintroduces operational
burden the platform avoids via managed services — Principle IV).

## Authentication & identity

SSH-public-key-signed challenge, **IdP-free** (Principle VI, [R-002](002-hybrid-tui-layer.md),
[R-014](014-guest-chat-client-scope-and-auth-ux.md)): the client signs a
challenge locally and **never transmits the private key**. Identity is
independent of GCP/Google or any external identity provider; participants —
including the outside-domain guest — are provisioned solely through the
owner-controlled public-key registry. Clients resolve their key via the SSH
agent first, falling back to an encrypted private-key file with a
passphrase prompt ([R-014](014-guest-chat-client-scope-and-auth-ux.md)).
WebAuthn/passkeys are reserved for an *if/when* future web client only
([R-020](020-webauthn-implementation-mechanics.md)) and are out of scope
for the TUI-only v1.

## Role model

Asymmetric, enforced **server-side** (Principle VII, [R-013](013-cli-command-mapping-and-privilege-model.md)):

| Role | Capabilities |
| --- | --- |
| **admin** (owner) | hard-delete (physically remove the message doc), edit a sent message, invite participants |
| **participant** (guest) | hide their own view only (set `hiddenFor`, never remove the document); cannot edit |

The per-participant `role` field is checked at the backend operation/adapter
layer, never in the clients. Admin operations surface **only** through
`kingfish`'s embedded owner view — they are never exposed to `bateau` and
never become a `skipper`-level command. There is no generic RBAC system;
this one asymmetry is the whole model.

## Confidentiality

Transport encryption (HTTPS) + Firestore at-rest encryption **only** —
**not** end-to-end encrypted; the backend can read `content` by design
(Principle V, [R-006](006-private-chat-application.md)). Retention is
**indefinite** (no automatic expiry). This is a deliberate, recorded
decision: it is load-bearing for admin edit, server-side role enforcement,
and the plaintext `content` field the sync path reads. Adopting E2EE (or
finite retention, group chat, or a real-time mode) is a
constitution-level amendment, not a routine feature.

## Deployment

`rebus` is deployed to GKE by `skipper app add --github`, and MUST be
deployable that way with no hand-run cluster or console mutation
(Principle VIII). This repo builds from source via WIF-authenticated CI
into Artifact Registry; the **persisted** Kustomize base and Config
Connector IAM/namespace config live in **`skipper`'s** `deployments/rebus/`,
**not** here (repo-separation, [R-023](023-platform-repo-inventory.md)).

- **Namespace** — `rebus` gets its own GKE namespace `skipper-rebus` with
  a default-deny `NetworkPolicy` and explicit allow rules only for what it
  needs ([R-019](019-gke-namespace-strategy.md)).
- **Identity** — `rebus` gets its own dedicated GCP IAM service account
  `skipper-rebus` bound via Workload Identity (Kubernetes `ServiceAccount`
  + Config Connector `IAMServiceAccount`/`IAMPolicyMember` CRDs).
  **No static service-account keys anywhere** in this repo, its images, or
  its CI ([R-021](021-iam-service-account-scoping-strategy.md)).
- **Container config** — resource requests/limits, liveness/readiness
  probes, non-root user (Principle VIII defaults).

## Dependency context (inherited from `skipper`, not owned here)

`rebus` is operated by `skipper`, not by itself. The platform-level
architecture — the three-layer platform, the TUI shell, the app catalog,
the Kustomize/`kubectl apply -k` deployment pipeline, the
`skipper app add`/`update`/`upgrade` commands — is `skipper`'s concern and
lives in `skipper`'s repo. The decisions `rebus` inherits are recorded in
the vendored corpus (see [README.md](README.md)); they are cited here, not
redrawn. For the platform's own architecture/roadmap, consult `skipper`
(`github.com/monamaret/skipper`), not this file.

The two `rebus` clients (`kingfish`, `bateau`) are likewise separate repos
with their own architectures; this doc covers only `rebus`'s side of the
boundary they cross via the `rebus/client` package.
