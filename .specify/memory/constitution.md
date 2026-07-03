<!--
Sync Impact Report
Version change: (initial ratification) → 1.0.0
This is rebus's first constitution. It is derived from the Skipper
Constitution v1.4.0 (the platform tool that scaffolds and operates
rebus), re-scoped from "the platform's scaffold/deploy tool" down to
"one application backend within that platform: the private 1:1 chat
backend." Platform-level decisions rebus inherits (rook/sight stateless
pattern, Firestore, SSH-key identity, transport+at-rest confidentiality,
asymmetric admin/participant roles, per-app namespace/IAM isolation) are
recorded in the vendored `specs/research/` corpus copied from `skipper`
and are cited per principle below rather than re-decided here.
Added principles:
  - I. Single-Owner, Minimal-Dependency Bias
  - II. Stateless RPC Backend, Client-Mediated (rook/sight)
  - III. The Public Client Package Is a Cross-Repo Contract
  - IV. Firestore-Backed, Incremental-Sync Data Model
  - V. Confidentiality: Transport + At-Rest, Not End-to-End
  - VI. SSH-Key Identity, IdP-Free
  - VII. Asymmetric Roles, Enforced Server-Side
  - VIII. Secure-by-Default GKE Deployment via skipper
  - IX. Test-First Development (NON-NEGOTIABLE)
Templates requiring updates:
  - AGENTS.md ✅ rewritten for rebus in the same change set
  - CLAUDE.md ⚠ keep in sync with AGENTS.md (see Governance)
  - .specify/templates/* — unchanged (generic spec-kit scaffolds; no
    rebus-specific rule embedded in them yet)
Follow-up TODOs:
  - Build / Test / Lint commands are "not yet established" — rebus's Go
    module does not exist yet. Record real commands in AGENTS.md and
    remove this TODO once `go build`/`go test` exist.
  - Feature item F-005 (rebus's own operation/method set) is not yet
    written; the exported client-package method list is finalized there.
-->

# Rebus Constitution

`rebus` (`github.com/monamaret/rebus`) is **one application backend within
the platform that `skipper` scaffolds and operates** — specifically, the
private 1:1 chat app's stateless backend. It is not the platform tool, not a
TUI shell, and not a client. This constitution governs `rebus`'s own code
and design; the platform-level decisions it inherits live in the vendored
`specs/research/` corpus (copied from `skipper`) and are cited, not
re-litigated, below.

## Core Principles

### I. Single-Owner, Minimal-Dependency Bias (No Speculative Abstraction)
Every dependency, abstraction, configuration knob, or generalization MUST
justify itself against `rebus`'s actual scope: a private, non-public chat
backend for **one owner** and exactly two known, owner-controlled client
consumers — `kingfish`'s embedded owner view and `bateau`, the guest
client. It serves multiple independent 1:1 conversations, not group chat,
rooms, federation, or a public multi-tenant service. Features pointing
toward productizing, open-sourcing, multi-party/group chat, real-time
presence, federation, additional cloud providers beyond GCP, or languages
other than Go MUST be rejected outright as out of scope unless a deliberate
future amendment to this constitution changes it.
**Rationale**: A private backend with one owner and two known consumers has
no need to carry the design weight of a general-purpose messaging platform.
Every abstraction added "just in case" is unjustified cost.

### II. Stateless RPC Backend, Client-Mediated (rook/sight Pattern)
`rebus` MUST remain a stateless RPC-over-HTTPS backend in the rook/sight
mold (`specs/research/006-private-chat-application.md`,
`002-hybrid-tui-layer.md`): it holds no open per-client session and renders
no server-side TUI (following `sight`'s `interactiveRejectionMiddleware`
precedent — interactive SSH sessions are rejected, not served). All client
data access is **backend-RPC-mediated**: clients (`kingfish`'s view,
`bateau`) NEVER read or write Firestore directly. Delivery is **async-only**
— there is no live/real-time/push mode; clients pull via incremental sync
(Principle IV). Each `func(ctx, Input) (Output, error)` operation is the
unit of the backend surface, per `011`'s shared-operation convention.
**Rationale**: Statelessness is what lets `rebus` be deployed, upgraded, and
scaled on GKE like any other stateless workload, and what keeps the two
clients honest — neither can smuggle in direct-database coupling that would
break the other or the wire contract.

### III. The Public Client Package Is a Cross-Repo Contract
`rebus` MUST publish exactly **one public (not `internal/`) Go package** —
`github.com/monamaret/rebus/client` — carrying the `Client`, its constructor
(`New(cfg Config)`), the request/response wire types, and the exported error
sentinels, following the shared shape fixed in
`specs/research/026-public-client-package-contract.md`. It is imported at
compile time by two separate modules (`kingfish` and `bateau`), so it is a
real cross-repo API contract regardless of single ownership: it MUST NOT
live under `internal/` (cross-module import would not compile), every method
takes `context.Context` first, and **any breaking change is a SemVer-major
release coordinated with both importers** — never a silent surface change.
There is deliberately no shared `*-clientkit` module (Principle I): the
*convention* is shared with `pocket`, the *code* is not.
**Rationale**: Once other repos import this package, its surface is expensive
to change silently. Treating it as a versioned contract up front is what
keeps `kingfish` and `bateau` buildable as `rebus` evolves.

### IV. Firestore-Backed, Incremental-Sync Data Model
Storage MUST be Cloud Firestore with a
`conversations/{conversationId}/messages/{messageId}` subcollection schema
(`006`). A conversation document holds the two participant identities (keyed
by registered public key), a per-participant `role` field (Principle VII),
and `createdAt`; a message document holds `sender`, `content`, `sentAt`,
`updatedAt` (distinct from `sentAt`, so an admin edit is distinguishable
from a new message in sync), and `hiddenFor` (the per-participant
soft-hide). Sync MUST be `updatedAt`-incremental —
`messages.where("updatedAt", ">", lastSyncTime)` per conversation — directly
extending `sight`'s `SyncNotes(ctx, since time.Time)` pattern. Relational
stores (Cloud SQL/Postgres) and self-hosted databases are rejected: there is
no join need, and self-hosting reintroduces operational burden the platform
avoids via managed services.
**Rationale**: The parent/child access pattern is a pure document store, and
Firestore is already the platform precedent; `updatedAt`-incremental sync is
the proven pattern the clients already expect.

### V. Confidentiality: Transport + At-Rest, Not End-to-End (Explicit Scope Boundary)
`rebus` provides **transport encryption (HTTPS) and Firestore at-rest
encryption only** — it is **NOT** end-to-end encrypted; the backend can read
message content by design (`006`). Retention is **indefinite** (no automatic
expiry). This is a deliberate, recorded scope decision, not an oversight: it
MUST NOT be quietly "improved" into E2EE, because doing so reshapes the
entire data model, the sync path, and the admin-edit capability. Adopting
E2EE (or finite retention) is therefore a **constitution-level amendment**,
not a routine feature.
**Rationale**: The confidentiality model is load-bearing for admin edit,
server-side role enforcement, and the plaintext `content` field the sync
model reads. Encoding it as a principle prevents a well-intentioned change
from silently breaking those guarantees.

### VI. SSH-Key Identity, IdP-Free (No Long-Lived Credential Leaves the Client)
Authentication MUST be an SSH-public-key-signed challenge: the client signs a
challenge locally and **never transmits the private key** (`002`, `014`).
Identity is independent of GCP/Google or any external IdP; participants —
including the outside-domain guest — are provisioned solely through the
owner-controlled public-key registry. Clients resolve their key SSH-agent
first, falling back to an encrypted private-key file with a passphrase prompt
(`014`). WebAuthn/passkeys are reserved for an *if/when* future web client
only and are out of scope for the TUI-only v1.
**Rationale**: Keeping identity SSH-key-based and IdP-free is what makes
inviting an arbitrary outside-domain participant possible without standing up
or trusting a third-party identity provider, and never transmitting the key
is the core security guarantee both clients rely on.

### VII. Asymmetric Roles, Enforced Server-Side
`rebus` MUST enforce an asymmetric permission model **at the backend**, not
in the clients (`006`, `013`): the **admin** (owner) may hard-delete
(physically remove the document), edit, and invite; a **participant** may
only hide their own view (setting `hiddenFor`, never removing the document).
Roles are stored as the per-participant `role` field and checked at the
operation/adapter layer. Admin operations surface **only** through
`kingfish`'s embedded owner view — they are NEVER exposed to `bateau` and
NEVER become a `skipper`-level command. There is no generic RBAC system;
this one asymmetry is the whole model.
**Rationale**: Server-side enforcement is the only enforcement that holds
when one of the two clients (`bateau`) is a lighter binary the owner does not
control at runtime; putting role checks in the client would make them
advisory, not real.

### VIII. Secure-by-Default GKE Deployment via skipper
`rebus` is deployed to GKE by `skipper app add --github`, and MUST be
deployable that way with no hand-run cluster or console mutation. It gets its
own GKE namespace (`skipper-rebus`) with a default-deny `NetworkPolicy`
(`019`) and its own dedicated GCP IAM service account (`skipper-rebus`) bound
via Workload Identity (`021`) — **no static service-account keys anywhere**
in this repo, its images, or its CI. This repo builds from source via
WIF-authenticated CI into Artifact Registry; the *persisted* Kustomize base
and Config Connector IAM/namespace config live in `skipper`'s
`deployments/rebus/`, **not** here (repo-separation, `023`). Container config
MUST default to production-sensible settings: resource requests/limits,
liveness/readiness probes, non-root user.
**Rationale**: `rebus` runs against a real GKE cluster and real Firestore
data; keyless Workload Identity and per-app namespace/IAM isolation bound the
blast radius of any compromise, and keeping deployment config in `skipper`
honors the platform's one-owner-of-deployment boundary.

### IX. Test-First Development (NON-NEGOTIABLE)
Tests MUST be written before implementation, confirmed to fail for the right
reason, and only then made to pass — red-green-refactor, strictly. This
applies to `rebus`'s backend operations, its Firestore access/sync logic, and
its published `client` package (whose contract, per Principle III, is exactly
what regressions are most expensive on). No feature, fix, or refactor lands
without a failing test demonstrating the gap it closes. Once real
build/test/lint commands exist they MUST be recorded in `AGENTS.md` (see
"Build / Test / Lint") and kept current.
**Rationale**: `rebus` holds real conversation data and publishes a contract
two other repos compile against — a regression has cross-repo and data
consequences, not just a failed local build.

## Security & Operational Posture

`rebus` operates against a real GCP project, GKE cluster, and Firestore
database, so the following are non-negotiable defaults rather than
configurable conveniences: no static GCP service-account keys anywhere in the
repo, generated configs, or container images — Workload Identity (cluster
workload) and Workload Identity Federation (CI/CD build+push) are the only
supported authentication paths. Application secrets live in Secret Manager
(or Kubernetes `Secret`s sourced from it), never in plain manifests or
committed env files. The SSH-key challenge auth (Principle VI) MUST never
require a client to transmit its private key. Because message `content` is
backend-readable (Principle V), access to the running service and its
Firestore data is itself sensitive and MUST stay private-by-default
(default-deny `NetworkPolicy`, least-privilege IAM) — the confidentiality
guarantee `rebus` makes to participants is "transport + at-rest," and the
operational posture MUST not undercut even that.

## Decision Recording & Research Workflow

`rebus` follows the same fixed idea→shipped pipeline as the rest of the
platform (the spec-kit tooling is shared) — steps are not skipped,
reordered, or collapsed, even for small features:

1. **Numbered research discussion document** — open or extend a numbered doc
   under `specs/research/` (copy `specs/research/research-template.md` for a
   new one; the `.claude/skills/tech-research/SKILL.md` skill automates
   this), every section filled in including a Web Citations section for any
   external source. Confirmed decisions — including supersessions — are
   appended to that doc's Decision Log with date, decision, rationale, and
   owner; superseded entries are kept (marked **Superseded**), not deleted.
   *The `specs/research/` docs presently in this repo are the vendored
   platform corpus copied from `skipper`; `rebus`'s own new research extends
   that corpus rather than replacing it.*
2. **Feature item creation** — create a feature item under `specs/features/`
   from `specs/features/TEMPLATE.md`, pointing back to the confirmed
   decisions rather than re-deciding them. (`rebus`'s own operation/method
   set is finalized in feature item **F-005**.)
3. **Roadmap inclusion** — add the feature item to
   `specs/research/roadmap.md` in its dependency-ordered place.
4. **Architecture sync** — update `specs/research/architecture.md` with any
   resulting diagram/call-flow changes *before* implementation starts.
5. **Spec Kit workflow** — only then run the Spec Kit workflow
   (`/speckit.specify` → `/speckit.clarify` → `/speckit.plan` →
   `/speckit.tasks` → `/speckit.implement`, etc.) to produce spec/plan/tasks
   artifacts and drive implementation, following the documented process at
   https://github.com/github/spec-kit/blob/main/spec-driven.md rather than
   improvising a different artifact shape.

`AGENTS.md` is the canonical, opencode-loaded project-instructions file and
the living source of truth for conventions and build/test/lint commands;
`CLAUDE.md` MUST be kept in sync with it rather than allowed to drift.

## Release Process

Each release MUST be packaged as a directory `specs/releases/X.XX-name/`
containing exactly: (1) a **final feature list** linking each shipped
capability back to its `specs/features/` item; (2) **release notes** written
for the owner/operator as the consumer (no external-user/marketing framing —
consistent with Principle I); (3) **unit and integration tests** exercising
the release's capabilities end to end, per Principle IX — a release is not
"final" until both layers exist and pass; and (4) a **usage guide** covering,
at minimum, how to build, configure, deploy (via `skipper`), upgrade, and
maintain `rebus` at that release, plus how importers consume the matching
`client` package version. Because the `client` package is a versioned
cross-repo contract (Principle III), each release MUST state its
`client`-package SemVer and call out any breaking change explicitly.

Every release also adds an entry to the root `CHANGELOG.md`, using the
scaffold in `specs/releases/CHANGELOG-TEMPLATE.md` (one line item per shipped
capability, each linked back to its `specs/features/` item; a **Deferred**
category for anything pushed out of scope). The
`.claude/skills/release-tracking/SKILL.md` skill automates both the
release-readiness check and this changelog step; before any implementation
work begins, that skill's scope-alignment gate (or the equivalent manual
check) MUST confirm the work traces to an existing `specs/features/` item,
roadmap entry, or referenced issue — no implementation proceeds against
unscoped work.

## Governance

This constitution supersedes any ad hoc convention or prior practice where
the two conflict. Amendments require: (1) a proposed change to this file with
a clear rationale, (2) a version bump per the policy below, (3) propagation
of any changed rule into `AGENTS.md`, `CLAUDE.md`, and the affected
`.specify/templates/*` files in the same change set, and (4) a Decision Log
entry in the relevant research document if the amendment reflects a new or
changed architectural/technical decision.

Versioning policy (semantic versioning for governance):
- **MAJOR** — a principle is removed or redefined in a backward-incompatible
  way (e.g. adopting end-to-end encryption, reversing the no-static-keys
  posture, exposing admin operations to `bateau`, or making the backend
  stateful/real-time).
- **MINOR** — a new principle or section is added, or existing guidance is
  materially expanded.
- **PATCH** — wording, clarification, typo, or other non-semantic refinement.

Every plan, spec, task breakdown, or research document that touches an area
governed by a principle above MUST state how it complies, or document and
justify any deviation as a deliberate, scoped exception (never as silent
drift). The project owner checks new work against these principles before
merging — complexity that cannot be justified against Principle I is the
default reason to push back. Use `AGENTS.md` for day-to-day runtime
development guidance; this constitution governs the *non-negotiable* rules
beneath it.

**Version**: 1.0.0 | **Ratified**: 2026-07-03 | **Last Amended**: 2026-07-03
